# Plan de Implementación: Actualización de Prisma y Optimización de Base de Datos

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Actualizar Prisma ORM a la última versión disponible, añadir índices de rendimiento y campos de auditoría al modelo STI `EmergencyCode`, eliminar extensiones fantasma, implementar `OnModuleDestroy` y extraer el sembrado a un script `seed.ts` dedicado.

**Architecture:** Mantiene el patrón Single Table Inheritance (STI) exigido por el proyecto, optimizando la persistencia en PostgreSQL con índices compuestos en `(type, createdAt DESC)`, clave foránea indexada y gestión limpia del ciclo de vida de conexiones en NestJS.

**Tech Stack:** NestJS, Prisma ORM (@prisma/client, prisma), PostgreSQL, pnpm, TypeScript.

**Spec:** `docs/superpowers/specs/2026-08-16-prisma-upgrade-db-audit-design.md`

## Global Constraints

- Gestor de dependencias exclusivo: `pnpm` (prohibido `npm`).
- Respetar arquitectura unificada: Mantener la tabla única `EmergencyCode` (Single Table Inheritance).
- Mantenimiento de documentación: Toda modificación estructural debe reflejarse en `DOCUMENTATION.md` usando la skill `doc-updater`.

---

### Task 1: Actualización de Dependencias de Prisma con `pnpm`

**Files:**
- Modify: `code-manager-backend/package.json`

**Interfaces:**
- Consumes: `@prisma/client`, `prisma`
- Produces: Versiones actualizadas de Prisma en `package.json` y `pnpm-lock.yaml`.

- [ ] **Step 1: Instalar la última versión de Prisma y Prisma Client usando pnpm**

Ejecutar en el directorio `code-manager-backend`:
```bash
pnpm add @prisma/client@latest
pnpm add -D prisma@latest
```

- [ ] **Step 2: Verificar instalación y versiones en `package.json`**

Inspeccionar `package.json` para comprobar que `prisma` y `@prisma/client` tienen las versiones actualizadas.

- [ ] **Step 3: Commit**

```bash
git add code-manager-backend/package.json code-manager-backend/pnpm-lock.yaml
git commit -m "chore(backend): upgrade prisma and @prisma/client to latest version"
```

---

### Task 2: Refactorización y Optimización de `schema.prisma`

**Files:**
- Modify: `code-manager-backend/prisma/schema.prisma`

**Interfaces:**
- Consumes: `schema.prisma` actual
- Produces: Esquema optimizado con índices compuestos (`[type, createdAt(sort: Desc)]`, `[createdAt(sort: Desc)]`, `[operatorId]`, `[isClosed]`), campo `updatedAt`, y eliminación de `postgresqlExtensions`/`vector`.

- [ ] **Step 1: Modificar `schema.prisma` con el nuevo diseño**

Reemplazar el contenido de `code-manager-backend/prisma/schema.prisma` con:
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
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

model EmergencyCode {
  id             String   @id @default(uuid())
  type           CodeType
  activeBy       String
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt
  activationTime DateTime
  location       String
  operatorId     String
  operator       Operator @relation(fields: [operatorId], references: [id])
  observations   String?

  // Ciclo de vida y estado universal
  isClosed Boolean?  @default(false)
  closedBy String?
  closedAt DateTime?

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

  // Índices de alto rendimiento
  @@index([type, createdAt(sort: Desc)])
  @@index([createdAt(sort: Desc)])
  @@index([operatorId])
  @@index([isClosed])
}

model Operator {
  id             String          @id @default(uuid())
  name           String
  emergencyCodes EmergencyCode[]
  createdAt      DateTime        @default(now())
  updatedAt      DateTime        @updatedAt
}
```

- [ ] **Step 2: Generar el cliente de Prisma para validar el schema**

Ejecutar:
```bash
pnpm exec prisma generate
```
Verificar que la generación termine con éxito sin errores de sintaxis.

- [ ] **Step 3: Commit**

```bash
git add code-manager-backend/prisma/schema.prisma
git commit -m "refactor(db): add performance indexes and audit fields to schema.prisma"
```

---

### Task 3: Refactorización de `PrismaService` y Limpieza de Código Muerto

**Files:**
- Modify: `code-manager-backend/src/prisma/prisma.service.ts`
- Modify: `code-manager-backend/src/prisma/prisma.module.ts`
- Modify: `code-manager-backend/src/app.module.ts`
- Delete: `code-manager-backend/src/prisma/prisma.controller.ts`

**Interfaces:**
- Consumes: `@prisma/client`
- Produces: `PrismaService` con `$connect()` en `onModuleInit()` y `$disconnect()` en `onModuleDestroy()`, sin sembrado hardcodeado.

- [ ] **Step 1: Actualizar `src/prisma/prisma.service.ts`**

```ts
import { Injectable, Logger, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  private readonly logger = new Logger('PrismaService');

  public async onModuleInit() {
    await this.$connect();
    this.logger.log('Connected to the database');
  }

  public async onModuleDestroy() {
    await this.$disconnect();
    this.logger.log('Disconnected from the database');
  }
}
```

- [ ] **Step 2: Eliminar `src/prisma/prisma.controller.ts` y limpiar `src/prisma/prisma.module.ts`**

En `code-manager-backend/src/prisma/prisma.module.ts`:
```ts
import { Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

- [ ] **Step 3: Verificar imports en `src/app.module.ts`**

Asegurar que `app.module.ts` siga importando correctamente `PrismaModule`.

- [ ] **Step 4: Commit**

```bash
git add code-manager-backend/src/prisma/
git commit -m "refactor(backend): improve PrismaService lifecycle management and remove dead controller"
```

---

### Task 4: Script de Seed Dedicado (`prisma/seed.ts`)

**Files:**
- Create: `code-manager-backend/prisma/seed.ts`
- Modify: `code-manager-backend/package.json`

**Interfaces:**
- Consumes: `@prisma/client`, `bcryptjs`
- Produces: Comando `pnpm exec prisma db seed` para inicialización de base de datos idempotente.

- [ ] **Step 1: Crear `code-manager-backend/prisma/seed.ts`**

```ts
import { PrismaClient, Role } from '@prisma/client';
import * as bcryptjs from 'bcryptjs';

const prisma = new PrismaClient();

async function main() {
  const adminEmail = 'prueba@gmail.com';
  const hashedPassword = bcryptjs.hashSync('prueba', 10);

  const admin = await prisma.user.upsert({
    where: { email: adminEmail },
    update: {},
    create: {
      email: adminEmail,
      name: 'Administrador Inicial',
      password: hashedPassword,
      role: Role.Admin,
      isActive: true,
    },
  });

  console.log(`Seed completed successfully: Admin user ready (${admin.email})`);
}

main()
  .catch((e) => {
    console.error('Error executing seed:', e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

- [ ] **Step 2: Configurar `"prisma": { "seed": "ts-node prisma/seed.ts" }` en `code-manager-backend/package.json`**

Añadir la propiedad en `package.json`:
```json
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
```

- [ ] **Step 3: Commit**

```bash
git add code-manager-backend/prisma/seed.ts code-manager-backend/package.json
git commit -m "feat(db): add dedicated prisma seed script"
```

---

### Task 5: Migración, Build y Actualización de Documentación

**Files:**
- Modify: `DOCUMENTATION.md`
- Generates: Nueva migración en `code-manager-backend/prisma/migrations/`

**Interfaces:**
- Consumes: `doc-updater` skill
- Produces: Base de datos sincronizada, compilación NestJS verificada y `DOCUMENTATION.md` actualizado.

- [ ] **Step 1: Ejecutar migración de Prisma o verificar diff de migración**

Ejecutar:
```bash
pnpm exec prisma migrate dev --name optimize_emergency_code_indexes_and_cleanup
```

- [ ] **Step 2: Validar compilación del backend**

Ejecutar:
```bash
pnpm run build
```
Verificar que compile con código 0 sin errores de tipado en NestJS.

- [ ] **Step 3: Actualizar `DOCUMENTATION.md` con la skill `doc-updater`**

Invocar `doc-updater` para reflejar en `DOCUMENTATION.md`:
1. Actualización de Prisma a la última versión.
2. Índices de rendimiento añadidos a la tabla STI `EmergencyCode`.
3. Procedimiento formal de `seed` con `pnpm exec prisma db seed`.

- [ ] **Step 4: Commit final**

```bash
git add DOCUMENTATION.md code-manager-backend/prisma/migrations/
git commit -m "docs: update architecture docs and persist migration"
```
