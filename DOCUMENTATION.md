# Code Manager - Arquitectura y Documentación Oficial

## Visión General
Esta aplicación gestiona códigos de emergencia en un entorno hospitalario o corporativo (Código Azul, Verde, Rojo, Aéreo y Fuga).

## Arquitectura de Datos (Backend - NestJS & Prisma)
### Patrón Aplicado: Single Table Inheritance (STI)
En lugar de tener una tabla en la base de datos para cada tipo de código de emergencia (lo cual genera duplicación y consultas complejas), **toda la información reside en una única tabla unificada llamada `EmergencyCode`**. 

- **Estrategia**: Se utiliza el enum `CodeType` (`GREEN`, `BLUE`, `AIR`, `RED`, `LEAK`) para discriminar qué tipo de emergencia representa cada fila.
- **Ventajas**:
  1. Escalabilidad: Agregar un nuevo código de emergencia solo requiere añadir un tipo al Enum y agregar las columnas opcionales a la tabla unificada, sin crear nuevas tablas.
  2. DTOs Dinámicos: En NestJS utilizamos `class-validator` con `@ValidateIf(o => o.type === CodeType.XXX)` para garantizar que el Payload entrante solo exija los campos de su código correspondiente.
  3. Análisis unificado: Para obtener totales por mes de **todas** las emergencias conjuntas basta con una sola consulta.

### Base de Datos & Rendimiento
- **IDs**: UUIDv4 para mayor seguridad.
- **Fechas y Auditoría**: 
  - `createdAt`: Administrado por Postgres de forma automática indicando la hora de inserción física.
  - `updatedAt`: Administrado automáticamente por Prisma (`@updatedAt`) para trazabilidad de modificaciones y cierres.
  - `activationTime`: Controlado desde el Frontend (indicando el momento real en que se activó la alarma).
- **Índices de Alto Rendimiento (PostgreSQL)**:
  - `@@index([type, createdAt(sort: Desc)])`: Acelera búsquedas y conteos paginados filtrados por tipo de emergencia y rango temporal.
  - `@@index([createdAt(sort: Desc)])`: Optimiza consultas agregadas globales y estadísticas mensuales.
  - `@@index([operatorId])`: Indexa la clave foránea en PostgreSQL para acelerar los `JOIN` y validaciones referenciales.
  - `@@index([isClosed])`: Filtro eficiente de emergencias activas frente a finalizadas.
- **Ciclo de Vida Universal de Emergencias**:
  - Los campos `isClosed`, `closedBy` y `closedAt` son transversales a todos los códigos de emergencia, permitiendo el cierre y trazabilidad operativa diferida.

### Gestión de Conexiones y Seeding (Prisma ORM v7)
- **Configuración Centralizada**: Prisma 7 utiliza `prisma.config.ts` en la raíz del backend para la configuración del datasource `DATABASE_URL` en migraciones y herramientas CLI.
- **Ciclo de Vida de Conexiones**: `PrismaService` implementa `OnModuleInit` (`$connect()`) y `OnModuleDestroy` (`$disconnect()`) asegurando la liberación ordenada de conexiones en NestJS.
- **Sembrado Oficial (Seed)**: El sembrado inicial de datos es idempotente y se ejecuta mediante `pnpm exec prisma db seed` configurado en `package.json` hacia `prisma/seed.ts` (creación del usuario administrador por defecto).

## Arquitectura del Cliente (Frontend - Next.js)
### Componentes Universales
Para empatar con el STI del backend, el frontend no duplica vistas.
- **Formularios Dinámicos**: En `src/app/(code)/components/form/EmergencyCodeForm.tsx` utilizamos React Hook Form y renderizado condicional. Si la prop es `type="GREEN"`, el componente renderizará los inputs de Carabineros y Evento; si es `type="BLUE"`, mostrará Equipo, etc.
- **Tablas Dinámicas**: `EmergencyCodeTable` toma el tipo, inyecta las columnas de `columns.tsx` correspondientes y procesa los modales de edición sin duplicar lógica de estado.
- **Server Actions Unificados**: En `src/actions/emergencyCodes/` tenemos `getEmergencyCodes` que recibe el `type` y despacha a la misma API, centralizando el caché y la revalidación.
- **Manejo de Estado y Caché (React Query)**: Utilizamos `useQuery` para el fetching dinámico desde el cliente en componentes de tabla, con `staleTime` para optimizar rendimiento. Al realizar mutaciones (crear/editar) en el `EmergencyCodeForm`, se invalida programáticamente la caché (`queryClient.invalidateQueries`) para garantizar frescura inmediata de los datos al retornar a la tabla.
- **Cierre de Códigos (Flujo Operativo)**: Los códigos pueden ser "Finalizados" en diferido (editando el registro existente). Esto se maneja dinámicamente en el formulario habilitando los campos `isClosed`, `closedBy` y `closedAt`.

## Stack Tecnológico
- **Backend**: NestJS 10, Prisma ORM 7.x, PostgreSQL 14+.
- **Frontend**: Next.js 16 (App Router), TailwindCSS, shadcn/ui, Zustand (Opcional/Sidebar), React Hook Form, Zod.
- **Gestor de Paquetes**: `pnpm` (Estricta prohibición de usar `npm` puro para evitar colisiones).

---
> **Nota para Agentes de IA**: Antes de sugerir refactorizaciones, asegúrate de mantener el paradigma de Tabla Única y actualiza este documento ante cualquier cambio estructural.
