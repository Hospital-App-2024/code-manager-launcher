# Code Manager - Arquitectura y Documentación Oficial

## Visión General
Esta aplicación gestiona códigos de emergencia en un entorno hospitalario o corporativo (Código Azul, Verde, Rojo, Aéreo y Fuga).

## Arquitectura de Datos (Backend - NestJS & Prisma)
### Patrón Aplicado: Single Table Inheritance (STI)
En lugar de tener una tabla en la base de datos para cada tipo de código de emergencia (lo cual genera duplicación y consultas complejas), **toda la información reside en una única tabla unificada llamada `EmergencyCode`**. 

- **Estrategia**: Se utiliza el enum `CodeType` (`GREEN`, `BLUE`, `AIR`, `RED`, `LEAK`) para discriminar qué tipo de emergencia representa cada fila.
- **Ventajas**:
  1. Escalabilidad: Agregar un nuevo código de emergencia solo requiere añadir un tipo al Enum y agregar las columnas opcionales a la tabla unificada, sin crear nuevas tablas.
  2. Integridad por tipo: NestJS valida el estado completo mediante una función de dominio y PostgreSQL replica las invariantes críticas con restricciones `CHECK`.
  3. Análisis unificado: Para obtener totales por mes de **todas** las emergencias conjuntas basta con una sola consulta.

### Base de Datos & Rendimiento
- **IDs**: Los IDs históricos `cuid` se conservan como `TEXT`; Prisma genera UUID para los registros nuevos sin romper relaciones existentes.
- **Fechas y Auditoría**: 
  - `createdAt`: Administrado por Postgres de forma automática indicando la hora de inserción física.
  - `updatedAt`: Administrado automáticamente por Prisma (`@updatedAt`) para trazabilidad de modificaciones y cierres.
  - `activationTime`: Controlado desde el Frontend (indicando el momento real en que se activó la alarma).
- **Índices de Alto Rendimiento (PostgreSQL)**:
  - `@@index([type, activationTime(sort: Desc)])`: Acelera búsquedas y conteos por tipo usando la fecha operativa.
  - `@@index([activationTime(sort: Desc)])`: Optimiza consultas globales y estadísticas mensuales.
  - `@@index([operatorId])`: Indexa la clave foránea en PostgreSQL para acelerar los `JOIN` y validaciones referenciales.
- **Cierre exclusivo del Código Verde**:
  - `closedBy`, `closedAt` y `closedByOperatorId` solo admiten valores para `GREEN`; los demás tipos mantienen estos campos en `NULL`. No existe un indicador `isClosed`: un código está abierto mientras `closedAt` es `NULL`.
  - Un cierre exige el funcionario que lo cierra (`closedBy`), la fecha (`closedAt >= activationTime`) y el operador que registra el cierre (`closedByOperatorId`, relación a `Operator` con `ON DELETE RESTRICT`). Los cierres históricos previos a este campo conservan `closedByOperatorId` en `NULL`.
- **Campos propios de Código Azul y Código Rojo**:
  - `teams` (`BlueTeam[]`: `EMERGENCY`, `ICU`, `PEDIATRIC_ICU`) admite uno o más equipos por Código Azul, sin repetidos; es una lista vacía en los demás tipos.
  - `cogridNotified` indica si hubo comunicación con COGRID y `cogridNotifiedAt` registra la fecha y hora de esa comunicación. Es opcional, solo se admite si `cogridNotified` es verdadero y no puede ser anterior a `activationTime`.

### Gestión de Conexiones y Seeding (Prisma ORM v7)
- **Configuración Centralizada**: Prisma 7 utiliza `prisma.config.ts` en la raíz del backend para la configuración del datasource `DATABASE_URL` en migraciones y herramientas CLI.
- **Ciclo de Vida de Conexiones**: `PrismaService` implementa `OnModuleInit` (`$connect()`) y `OnModuleDestroy` (`$disconnect()`) asegurando la liberación ordenada de conexiones en NestJS.
- **Sembrado Oficial (Seed)**: `pnpm run db:seed` ejecuta un `upsert` idempotente. Requiere `ADMIN_SEED_EMAIL` y `ADMIN_SEED_PASSWORD`; no existen credenciales administrativas fijas en el código.
- **Build de Producción**: `tsconfig.build.json` compila solo `src/`, por lo que el arranque es `dist/main.js`, como esperan `start:prod` y el `CMD` de `dockerfile.prod`. Si se incluyeran otros `.ts` de la raíz (por ejemplo `prisma/seed.ts` o `prisma.config.ts`), la salida pasaría a `dist/src/main.js` y el contenedor no arrancaría.
- **Migraciones de Producción**: `prisma migrate deploy` se ejecuta como un job de release independiente. La construcción de imágenes no se conecta a PostgreSQL.

## Arquitectura del Cliente (Frontend - Next.js)
### Componentes Universales
Para empatar con el STI del backend, el frontend no duplica vistas.
- **Formularios Dinámicos**: En `src/app/(code)/components/form/EmergencyCodeForm.tsx` utilizamos React Hook Form y renderizado condicional. Si la prop es `type="GREEN"`, el componente renderizará los inputs de Carabineros y Evento; si es `type="BLUE"`, mostrará la selección de uno o más equipos, etc.
- **Tablas Dinámicas**: `EmergencyCodeTable` toma el tipo, inyecta las columnas de `columns.tsx` correspondientes y procesa los modales de edición sin duplicar lógica de estado. Los textos libres se recortan a 2 líneas (`TruncatedText`) y el texto completo se ve en "Ver detalles". El menú de acciones (⋮) agrupa Ver detalles, Editar y Finalizar; sus diálogos viven fuera del menú y se controlan por estado. La paginación (`components/table/pagination.tsx`) usa `meta.totalPages` que devuelve el backend y permite elegir 5, 10, 20, 50 o 100 filas por página.
- **Server Actions Unificados**: En `src/actions/emergencyCodes/` tenemos `getEmergencyCodes` que recibe el `type` y despacha a la misma API, centralizando el caché y la revalidación.
- **Manejo de Estado y Caché (React Query)**: Utilizamos `useQuery` para el fetching dinámico desde el cliente en componentes de tabla, con `staleTime` para optimizar rendimiento. Al realizar mutaciones (crear/editar/cerrar) en `EmergencyCodeForm` o `CloseCodeModal`, se invalida programáticamente la caché (`queryClient.invalidateQueries`) para garantizar consistencia inmediata de los datos sin recargar la página.
- **Filtro por rango de fechas**: `SearchDate` guarda `from` y `to` (YYYY-MM-DD) en la URL y la tabla y el reporte PDF leen los mismos parámetros. `toRangeBounds` (`src/lib/date-range.ts`) los convierte al inicio del día "desde" y al final del día "hasta" en la zona horaria del navegador, de modo que ambos días queden completos; la API recibe esos instantes y filtra por `activationTime`. El PDF indica el rango en su título y un rango invertido se rechaza con 400. Al cambiar el filtro se vuelve a la primera página.
- **Reportes PDF**: el backend los genera con pdfmake en `GET /emergency-codes/report?type=...` y exige el token JWT, que un `<iframe>` o un enlace directo no pueden enviar (devolvían 401). `PdfRender` (`src/app/(code)/components/utils/PdfRender.tsx`) los pide con `emergency_codes.report` (fetch con el token de la sesión), los muestra desde un blob en el visor del navegador y ofrece "Descargar". Las fechas del PDF usan `formatDateTime` (DD/MM/YYYY) en vez de `toLocaleString()`, que en el contenedor de producción devuelve formato estadounidense.
- **Cierre de Código Verde**:
  - `CloseCodeModal` se abre desde la opción **Finalizar** del menú de acciones (⋮), solo para códigos `GREEN` abiertos, y exige el nombre de quien finaliza (`closedBy`), la fecha/hora (`closedAt`) y el operador que registra el cierre (`closedByOperatorId`, elegido de la lista de operadores).
  - Formulario Unificado (`EmergencyCodeForm`): Permite la captura precisa y manual de la hora de activación (`activationTime`), la hora de llamado a bomberos (`firefighterCalledTime` en Código Rojo) y los campos de cierre diferido.
  - Vistas de Creación Estandarizadas: Todas las rutas `/code-[tipo]/create` comparten una experiencia homogénea con navegación de retorno y botón unificado "Crear código [tipo]".

## Stack Tecnológico
- **Backend**: NestJS 10, Prisma ORM 7.x, PostgreSQL 14+.
- **Frontend**: Next.js 16 (App Router), TailwindCSS, shadcn/ui, Zustand (Opcional/Sidebar), React Hook Form, Zod.
- **Gestor de Paquetes**: `pnpm` (Estricta prohibición de usar `npm` puro para evitar colisiones).

## Operación de Migraciones

La migración desde las tablas históricas conserva IDs, operadores y fechas, copia `createdAt` histórico a `activationTime` y retiene las cinco tablas originales con sufijo legacy durante siete días. El procedimiento, verificaciones y límites de rollback están documentados en `docs/production-emergency-code-migration.md`.

---
> **Nota para Agentes de IA**: Antes de sugerir refactorizaciones, asegúrate de mantener el paradigma de Tabla Única y actualiza este documento ante cualquier cambio estructural.
