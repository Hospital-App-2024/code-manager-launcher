# Migración productiva de códigos de emergencia

## Condiciones obligatorias

- Ejecutar primero sobre una restauración reciente de producción.
- Confirmar que `20260711143243_new_init` no figure como aplicada.
- Detener todas las escrituras antes de ejecutar la migración.
- Disponer de un snapshot o `pg_dump` cuya restauración haya sido comprobada.
- No ejecutar el job si `prisma migrate status` informa drift, migraciones fallidas o un historial distinto del auditado.

## Preflight de solo lectura

Registrar los resultados de estas consultas antes del mantenimiento:

```sql
SELECT migration_name, finished_at, rolled_back_at
FROM "_prisma_migrations"
ORDER BY started_at;

SELECT 'GREEN' AS type, COUNT(*) FROM "CodeGreen"
UNION ALL SELECT 'BLUE', COUNT(*) FROM "CodeBlue"
UNION ALL SELECT 'AIR', COUNT(*) FROM "CodeAir"
UNION ALL SELECT 'RED', COUNT(*) FROM "CodeRed"
UNION ALL SELECT 'LEAK', COUNT(*) FROM "CodeLeak";
```

Registrar también el estado de cierre de los Códigos Verdes. La migración
conserva `observations` y el cierre (`closedBy`, `closedAt`) de cada código:

```sql
SELECT
  COUNT(*) FILTER (WHERE "isClosed") AS cerrados,
  COUNT(*) FILTER (
    WHERE "isClosed"
      AND (NULLIF(BTRIM("closedBy"), '') IS NULL OR "closedAt" IS NULL OR "closedAt" < "createdAt")
  ) AS cerrados_incompletos,
  COUNT(*) FILTER (
    WHERE NOT "isClosed"
      AND NULLIF(BTRIM("closedBy"), '') IS NOT NULL
      AND "closedAt" IS NOT NULL AND "closedAt" >= "createdAt"
  ) AS cerrados_con_bandera_desactualizada
FROM "CodeGreen";
```

- `cerrados_incompletos` debe ser `0`: si no, la migración aborta sin modificar
  nada y esos códigos hay que corregirlos antes (nombre y fecha de cierre).
- `cerrados_con_bandera_desactualizada` son códigos con `isClosed = false` pero
  con nombre y fecha de cierre completos; se migran como cerrados.
- Un código abierto con datos de cierre a medias (solo nombre o solo fecha) se
  migra abierto y esos datos se descartan; siguen en la tabla legacy.

Registrar también los equipos escritos en los Códigos Azules, que la
migración `20261003120000_blue_teams_cogrid_time_closure_operator` convierte al
enum `BlueTeam` (`EMERGENCY`, `ICU`, `PEDIATRIC_ICU`):

```sql
SELECT "team", COUNT(*) FROM "CodeBlue" GROUP BY "team" ORDER BY 2 DESC;
```

Se reconocen urgencia(s), UCI y UCI pediátrica, con o sin el prefijo
"Equipo", con o sin tildes y en cualquier capitalización. Si aparece otro
valor, esa migración aborta nombrándolo y no modifica nada: hay que ampliar su
tabla de equivalencias antes de la ventana de mantenimiento.

La propia migración aborta transaccionalmente si falta una tabla, existe
`EmergencyCode`, hay IDs repetidos entre orígenes, existen operadores
huérfanos o no coinciden los conteos copiados.

## Ensayo y despliegue

1. Construir explícitamente los targets `migrate` y `production`.
2. Ejecutar `pnpm exec prisma migrate status` contra la restauración.
3. Ejecutar el job de migración y registrar duración y salida completa.
4. Verificar conteos globales y por `CodeType`.
5. Probar lectura, creación, edición, cierre de GREEN, filtros, estadísticas y PDF.
6. Repetir el procedimiento en producción bajo modo mantenimiento.

Con un PostgreSQL desechable cuyo nombre termine en `_migration_test`, el
ensayo automatizado se ejecuta mediante `pnpm test:migration` usando
`MIGRATION_TEST_DATABASE_URL`.

El job se ejecuta de forma explícita:

```bash
docker compose -f docker-compose.prod.yml --profile migration run --rm code-manager-migrate
```

La aplicación se inicia únicamente después de que el job finalice con código 0:

```bash
docker compose -f docker-compose.prod.yml up -d code-manager-backend code-manager-frontend
```

## Verificación posterior

```sql
SELECT type, COUNT(*) FROM "EmergencyCode" GROUP BY type ORDER BY type;

SELECT COUNT(*) AS orphan_operators
FROM "EmergencyCode" emergency
LEFT JOIN "Operator" operator ON operator.id = emergency."operatorId"
WHERE operator.id IS NULL;

SELECT COUNT(*) AS invalid_non_green_closures
FROM "EmergencyCode"
WHERE type <> 'GREEN'
  AND ("closedBy" IS NOT NULL OR "closedAt" IS NOT NULL OR "closedByOperatorId" IS NOT NULL);
```

El resultado esperado para las dos últimas consultas es `0`.

Comprobar que los Códigos Verdes cerrados coinciden con el preflight
(`cerrados` + `cerrados_con_bandera_desactualizada`):

```sql
SELECT COUNT(*) AS verdes_cerrados
FROM "EmergencyCode"
WHERE type = 'GREEN' AND "closedAt" IS NOT NULL;
```

Comprobar además que ningún Código Azul quedó sin equipos:

```sql
SELECT COUNT(*) AS blue_without_teams
FROM "EmergencyCode"
WHERE type = 'BLUE' AND cardinality(teams) = 0;
```

El resultado esperado es `0`.

## Rollback

Mientras siga activo el modo mantenimiento y no existan escrituras nuevas, se
puede detener la aplicación nueva, renombrar las tablas legacy a sus nombres
originales y arrancar la imagen anterior sin ejecutar migraciones. El historial
de Prisma debe reconciliarse después antes de cualquier nuevo deploy.

Una vez reabiertas las escrituras, no se permite ese rollback porque perdería
eventos nuevos. Desde ese punto se aplica un forward-fix o se restaura el backup
con una reconciliación explícita de las escrituras posteriores.

Las tablas `*_legacy_20260824` permanecen siete días en solo lectura. Su
eliminación requiere una migración independiente, un backup vigente y una nueva
comparación de conteos.
