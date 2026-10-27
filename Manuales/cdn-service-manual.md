---
tipo: manual
titulo: "Manual del servicio CDN — docs-service"
autor: "Efraín García"
creado: 2026-10-06
actualizado: 2026-10-06
estado: vigente
proyecto: tecmm
tags:
  - cdn
  - docs-service
  - api
  - caddy
  - privados
  - manual
---

# Manual del servicio CDN — `docs-service`

> Servicio de carga, procesamiento y consulta de archivos del CDN institucional. Optimiza imágenes (JPEG/PNG → WebP), indexa metadatos en PostgreSQL y publica los archivos bajo `tecmm.mx/cdn/...` a través de **Caddy**. Este manual cubre el consumo de la API, incluido el flujo de **lectura de archivos privados**. Relacionado con [[to-production-manual]], [[repositorio-imagenes]] y [[Proyecto]].

- **Dominio:** `https://tecmm.mx`
- **Rutas de archivo:** `/cdn/public/...` (abiertas) y `/cdn/private/...` (autorizadas).
- **Backend:** `upload-service` (imagen `cdn-service:latest`), escucha en `8080` dentro de la red del stack.
- **Proxy:** Caddy valida `X-Api-Key` y sirve `/cdn/*`.

## Resumen

- `POST /api/upload` — subida normal; procesa y responde metadata final.
- `POST /api/uploadbatch` — subida por lote; encola procesamiento asíncrono cuando aplica.
- `GET /api/files/{id}` — metadata, estado y `publicUrl`.
- `GET /api/files` — listado administrativo (solo `DOCS_ADMIN`).
- `GET /api/cdn/private-access/{ruta}` — autorización de lectura privada, la invoca **Caddy** (no las apps) con `forward_auth`.

## Requisitos previos

Headers obligatorios en toda llamada a la API:

| Header | Rol |
| --- | --- |
| `X-Api-Key` | Deja pasar por Caddy. |
| `X-User-Key` | Identifica al usuario y define sus permisos. |

Las apps que consumen el servicio deben exponer un endpoint propio para recibir la nueva `api-key` cuando se rote por pub-sub.

## Uso

### Modelo de colecciones

```text
user_collection -> app_collection -> collection_identifier
       └──────────────────────────> files.collection_id (compatibilidad histórica)
```

Cada upload envía los tres datos de resolución:

| Campo | Significado |
| --- | --- |
| `collectionId` | `user_collection.id`, raíz de la app. |
| `appCollectionId` | `app_collection.id`, colección lógica hija de esa app. |
| `collectionIdentifier` | Nombre del identificador dentro de la colección lógica. |

Reglas: `appCollectionId` debe pertenecer al `collectionId`; `collectionIdentifier` acepta de 1 a 150 caracteres con letras, números, `_` y `-` (si ya existe se reutiliza, si no se crea). El nombre de `app_name` y de `app_collection` también se restringe a ese charset en origen (es parte de la ruta de almacenamiento).

Rutas construidas:

- Pública: `public/{app_name}/{app_collection}/{collection_identifier}/{file}`
- Privada: `private/{app_name}/{app_collection}/{collection_identifier}/{file}`

URLs publicadas (segmento `/cdn/`):

- `https://tecmm.mx/cdn/public/WEB-TSJ/DOCS/GENERAL/archivo.pdf`
- `https://tecmm.mx/cdn/private/BOLSA-TSJ/DOCS/COMPANY/archivo.pdf`

### Upload normal

```http
POST /api/upload
Content-Type: multipart/form-data
```

| Campo | Tipo | Requerido | Descripción |
| --- | --- | --- | --- |
| `file` | File | Sí | Archivo a subir (la parte debe llamarse `file`). |
| `collectionId` | Number | Sí | `user_collection.id`. |
| `appCollectionId` | Number | Sí | `app_collection.id` hija de `collectionId`. |
| `collectionIdentifier` | String | Sí | Identificador de ruta. Si no existe, se crea. |
| `isPrivate` | Boolean | No | Default `false`. Con `true` se guarda bajo `private/...` y se exigirá `private_route_key` para leerlo. |

```bash
curl -X POST https://tecmm.mx/api/upload \
  -H "X-Api-Key: <api-key>" \
  -H "X-User-Key: <user-key>" \
  -F "file=@imagen.png" \
  -F "collectionId=1" \
  -F "appCollectionId=2" \
  -F "collectionIdentifier=GENERAL" \
  -F "isPrivate=false"
```

Responde `200 OK` con `id`, `originalName`, `contentType`, `format`, `publicUrl`, `sizeBytes`, `originalSizeBytes`, `sha256`, `isPrivate`, `createdAt`, `collectionId` y `status` (en normal suele ser `ready`).

### Upload batch async

```http
POST /api/uploadbatch
Content-Type: multipart/form-data
```

| Campo | Tipo | Requerido | Descripción |
| --- | --- | --- | --- |
| `files` | File[] | Sí | Uno o más archivos (cada parte llamada `files`). |
| `collectionId` | Number | Sí | `user_collection.id`. |
| `appCollectionId` | Number | Sí | `app_collection.id`. |
| `collectionIdentifier` | String | Sí | Identificador compartido por el lote. |
| `isPrivate` | Boolean | No | Default `false`. |

Responde `200 OK` con `items: [{ id, status }]`. `queued` significa aceptado sin metadata final todavía; consultar cada `id` hasta `ready`.

### Consultar y listar

```bash
curl https://tecmm.mx/api/files/123 -H "X-Api-Key: <api-key>" -H "X-User-Key: <user-key>"
curl "https://tecmm.mx/api/files?limit=50&offset=0" -H "X-Api-Key: <api-key>" -H "X-User-Key: <user-key>"
```

Estados: `queued` · `processing` · `ready` · `failed`.

### Autorizar lectura privada

Este endpoint **no lo consumen las apps**: lo invoca Caddy con `forward_auth` antes de servir cualquier archivo bajo `/cdn/private/*`. Sirve para depurar permisos con `curl`.

```http
GET /api/cdn/private-access/{appName}/{appCollection}/{collectionIdentifier}/{file}
```

La ruta es la misma del archivo, sin los segmentos `cdn/` ni `private/`.

| Header | Descripción |
| --- | --- |
| `X-Api-Key` | Requerido por Caddy. |
| `X-User-Key` | Identifica al usuario. |
| `X-Private-Route-Key` | `private_route_key` de la app dueña de la colección. |

La autorización exige las dos condiciones a la vez: que el usuario tenga acceso a la colección **y** que la `private_route_key` corresponda a la app.

| Rol | Acceso a privados |
| --- | --- |
| `DOCS_ADMIN` | Cualquier privado. |
| `COLLECTION_MANAGER` | Solo los de sus colecciones. |
| `UPLOADER` | Ninguno. |

```bash
curl -i https://tecmm.mx/api/cdn/private-access/BOLSA-TSJ/DOCS/COMPANY/archivo.pdf \
  -H "X-Api-Key: <api-key>" \
  -H "X-User-Key: <user-key>" \
  -H "X-Private-Route-Key: <private-route-key-de-la-app>"
```

Respuesta cuando Caddy puede servir el archivo (`200 OK`):

```json
{ "allowed": true, "reason": "allowed" }
```

Cualquier otra respuesta hace que Caddy bloquee la descarga. El campo `reason` (visible solo al depurar; Caddy lo reemplaza por un `404` genérico) sirve para diagnóstico:

| Respuesta | `reason` | Causa |
| --- | --- | --- |
| `200` | `allowed` | Permiso concedido. |
| `401` | n/a | `X-User-Key` ausente, inactivo o incorrecto. |
| `404` | `invalid_path` | La ruta no tiene forma de ruta privada. |
| `404` | `not_found` | El archivo no existe o no es privado. |
| `404` | `missing_private_route_key` | No se envió `X-Private-Route-Key`. |
| `404` | `invalid_private_route_key` | La key no corresponde a la app de la colección. |
| `404` | `forbidden_collection` | El usuario no es dueño de la colección. |

Un privado inexistente y uno no autorizado devuelven ambos `404`, para no revelar qué rutas privadas existen. La `private_route_key` se pide al equipo backend para la app correspondiente y se compara en tiempo constante contra `user_collection.private_route_key`.

### Roles

| Rol | Permisos |
| --- | --- |
| `UPLOADER` | `POST /api/upload`. |
| `COLLECTION_MANAGER` | `POST /api/upload`, `POST /api/uploadbatch` y consultar sus archivos. |
| `DOCS_ADMIN` | Upload normal y batch, consultar cualquier archivo y listar. |

### Rate limit

Los límites se aplican por `X-User-Key`; sin ese header, por IP.

| Política | Endpoints | Límite |
| --- | --- | --- |
| `work-limiter` | `POST /api/upload`, `POST /api/uploadbatch` | Token bucket de 100, repone 20 cada 10 min. |
| `consult-limiter` | `GET /api/files/{id}`, `GET /api/files` | Token bucket de 50, repone 10 por minuto. |
| `cdn-limiter` | `GET /api/cdn/private-access/{ruta}` | Token bucket de 1000, repone 500 por minuto. |

`cdn-limiter` es alto a propósito: corre en cada descarga privada y una página con muchas imágenes privadas no debe recibir `429`. Se particiona por `X-Private-Route-Key`, luego `X-User-Key`, y por IP como último recurso. Al excederse: `429 Too Many Requests`.

## Arquitectura

- **Caddy** valida `X-Api-Key` y sirve `/cdn/*`. Para `/cdn/public/*` sirve directo desde `/srv/documents-cdn/public`.
- Para `/cdn/private/*`, Caddy hace `forward_auth` contra `upload-service:8080`:
  - Reescribe la ruta a `/api/cdn/private-access{uri}` (sin `cdn/private`) y reenvía `X-User-Key` y `X-Private-Route-Key` del cliente.
  - `200` → sirve el archivo desde `/srv/documents-cdn/private` con `Cache-Control: private, no-cache, no-store`. Cualquier otro código → `404` genérico.
- **upload-service** indexa metadatos y autoriza; **no sirve** archivos. Escribe en `/storage` (`Storage__Root`), el mismo directorio del host que Caddy monta como `/srv/documents-cdn`.
- `StoredPath` guarda `private/...` o `public/...`; la autorización resuelve por coincidencia exacta de `stored_path`.

## Troubleshooting

| Síntoma | Causa / solución |
| --- | --- |
| `401` de Caddy | Falta el header `X-Api-Key` o no coincide con `CDN_API_KEY`. |
| `401` del backend | `X-User-Key` ausente, inactivo o incorrecto. |
| `404` al leer un privado | Falta `X-Private-Route-Key`, no corresponde a la app, el usuario no es dueño, o el archivo no existe. |
| `403` en `/cdn/private/*` | Caddy todavía tiene el bloque privado en modo bloqueo; falta habilitar el `forward_auth` a `upload-service`. |
| `429` | Se excedió el rate limit; revisar la política aplicada. |
| `queued` sin `publicUrl` | Es normal en batch: consultar `GET /api/files/{id}` hasta `ready`. |

## Relacionado

- [[to-production-manual]] — despliegue del servicio (`to-production.sh`).
- [[repositorio-imagenes]] — catálogo de imágenes públicas del CDN.
- [[Proyecto]] — estatus general del proyecto.
