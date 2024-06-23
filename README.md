# Code Manager Launcher

# Instalación de la aplicación en Producción
1. Clone el repositorio
2. Reconstruya los submodules
````bash
git submodule update --init --recursive
````
3. Construya la imagen de docker
````bash
docker compose -f docker-compose.yml build
````
