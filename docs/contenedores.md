# Contenedores: qué se cambió y por qué

Respuesta a la observación del auditor: «todas las imágenes se construyen en un solo paso sobre imágenes base completas, y
los procesos corren como root».

| Servicio | Lenguaje | Etapas | Imagen final | Usuario |
|---|---|---|---|---|
| catalog | Go | `golang:1.24-bookworm` → runtime | `gcr.io/distroless/base-debian12:nonroot` | `nonroot` (65532) |
| orders | Go | `golang:1.24-alpine` → runtime | `gcr.io/distroless/static-debian12:nonroot` | `nonroot` (65532) |
| cart | Python | venv en `python:3.12-slim` → runtime | `python:3.12-slim` | `app` (10001) |
| checkout | NestJS | build → deps de producción → runtime | `node:22-alpine` | `node` |
| ui | Node/TS | build → deps de producción → runtime | `node:22-alpine` | `node` |
| admin | Node/TS | build → deps de producción → runtime | `node:22-alpine` | `node` |

## Decisiones

- **Multi-stage:** compiladores, dependencias de desarrollo y código fuente quedan en las etapas de build; a la imagen
  final pasan solo el binario, o `dist/` más las dependencias de producción.
- **catalog usa distroless/base y no static:** su modo en memoria usa SQLite (`go-sqlite3`), que necesita CGO y glibc.
  `orders` no usa CGO, así que va sobre `distroless/static`, que es más chica.
- **cart usa `python:3.12-slim` y no alpine:** `psycopg2-binary` y `uvloop` traen ruedas precompiladas para glibc. Con
  musl habría que compilarlas, y eso agranda el build y suma riesgo.
- **Sin root:** cada imagen fija un `USER` sin privilegios. Las distroless `:nonroot` no tienen shell, lo que reduce la
  superficie de ataque y lo que puede hacer alguien que entre al contenedor.
- **`.dockerignore` por servicio:** deja fuera `node_modules`, `dist`, cachés, `.env` y `.git`. Así las builds son más
  rápidas y ningún secreto local llega a una imagen.
- **BD y Redis sin puertos publicados en el host:** en `docker-compose.yml` solo los alcanzan los servicios de la red
  interna. Únicamente `ui` (8080) y `admin` (8081) quedan expuestos.

## Verificación local

`docker compose build` y `up` con un `.env` de prueba:

- Tamaños: orders 27 MB, catalog 76 MB, ui 245 MB, admin 242 MB, cart 287 MB, checkout 324 MB. Las imágenes base
  completas de antes (`golang:1.24`, `node:22`, `python:3.12`) pesan cerca de 1 GB cada una.
- Usuario efectivo: `node` (uid 1000) en ui, admin y checkout; `app` (uid 10001) en cart; `nonroot` (uid 65532) en catalog
  y orders, que no tienen shell.
- Respuestas: tienda 200; `/api/catalog/products`, `/api/carts/<id>` y `/api/orders` 200 a través de la ui; admin 200;
  login del admin con las credenciales del `.env` 200; `admin/admin` 401.
- `admin` sin variables obligatorias se niega a arrancar con «Falta la variable de entorno obligatoria ADMIN_JWT_SECRET».
- BD y Redis aparecen sin puertos en el host (`5432/tcp`, `6379/tcp` solo internos).

## Hallazgos al construir

- `src/checkout/yarn.lock` no coincidía con `package.json`: tenía entradas de dependencias que ya no se usaban. El
  Dockerfile original lo ocultaba porque corría `yarn install` sin `--immutable`. Se regeneró el lockfile y ahora la
  build exige `--immutable`, así cada build instala exactamente las mismas versiones.
- `ui` y `admin` no tenían `package-lock.json`, así que sus dependencias no quedaban fijadas. Se generaron los
  lockfiles y los Dockerfiles instalan con `npm ci`.
