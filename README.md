# EKSApp

Aplicacion base de laboratorio para administrar una pequena app de notas con frontend Nuxt, backend Laravel y PostgreSQL. El repositorio queda preparado para ejecucion local con Docker Compose y para construir imagenes aptas para un despliegue posterior en Kubernetes detras de un unico Ingress.

## Arquitectura

- `frontend`: Nuxt 3.13, Node 20, servido en modo produccion con `node .output/server/index.mjs`.
- `backend`: Laravel 11, PHP 8.2, Nginx y PHP-FPM en un unico contenedor para simplificar Compose.
- `postgres`: PostgreSQL 16 con volumen persistente.
- `migrate`: job explicito que ejecuta migraciones y termina.
- `gateway`: Nginx local que reproduce el comportamiento esperado del Ingress.

Ruteo previsto:

- `/` -> frontend
- `/api` -> backend Laravel
- `/health` y `/ready` -> healthchecks del backend

## Requisitos

- Docker
- Docker Compose v2

## Configuracion inicial

```bash
git clone <repositorio>
cd EKSApp
cp .env.example .env
```

Para un entorno real, genera un `APP_KEY` propio y cambia las credenciales de base de datos. El valor incluido en `.env.example` es solo un ejemplo local.

## Ejecucion local con Docker

```bash
docker compose up --build
```

Compose construye frontend y backend, espera a PostgreSQL, ejecuta `migrate` una vez y luego inicia la aplicacion detras del gateway.

## Migraciones

Las migraciones no se ejecutan al arrancar cada replica del backend. Se ejecutan con el servicio dedicado:

```bash
docker compose run --rm migrate
```

`docker compose up --build` tambien puede ejecutar `migrate` antes del backend usando `depends_on: condition: service_completed_successfully`. Si tu version de Compose no respeta esa condicion, ejecuta primero el comando anterior.

## URLs de acceso

Con los valores por defecto:

- Aplicacion: `http://localhost:8080`
- API de notas: `http://localhost:8080/api/notes`
- Healthcheck: `http://localhost:8080/health`
- Readiness: `http://localhost:8080/ready`

Endpoints principales:

- `GET /api/notes`
- `POST /api/notes`

## Variables principales

- `APP_DOMAIN`, `APP_SCHEME`, `APP_URL`, `FRONTEND_URL`: dominio y URLs publicas.
- `API_BASE_PATH`, `API_PUBLIC_URL`, `NUXT_PUBLIC_API_BASE`: ruta publica de API. Por defecto, `/api`.
- `DB_CONNECTION`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`: conexion PostgreSQL.
- `APP_ENV`, `APP_DEBUG`, `APP_KEY`, `LOG_CHANNEL`, `LOG_LEVEL`: configuracion Laravel.
- `TRUSTED_PROXIES`: proxies confiables para `X-Forwarded-*`.
- `CORS_ALLOWED_ORIGINS`, `CORS_SUPPORTS_CREDENTIALS`: CORS opcional para desarrollo.
- `GATEWAY_PORT`: puerto local publicado por el gateway.

## Comandos utiles

```bash
docker compose ps
docker compose logs -f backend
docker compose logs -f frontend
docker compose run --rm backend php artisan route:list
docker compose run --rm migrate
```

## Detencion y limpieza

```bash
docker compose down
docker compose down -v
```

`down -v` elimina el volumen de PostgreSQL y, por lo tanto, los datos locales.

## Resolucion de problemas frecuentes

- Si Laravel indica que falta `APP_KEY`, revisa que exista `.env` y que `APP_KEY` tenga un valor valido.
- Si `/ready` responde `503`, PostgreSQL no esta disponible o las variables `DB_*` no coinciden.
- Si el frontend no carga notas, verifica que `NUXT_PUBLIC_API_BASE=/api` y accede por el gateway, no por el puerto interno del frontend.
- Si Compose no espera la migracion, ejecuta `docker compose run --rm migrate` antes de `docker compose up`.

## Consideraciones para Kubernetes

No se incluyen manifiestos de Kubernetes, Terraform ni configuracion de AWS. Para desplegar despues:

- Publica un unico Ingress para el dominio de la aplicacion.
- Enruta `/` al servicio del frontend y `/api` al servicio del backend.
- Inyecta configuracion con variables de entorno o secretos del cluster.
- Ejecuta migraciones como Job separado antes de iniciar o actualizar replicas del backend.
- Configura `TRUSTED_PROXIES` con los rangos o direcciones del proxy/Ingress confiable.
