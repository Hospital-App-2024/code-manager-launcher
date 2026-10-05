# Code Manager Launcher

# Instalación de la aplicación en Producción

Todos los comandos se ejecutan **desde la raíz** del repositorio.

1. Clone el repositorio con los submódulos (si ya lo clonó sin ellos, las carpetas `code-manager-backend` y `code-manager-frontend` quedan vacías y el build falla)

```bash
git clone --recurse-submodules https://github.com/Hospital-App-2024/code-manager-launcher.git
```

```bash
git submodule update --init --recursive
```

2. Cree el archivo `.env` en la raíz a partir de `.env.example` y ajuste los valores:
   - `DATABASE_URL`: PostgreSQL accesible desde el contenedor. Si la base corre en el mismo equipo use `host.docker.internal` (funciona en Windows, Mac y Linux).
   - `URL_BACKEND`: backend visto **desde el contenedor del frontend**: `http://code-manager-backend:<CODE_MANAGER_BACKEND_PORT>/api` (no `localhost` ni la IP). El navegador no llama al backend directo: pasa por el frontend (`/api/backend`), así que no hace falta configurar la IP ni abrir el puerto del backend.
   - `NEXTAUTH_URL`: URL con la que los usuarios abren la app, p. ej. `http://<IP-del-servidor>:8080`.
   - `AUTH_SECRET`, `JWT_SECRET`, `JWT_REFRESH_SECRET`: valores propios de esta instalación.

```bash
cp .env.example .env
```

3. Construya las imágenes (`--pull` actualiza la imagen base de Node si en ese equipo hay una vieja en caché)

```bash
docker compose -f docker-compose.prod.yml --profile migration build --pull
```

4. Aplique las migraciones de Prisma (job aparte; no se ejecuta con `up`)

```bash
docker compose -f docker-compose.prod.yml --profile migration run --rm code-manager-migrate
```

5. Levante la aplicación

```bash
docker compose -f docker-compose.prod.yml up -d
```

### Comandos de Prisma en producción

Todos usan el servicio `code-manager-migrate` (imagen del backend con dependencias de desarrollo y la `DATABASE_URL` del `.env` de la raíz).

| Acción | Comando |
|---|---|
| Aplicar migraciones pendientes | `docker compose -f docker-compose.prod.yml --profile migration run --rm code-manager-migrate` |
| Ver estado de migraciones | `docker compose -f docker-compose.prod.yml --profile migration run --rm code-manager-migrate pnpm exec prisma migrate status` |
| Crear usuario administrador inicial (requiere `ADMIN_SEED_*` en `.env`) | `docker compose -f docker-compose.prod.yml --profile migration run --rm code-manager-migrate pnpm run db:seed` |
| Versión de Prisma | `docker compose -f docker-compose.prod.yml --profile migration run --rm code-manager-migrate pnpm exec prisma --version` |

Si se actualiza el código (`git pull` + `git submodule update`), reconstruya y repita los pasos 3 a 5.

### Problemas frecuentes

- **`Cannot find matching keyid`** o errores de corepack/pnpm: reconstruir con `--pull`. Los Dockerfile ya instalan pnpm con npm, sin corepack.
- **`permission denied` sobre `code-manager-backend/postgres`** al construir: es la carpeta de datos de la base local; ya está excluida en `.dockerignore`.
- **No hace login o no cargan los datos**: revisar que `URL_BACKEND` use el nombre del servicio (`code-manager-backend`) y el puerto de `CODE_MANAGER_BACKEND_PORT`, no `localhost`. Los logs del frontend muestran el error: `docker compose -f docker-compose.prod.yml logs code-manager-frontend`.
- **`Falta XXX en .env`**: no existe el `.env` en la raíz o le falta esa variable.

### Pasos para crear los Git Submodules

1. Crear un nuevo repositorio en GitHub
2. Clonar el repositorio en la máquina local
3. Añadir el submodule, donde `repository_url` es la url del repositorio y `directory_name` es el nombre de la carpeta donde quieres que se guarde el sub-módulo (no debe de existir en el proyecto)

```
git submodule add <repository_url> <directory_name>
```

4. Añadir los cambios al repositorio (git add, git commit, git push)
   Ej:

```
git add .
git commit -m "Add submodule"
git push
```

5. Inicializar y actualizar Sub-módulos, cuando alguien clona el repositorio por primera vez, debe de ejecutar el siguiente comando para inicializar y actualizar los sub-módulos

```
git submodule update --init --recursive
```

6. Para actualizar las referencias de los sub-módulos

```
git submodule update --remote
```

## Importante

Si se trabaja en el repositorio que tiene los sub-módulos, **primero actualizar y hacer push** en el sub-módulo y **después** en el repositorio principal.

Si se hace al revés, se perderán las referencias de los sub-módulos en el repositorio principal y tendremos que resolver conflictos.
