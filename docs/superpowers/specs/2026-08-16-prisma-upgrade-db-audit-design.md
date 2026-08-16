# Especificación de Diseño: Actualización de Prisma y Optimización de Base de Datos

- **Fecha**: 2026-08-16
- **Estado**: Aprobado por el usuario
- **Alcance**: Backend (`code-manager-backend`)

---

## 1. Contexto y Objetivos

La aplicación gestiona códigos de emergencia en un entorno hospitalario/corporativo siguiendo una arquitectura basada en **Single Table Inheritance (STI)** sobre una tabla unificada `EmergencyCode`.

### Objetivos Principales:
1. Actualizar Prisma ORM y `@prisma/client` a la última versión disponible utilizando exclusivamente `pnpm`.
2. Resolver deficiencias críticas en la capa de persistencia:
   - Añadir índices estratégicos (`[type, createdAt]`, `[createdAt]`, `[operatorId]`, `[isClosed]`) para evitar scans secuenciales en PostgreSQL.
   - Eliminar configuraciones y extensiones obsoletas/no utilizadas (`previewFeatures = ["postgresqlExtensions"]`, `extensions = [vector]`).
   - Normalizar los campos de ciclo de vida (`isClosed`, `closedBy`, `closedAt`) y agregar auditoría temporal (`updatedAt`).
3. Eliminar antipatrones en el código:
   - Extraer el sembrado de datos (seed) de `onModuleInit` a un script dedicado `prisma/seed.ts`.
   - Implementar `OnModuleDestroy` (`$disconnect`) en `PrismaService`.
   - Eliminar controladores vacíos/muertos (`PrismaController`).

---

## 2. Esquema de Base de Datos Optimizado (`schema.prisma`)

### 2.1 Configuración de Generador y Datasource
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

### 2.2 Modelos

```prisma
enum Role {
  User
  Admin
  Operator
}

enum CodeType {
  GREEN
  BLUE
  AIR
  RED
  LEAK
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  role      Role     @default(User)
  isActive  Boolean  @default(false)
}

model Operator {
  id             String          @id @default(uuid())
  name           String
  emergencyCodes EmergencyCode[]
  createdAt      DateTime        @default(now())
  updatedAt      DateTime        @updatedAt
}

model EmergencyCode {
  id             String   @id @default(uuid())
  type           CodeType
  activeBy       String
  activationTime DateTime
  location       String
  operatorId     String
  operator       Operator @relation(fields: [operatorId], references: [id])
  observations   String?

  // Ciclo de vida y auditoría (Universales)
  isClosed  Boolean?  @default(false)
  closedBy  String?
  closedAt  DateTime?
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt

  // Campos específicos Código Verde
  event  String?
  police Boolean?

  // Campos específicos Código Azul
  team String?

  // Campos específicos Código Aéreo
  emergencyDetail String?

  // Campos específicos Código Rojo
  COGRID                Boolean?
  firefighterCalledTime DateTime?

  // Campos específicos Código Fuga
  patientName        String?
  patientDescription String?

  // Índices de rendimiento
  @@index([type, createdAt(sort: Desc)])
  @@index([createdAt(sort: Desc)])
  @@index([operatorId])
  @@index([isClosed])
}
```

---

## 3. Arquitectura del Servicio Prisma y Seeding

### 3.1 `PrismaService` (`src/prisma/prisma.service.ts`)
- Implementa `OnModuleInit` con `$connect()` y manejo de logs.
- Implementa `OnModuleDestroy` con `$disconnect()` para garantizar el cierre ordenado del pool de conexiones.
- Remueve la llamada de creación de usuarios por defecto en tiempo de arranque.

### 3.2 Script de Seed (`prisma/seed.ts`)
- Crea el usuario administrador inicial (`prueba@gmail.com`) de forma idempotente con `upsert`.
- Configurado en `package.json`:
  ```json
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
  ```

### 3.3 Limpieza de Módulos (`src/prisma/prisma.module.ts`)
- Eliminar `PrismaController` y exportar únicamente `PrismaService`.
- Eliminar el archivo `src/prisma/prisma.controller.ts`.

---

## 4. Estrategia de Migración y Compatibilidad

1. **Retrocompatibilidad**: Todas las columnas existentes mantienen sus tipos y nombres. La adición de `updatedAt` e índices no rompe datos existentes en producción.
2. **Generación de Migración**: Ejecutar `pnpm exec prisma migrate dev --name optimize_emergency_code_indexes_and_cleanup`.
3. **Regeneración de Cliente**: Ejecutar `pnpm exec prisma generate`.
4. **Verificación de Tipos y Compilación**: `pnpm build` sin errores.

---

## 5. Actualización de Documentación

Actualizar `DOCUMENTATION.md` en la raíz del proyecto para registrar:
- Las nuevas versiones de Prisma.
- La estrategia de indexación añadida al patrón STI.
- El procedimiento de seeding oficial con `pnpm exec prisma db seed`.
