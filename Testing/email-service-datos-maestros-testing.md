---
tipo: testing
titulo: EmailSendService Fase 2 (datos maestros)
autor: Efraín García
creado: 2026-11-03
actualizado: 2026-11-03
estado: vigente
proyecto: email-service
tags:
  - testing
  - qa
  - email-service
  - fase-2
  - datos-maestros
---

# EmailSendService Fase 2 (datos maestros) — Documentación para QA

> Cubre la **Fase 2** del servicio de correo institucional (EmailSendService): plantillas, sincronización de contactos con el Core, contactos locales y grupos, en la API (`/api/v1`) y en el panel de administración. Se prueba en el entorno de **testing** (red local, Dokploy) y va dirigida al equipo de QA.
> Esta nota orienta a QA sobre **qué hace** la app y **por dónde empezar**; no define cómo ejecutar las pruebas.

## Referencias

| Recurso | Enlace |
|---------|--------|
| Plan de desarrollo (bóveda local) | [[email-service-plan]] |
| Sub-planes de la fase (bóveda local) | [[email-service-plantillas-sub-plan]] · [[email-service-sync-core-sub-plan]] · [[email-service-contactos-sub-plan]] · [[email-service-grupos-sub-plan]] |
| Plan / sub-planes en Nextcloud | `2. Proyectos/Servicio de correo/email-service-plan.md` (y `email-service-*-sub-plan.md` en la misma carpeta) |
| Repositorio git | http://187.174.80.181:3001/sistemas-tsj/notifications-service |
| Wiki del repositorio | http://187.174.80.181:3001/sistemas-tsj/notifications-service/wiki — páginas *API*, *Seguridad y permisos*, *Reglas de negocio*, *Panel* y *Despliegue y configuración* |
| Historial de versiones | `docs/versions/version-history.md` en el repositorio (rama `staging`) |
| Tareas en Plane | — |

## Objetivo

EmailSendService es la API que enviará correos 1 a 1 y a grupos en nombre de las aplicaciones institucionales (consumidores), con un panel para administrarlo. La Fase 2 deja listos los **datos maestros** que el envío necesita: las **plantillas** de correo (con variables y vista previa), la **base de contactos** (copiada del Core por poblaciones y completada con contactos locales) y los **grupos** de contactos con sus **asignaciones** a consumidores. El envío real (Fase 3) todavía no existe: en esta fase se prueba que los datos se crean, se consultan, se modifican, se dan de baja y se restauran con las reglas de acceso correctas, tanto por API como desde el panel.

## Funcionamiento

### Actores

Todos entran con la cabecera `X-Api-Key`; la key dice quién es y qué roles tiene:

| Identidad | Quién es | Qué puede en esta fase |
|---|---|---|
| **Admin** | La key de administración del servicio (una sola) | Todo: gestiona consumidores, managers y keys; plantillas; contactos; sincronización; todos los grupos; y decide qué plantillas y grupos puede usar cada consumidor |
| **Manager** con `TemplateManager` | Persona del equipo | Plantillas (crear, editar, activar/desactivar, vista previa) y la pantalla `/plantillas` |
| **Manager** con `GroupManager` | Persona del equipo | Contactos (lista, alta, edición, baja, restauración, exportación), sincronización con el Core y **sus** grupos; pantallas `/contactos` y `/grupos` |
| **Consumer** | Una aplicación (backend) | En esta fase, nada: no entra al panel y las rutas de gestión le responden `401` |

Un manager puede tener los dos roles. Una identidad que llama a una ruta de Admin recibe el mismo `401` que una key inválida; una que tiene key válida pero no el rol de la ruta recibe `403`.

### Flujo principal

1. El Admin crea en el panel (o por API) a los managers con sus roles y les entrega su key (se muestra una sola vez).
2. Un `TemplateManager` crea plantillas en el editor visual del panel: nombre, asunto, variables `{{clave}}`, cuerpo; puede ver una vista previa con valores de ejemplo y activar o desactivar la plantilla.
3. Un `GroupManager` abre Contactos, elige una población (estudiantes, personal, administrativos, aspirantes), pulsa "Obtener" y revisa lo que el Core trae de nuevo, con diferencias o dado de baja; aplica solo lo que marca. También da de alta contactos locales, corrige nombre o correo de cualquiera y exporta a Excel.
4. Ese `GroupManager` crea grupos y les agrega miembros: contactos sueltos, una población completa (desmarcando a quien no quiera) o los miembros de otro grupo suyo; puede quitar miembros, dar de baja y restaurar el grupo. Solo ve sus grupos; el Admin ve y gestiona todos.
5. El Admin asigna a cada consumidor las plantillas que podrá usar (para envíos 1 a 1 o a grupos) y los grupos a los que podrá enviar. Esas asignaciones las consumirá la Fase 3.

### Reglas de negocio relevantes

- **Plantillas:** nombre único (`409` si se repite); asunto y cuerpo usan marcadores `{{clave}}` y cada marcador debe estar declarado como variable (si no, `400`); la clave empieza en minúscula, admite letras, números y `_`, hasta 30 caracteres. No se borran: se desactivan, y una inactiva no se ofrece para enviar. El HTML se sanea en el servidor (se quitan scripts, eventos y URLs que no sean `http`, `https` o `mailto`). Límites: nombre 50, asunto 100, cuerpo 200 000 caracteres.
- **Sincronización:** por población; la revisión no escribe nada y la aplicación solo cambia lo autorizado. Nunca toca la suscripción del contacto. Si el Core devuelve 0 filas al aplicar, se rechaza (`409 CoreEmpty`) para no dar de baja a todos. Límite de 3 peticiones por 5 minutos por IP y endpoint (`429`).
- **Contactos locales:** su correo no puede repetir el de ningún contacto, de cualquier población, ni de uno dado de baja, sin distinguir mayúsculas (`409`). Solo los locales se dan de baja y se restauran desde aquí; los del Core se dan de baja desde la sincronización (`409` si se intenta por REST). Editar nombre o correo vale para cualquier contacto, también del Core; la siguiente revisión lo mostrará como "con diferencias".
- **Grupos:** nombre único entre los vigentes, sin distinguir mayúsculas y aunque sean de dueños distintos (`409`). El manager que lo crea es su dueño; otro manager recibe `403 ResourceNotGranted`. El Admin gestiona cualquiera, crea grupos sin dueño y es el único que fija `ownerId`. La baja es lógica, conserva miembros y asignaciones y libera el nombre; restaurar con el nombre ya tomado da `409`.
- **Miembros:** solo entran contactos vigentes y suscritos; los ya miembros se ignoran (`skipped`). Un contacto dado de baja (por REST o por la sincronización) sale solo de todos los grupos y restaurarlo no lo devuelve.
- **Asignaciones a consumidores:** el `PUT` reemplaza el conjunto completo; solo admite plantillas existentes y activas, y grupos existentes y vigentes (`400` si no).
- **Panel:** sesión de 8 horas en cookie; la key nunca llega al navegador. Un manager solo ve las pantallas de su rol. Los borradores de plantilla se guardan en el navegador al perder el foco y se ofrecen al volver ("Recuperar borrador").

## Entornos y acceso

| Entorno | URL base | Autenticación | Notas |
|---------|----------|---------------|-------|
| Testing — API | `http://192.168.14.131:5037/api/v1` | Cabecera `X-Api-Key` | Solo desde la red institucional. `GET /health/ready` debe responder `200`. |
| Testing — Panel | `http://192.168.14.131:5038` | Login con la misma key | Entra el Admin (todo) o un manager con rol (sus pantallas). |
| Testing — GraphQL | `POST http://192.168.14.131:5037/api/v1/graphql` | Cabecera `X-Api-Key` | Solo consultas; sin introspección en testing. |

Cómo obtener acceso: pedir al responsable del proyecto (Efraín García) una key de **manager** con los roles que se vayan a probar (`TemplateManager`, `GroupManager` o ambos) o una key de **consumer** para comprobar que no entra. La key de Admin la administra solo el responsable; si QA necesita probar las funciones de Admin (consumidores, managers, asignaciones), se acuerda con él. Las keys se muestran una sola vez al crearlas y rotarlas; la anterior deja de servir en el acto.

Versión desplegada: la fila más reciente de `docs/versions/version-history.md` en `staging` (Fase 2 desde la `0.0.0.04`). Cada entrega a testing la hace el pipeline del repositorio al hacer merge en `staging`; el esquema de la base lo aplica el responsable.

## Endpoints

### Índice

Rutas de la Fase 2 (las de consumidores, managers y keys son de la Fase 1 y están en la wiki *API*). Todas cuelgan de `/api/v1` y exigen `X-Api-Key`; "Manager" significa manager con el rol indicado, y el Admin siempre pasa.

| # | Método | Ruta | Autorización | Respuestas |
|---|--------|------|--------------|------------|
| 0 | `GET` | `/whoami` | Cualquier key válida | 200, 401 |
| 1 | `POST` | `/templates` | `TemplateManager` | 201, 400, 401, 403, 409 |
| 2 | `GET` | `/templates/{id}` | `TemplateManager` | 200, 401, 403, 404 |
| 3 | `PUT` | `/templates/{id}` | `TemplateManager` | 204, 400, 401, 403, 404, 409 |
| 4 | `PATCH` | `/templates/{id}` | `TemplateManager` | 204, 400, 401, 403, 404 |
| 5 | `POST` | `/templates/preview` | `TemplateManager` | 200, 400, 401, 403 |
| 6 | `GET` / `PUT` | `/consumers/{id}/templates` | Admin | 200 / 204, 400, 401, 404 |
| 7 | `POST` | `/sync/{kind}` | `GroupManager` | 200, 400, 401, 403, 404, 429, 502, 503 |
| 8 | `POST` | `/sync/{kind}/apply` | `GroupManager` | 200, 400, 401, 403, 404, 409, 429, 502, 503 |
| 9 | `POST` | `/contacts` | `GroupManager` | 201, 400, 401, 403, 409 |
| 10 | `GET` | `/contacts/{id}` | `GroupManager` | 200, 401, 403, 404 |
| 11 | `PUT` | `/contacts/{id}` | `GroupManager` | 204, 400, 401, 403, 404, 409 |
| 12 | `DELETE` | `/contacts/{id}` | `GroupManager` | 204, 401, 403, 404, 409 |
| 13 | `POST` | `/contacts/{id}/restore` | `GroupManager` | 204, 401, 403, 404, 409 |
| 14 | `GET` | `/contacts/export` | `GroupManager` | 200, 400, 401, 403 |
| 15 | `POST` | `/groups` | `GroupManager` | 201, 400, 401, 403, 409 |
| 16 | `GET` | `/groups/{id}` | Dueño o Admin | 200, 401, 403, 404 |
| 17 | `PUT` | `/groups/{id}` | Dueño o Admin | 204, 400, 401, 403, 404, 409 |
| 18 | `DELETE` | `/groups/{id}` | Dueño o Admin | 204, 401, 403, 404 |
| 19 | `POST` | `/groups/{id}/restore` | Dueño o Admin | 204, 401, 403, 404, 409 |
| 20 | `POST` | `/groups/{id}/members` | Dueño o Admin | 200, 400, 401, 403, 404, 409 |
| 21 | `POST` | `/groups/{id}/members/remove` | Dueño o Admin | 204, 400, 401, 403, 404 |
| 22 | `GET` / `PUT` | `/consumers/{id}/groups` | Admin | 200 / 204, 400, 401, 404 |
| 23 | `POST` | `/graphql` | Admin y managers (cada consulta exige su rol) | 200 (errores dentro de `errors`), 401 |
| 24 | `GET` | `/health/live` · `/health/ready` | Público | 200, 503 |

Las listas paginadas (`templates`, `contacts`, `groups`, `groupMembers`) **no son REST**: van por GraphQL (#23). Las rutas de contactos y grupos llevan el rate limit *Moderate* (60 peticiones por minuto por IP; `429` con `Retry-After`).

### 0. Identidad de la key

`GET /whoami`

- **Objetivo:** saber quién es la key con la que se llama y qué roles tiene. Es el primer paso para comprobar una key.
- **Autorización:** cualquier key válida.

**Response `200`**

```json
{ "id": "7c2e3b1a-…", "name": "Ana Pérez", "kind": "Manager", "roles": ["GroupManager", "TemplateManager"] }
```

El Admin responde `id: null`, `name: "admin"` y `Admin` más todos los roles. **Errores:** `401` key inválida, vencida, revocada o de un dueño inactivo.

### 1. Crear plantilla

`POST /templates`

- **Objetivo:** da de alta una plantilla de correo con sus variables.
- **Autorización:** `TemplateManager` (o Admin).
- **Reglas:** nombre único (hasta 50), asunto hasta 100, cuerpo hasta 200 000; todo marcador `{{clave}}` del asunto o del cuerpo debe estar en `variables`; claves sin repetir; `contentJson` debe ser un objeto (es el documento del editor del panel, se guarda para volver a editar). El HTML se sanea al guardar.

**Request**

```json
{
  "name": "bienvenida",
  "subject": "Hola {{nombre}}",
  "contentJson": { "type": "doc", "content": [] },
  "bodyHtml": "<p>Hola {{nombre}}</p>",
  "variables": [{ "key": "nombre", "required": true }],
  "transactional": false
}
```

**Response `201`**

```json
{ "id": "a1f2c3d4-…" }
```

**Errores:** `400` validación por campo o marcador sin variable (el motivo va en `detail`) · `403` sin el rol · `409` ya existe una plantilla con ese nombre.

### 2. Detalle de plantilla

`GET /templates/{id}`

- **Objetivo:** devuelve la plantilla completa para editarla.
- **Autorización:** `TemplateManager`.

**Response `200`**

```json
{
  "id": "a1f2c3d4-…",
  "name": "bienvenida",
  "subject": "Hola {{nombre}}",
  "contentJson": { "type": "doc", "content": [] },
  "bodyHtml": "<p>Hola {{nombre}}</p>",
  "variables": [{ "key": "nombre", "required": true }],
  "transactional": false,
  "active": true,
  "createdAt": "2026-11-03T18:00:00Z"
}
```

**Errores:** `404` no existe.

### 3. Reemplazar plantilla

`PUT /templates/{id}`

- **Objetivo:** reemplaza nombre, asunto, contenido, variables y el indicador transaccional.
- **Autorización:** `TemplateManager`.
- **Reglas:** las mismas de la alta; el nombre sigue siendo único.

**Request:** el mismo cuerpo que en #1. **Response `204`** sin cuerpo. **Errores:** `400`, `404`, `409` nombre repetido.

### 4. Activar o desactivar plantilla

`PATCH /templates/{id}`

- **Objetivo:** una plantilla inactiva no se ofrece para enviar ni se puede asignar. No hay borrado.
- **Autorización:** `TemplateManager`.

**Request**

```json
{ "active": false }
```

**Response `204`**. **Errores:** `400` sin `active` · `404`.

### 5. Vista previa

`POST /templates/preview`

- **Objetivo:** renderiza un borrador (no hace falta que esté guardado) con valores de ejemplo.
- **Autorización:** `TemplateManager`.
- **Reglas:** los valores se codifican en el HTML; el asunto pierde los saltos de línea; `missing` lista las variables obligatorias sin valor.

**Request**

```json
{
  "subject": "Hola {{nombre}}",
  "bodyHtml": "<p>Hola {{nombre}}, tu folio es {{folio}}</p>",
  "variables": [{ "key": "nombre", "required": true }, { "key": "folio", "required": true }],
  "values": { "nombre": "Ana" }
}
```

**Response `200`**

```json
{ "subject": "Hola Ana", "html": "<p>Hola Ana, tu folio es </p>", "missing": ["folio"] }
```

**Errores:** `400` marcador sin variable o claves inválidas.

### 6. Plantillas asignadas a un consumidor

`GET /consumers/{id}/templates` · `PUT /consumers/{id}/templates`

- **Objetivo:** qué plantillas podrá usar el consumidor al enviar: `message` para envíos 1 a 1 y `group` para envíos a grupos.
- **Autorización:** Admin.
- **Reglas:** el `PUT` reemplaza el conjunto completo; solo admite plantillas existentes y activas.

**Response `200` del GET / Request del PUT**

```json
{ "message": ["a1f2c3d4-…"], "group": ["b5e6f7a8-…"] }
```

**Response `204`** en el PUT. **Errores:** `400` plantilla inexistente o inactiva, ids repetidos · `404` el consumidor no existe · `401` para cualquier key que no sea Admin.

### 7. Revisión de una población del Core

`POST /sync/{kind}?limit=N`

- **Objetivo:** compara la población del Core con los contactos locales y reporta qué hay de nuevo, qué cambió, qué ya no existe y qué se omitió. **No escribe.**
- **Autorización:** `GroupManager`.
- **Reglas:** `{kind}` es `students`, `personal`, `admins` o `applicants`; `limit` (1 a 5000, 500 por defecto) recorta los `items` de cada lista, `total` siempre es el real. Sin cuerpo. Lee toda la población (unas 88 000 filas en estudiantes). Rate limit *Sync*: 3 peticiones por 5 minutos por IP y endpoint.

**Response `200`**

```json
{
  "kind": "STUDENT",
  "read": 88012,
  "skipped": { "total": 1, "items": [{ "coreId": 41, "email": "", "displayName": "Ana Pérez", "reason": "EMPTY_EMAIL" }] },
  "added": { "total": 310, "items": [{ "coreId": 90001, "email": "luis@example.com", "displayName": "Luis Gómez" }] },
  "changed": {
    "total": 12,
    "items": [{
      "id": "0b6f6f0e-…", "coreId": 77,
      "before": { "email": "a@example.com", "displayName": "Ana", "deleted": false },
      "after": { "email": "ana@example.com", "displayName": "Ana Ruiz" }
    }]
  },
  "removed": { "total": 3, "items": [{ "id": "5d1e…", "coreId": 12, "email": "x@example.com", "displayName": "X" }] }
}
```

`added`: filas del Core sin contacto local. `changed`: correo o nombre distintos, o un contacto dado de baja que reaparece (`before.deleted: true`). `removed`: contactos locales vivos que el Core ya no devuelve. `skipped`: filas que no se pueden copiar (`EMPTY_EMAIL`, `INVALID_EMAIL`, `EMAIL_TOO_LONG`, `DUPLICATE_CORE_ID`); una fila omitida no da de baja al contacto.

**Errores:** `400` `limit` fuera de rango · `403` sin el rol · `404` población desconocida · `429` límite · `502` el Core no responde · `503` el Core no está configurado.

### 8. Aplicar la sincronización

`POST /sync/{kind}/apply`

- **Objetivo:** vuelve a leer el Core y aplica solo lo autorizado en cada sección (inserta, actualiza, da de baja).
- **Autorización:** `GroupManager`.
- **Reglas:** cada sección admite `all: true` o `ids` (los `coreId` de la revisión); al menos una sección. Nunca toca la suscripción; al actualizar limpia la baja; la baja del Core es lógica. Mismo rate limit que #7.

**Request**

```json
{ "added": { "ids": [90001, 90002] }, "changed": { "all": true }, "removed": { "ids": [12] } }
```

**Response `200`**

```json
{ "inserted": 2, "updated": 12, "deleted": 1 }
```

**Errores:** `400` cuerpo sin nada que aplicar · `404` población desconocida · `409` `CoreEmpty` (el Core devolvió 0 filas) · `429`, `502`, `503` como en #7.

### 9. Alta de contacto local

`POST /contacts`

- **Objetivo:** crea un contacto `kind = LOCAL`, `source = LOCAL`, suscrito y sin `coreId`.
- **Autorización:** `GroupManager`.
- **Reglas:** correo válido (hasta 100) y único entre **todos** los contactos, incluidos los dados de baja, sin distinguir mayúsculas; `displayName` opcional (hasta 150).

**Request**

```json
{ "email": "ana@example.com", "displayName": "Ana Pérez" }
```

**Response `201`**

```json
{ "id": "0b6f6f0e-…" }
```

**Errores:** `400` validación por campo · `409` "Ya existe un contacto con ese correo".

### 10. Detalle de contacto

`GET /contacts/{id}`

- **Autorización:** `GroupManager`.

**Response `200`**

```json
{
  "id": "0b6f6f0e-…",
  "coreId": null,
  "kind": "LOCAL",
  "source": "LOCAL",
  "email": "ana@example.com",
  "displayName": "Ana Pérez",
  "subscribed": true,
  "unsubscribedAt": null,
  "unsubscribedReason": null,
  "syncedAt": "2026-10-30T20:15:00Z",
  "deletedAt": null
}
```

**Errores:** `404` no existe.

### 11. Editar contacto

`PUT /contacts/{id}`

- **Objetivo:** corrige correo y nombre de **cualquier** contacto, también del Core (corrección local).
- **Autorización:** `GroupManager`.
- **Reglas:** mismo cuerpo y validaciones que #9; no toca población, origen, `coreId` ni suscripción. En uno del Core, la siguiente revisión lo mostrará como "con diferencias".

**Request:** `{ "email": "ana.perez@example.com", "displayName": "Ana Pérez Ruiz" }`. **Response `204`**. **Errores:** `400`, `404`, `409` correo repetido.

### 12. Baja de contacto local

`DELETE /contacts/{id}`

- **Objetivo:** baja lógica, idempotente; el contacto sale automáticamente de todos los grupos.
- **Autorización:** `GroupManager`.
- **Reglas:** solo contactos `LOCAL`.

**Response `204`**. **Errores:** `404` · `409` es un contacto del Core ("La baja de un contacto del Core se hace desde la sincronización").

### 13. Restaurar contacto local

`POST /contacts/{id}/restore`

- **Objetivo:** deshace la baja; **no** lo devuelve a los grupos de los que salió.
- **Autorización:** `GroupManager`.

**Response `204`**. **Errores:** `404` · `409` es un contacto del Core.

### 14. Exportar contactos

`GET /contacts/export?kind=LOCAL&deleted=false&orderBy=NAME&direction=ASC`

- **Objetivo:** devuelve **todas** las filas del filtro como un arreglo JSON (sin paginar). El panel lo convierte en `.xlsx`.
- **Autorización:** `GroupManager`.
- **Reglas:** query opcional `kind` (`STUDENT`, `PERSONAL`, `ADMIN`, `APPLICANT`, `LOCAL`), `search`, `deleted`, `subscribed`, `orderBy` (`NAME`, `EMAIL`, `SYNCED_AT`), `direction` (`ASC`, `DESC`).

**Response `200`:** arreglo de objetos con la forma del detalle (#10). **Errores:** `400` `kind`, `orderBy` o `direction` inválidos.

### 15. Crear grupo

`POST /groups`

- **Objetivo:** da de alta un grupo; el manager que lo crea queda como dueño.
- **Autorización:** `GroupManager`.
- **Reglas:** `name` 1 a 50, único entre los vigentes sin distinguir mayúsculas; `description` hasta 255; `ownerId` solo lo envía el Admin (un manager que lo envíe recibe `400`); el Admin sin `ownerId` crea un grupo sin dueño.

**Request**

```json
{ "name": "Becarios 2026", "description": "Alumnos con beca vigente", "ownerId": "7c2e3b1a-…" }
```

**Response `201`**

```json
{ "id": "3f9d1c2a-…" }
```

**Errores:** `400` validación, `ownerId` enviado por un manager o de un manager inexistente · `409` "Ya existe un grupo con ese nombre".

### 16. Detalle de grupo

`GET /groups/{id}`

- **Autorización:** dueño o Admin.

**Response `200`**

```json
{
  "id": "3f9d1c2a-…",
  "name": "Becarios 2026",
  "description": "Alumnos con beca vigente",
  "ownerId": "7c2e3b1a-…",
  "ownerName": "Ana Pérez",
  "membersCount": 120,
  "createdAt": "2026-11-02T18:40:00Z",
  "deletedAt": null
}
```

`ownerId` y `ownerName` son `null` en los grupos sin dueño (del Admin). **Errores:** `403` `ResourceNotGranted` (grupo de otro manager) · `404`.

### 17. Editar grupo

`PUT /groups/{id}`

- **Objetivo:** reemplaza nombre y descripción; el Admin además fija el dueño (`ownerId` ausente = sin dueño), un manager conserva el suyo.
- **Autorización:** dueño o Admin.

**Request:** el mismo cuerpo que #15. **Response `204`**. **Errores:** `400`, `403`, `404`, `409` nombre repetido.

### 18. Baja de grupo

`DELETE /groups/{id}`

- **Objetivo:** baja lógica e idempotente; conserva miembros y asignaciones y libera el nombre.
- **Autorización:** dueño o Admin.

**Response `204`**. **Errores:** `403`, `404`.

### 19. Restaurar grupo

`POST /groups/{id}/restore`

- **Autorización:** dueño o Admin.
- **Reglas:** si otro grupo vigente ya usa el nombre, `409`; se puede editar el grupo dado de baja para renombrarlo antes.

**Response `204`**. **Errores:** `403`, `404`, `409` "Ya existe un grupo vigente con ese nombre; cámbialo antes de restaurar".

### 20. Agregar miembros

`POST /groups/{id}/members`

- **Objetivo:** añade contactos al grupo desde uno o varios orígenes combinables.
- **Autorización:** dueño o Admin.
- **Reglas:** `contactIds` (ids concretos), `kind` (toda una población: `STUDENT`, `PERSONAL`, `ADMIN`, `APPLICANT`, `LOCAL`) y `fromGroupId` (miembros de otro grupo que el actor pueda gestionar; otro ajeno `403`, el mismo grupo `400`). `excludeContactIds` descarta contactos de los orígenes `kind` y `fromGroupId` (los de `contactIds` siempre cuentan). Al menos un origen. Solo entran contactos vigentes y suscritos; los ya miembros se ignoran. Sin límite de miembros.

**Request**

```json
{
  "contactIds": ["0b6f6f0e-…"],
  "kind": "STUDENT",
  "fromGroupId": "9a1b…",
  "excludeContactIds": ["5d1e…"]
}
```

**Response `200`**

```json
{ "added": 118, "skipped": 4 }
```

**Errores:** `400` sin origen ("Indica al menos un origen: contactIds, kind o fromGroupId."), `kind` inválido, `fromGroupId` igual al grupo · `403` · `404` · `409` "El grupo está dado de baja".

### 21. Quitar miembros

`POST /groups/{id}/members/remove`

- **Autorización:** dueño o Admin.
- **Reglas:** `contactIds` con al menos un id; los que no son miembros se ignoran.

**Request**

```json
{ "contactIds": ["0b6f6f0e-…"] }
```

**Response `204`**. **Errores:** `400` lista vacía · `403` · `404`.

### 22. Grupos asignados a un consumidor

`GET /consumers/{id}/groups` · `PUT /consumers/{id}/groups`

- **Objetivo:** a qué grupos podrá enviar el consumidor (rol `GroupSender`) en la Fase 3.
- **Autorización:** Admin.
- **Reglas:** el `PUT` reemplaza el conjunto; solo grupos existentes y vigentes; `groups` vacío o ausente lo deja sin grupos.

**Response `200` del GET / Request del PUT**

```json
{ "groups": ["3f9d1c2a-…"] }
```

**Response `204`** en el PUT. **Errores:** `400` ids repetidos, grupo inexistente o dado de baja · `404` el consumidor no existe · `401` para cualquier key que no sea Admin.

### 23. Consultas GraphQL (listas paginadas)

`POST /graphql`

- **Objetivo:** las listas con búsqueda, filtro, orden y paginación. Solo consultas.
- **Autorización:** Admin y managers; cada consulta exige su rol: `templates` → `TemplateManager`; `contacts`, `groups`, `groupMembers` → `GroupManager`; `consumers` y `managers` → Admin. Un manager en `groups` solo obtiene **sus** grupos y pedir `groupMembers` de un grupo ajeno responde `ResourceNotGranted` dentro de `errors`.
- **Reglas:** `page` desde 1, `pageSize` 1 a 100 (20 por defecto); la respuesta es `{ items, page, pageSize, total }`. `filter.search` hasta 100 caracteres. Sin `orderBy`, los más recientes primero.

**Request** (cuerpo `{ "query": "…", "variables": { … } }`)

```graphql
query {
  templates(page: 1, pageSize: 20, filter: { search: "bienvenida", active: true }, orderBy: { field: NAME, direction: ASC }) {
    items { id name subject transactional active createdAt }
    page pageSize total
  }
  contacts(page: 1, pageSize: 20, filter: { kind: STUDENT, search: "ana", deleted: false, subscribed: true },
           orderBy: { field: NAME, direction: ASC }) {
    items { id coreId kind source email displayName subscribed unsubscribedAt syncedAt deletedAt }
    page pageSize total
  }
  groups(page: 1, pageSize: 20, filter: { search: "becarios", deleted: false }, orderBy: { field: NAME, direction: ASC }) {
    items { id name description ownerId ownerName membersCount createdAt deletedAt }
    page pageSize total
  }
  groupMembers(groupId: "3f9d1c2a-…", page: 1, pageSize: 20, filter: { kind: STUDENT, search: "ana" },
               orderBy: { field: NAME, direction: ASC }) {
    items { id coreId kind source email displayName subscribed syncedAt deletedAt }
    total
  }
}
```

**Response `200`**

```json
{ "data": { "groups": { "items": [ { "id": "3f9d1c2a-…", "name": "Becarios 2026", "membersCount": 120 } ], "page": 1, "pageSize": 20, "total": 1 } } }
```

Filtros: `contacts.filter.kind` (`STUDENT`, `PERSONAL`, `ADMIN`, `APPLICANT`, `LOCAL`), `deleted` y `subscribed` booleanos, `orderBy.field` `NAME`, `EMAIL`, `SYNCED_AT`; `groups.filter.deleted` separa vigentes (`false`) y dados de baja (`true`), `orderBy.field` `NAME`, `CREATED_AT`; `groupMembers` usa el filtro y el orden de contactos. **Errores:** `401` sin key; sin el rol de la consulta, el error va en `errors` con el `data` en `null`.

### 24. Salud

`GET /health/live` · `GET /health/ready`

- **Objetivo:** `live` dice si el proceso responde; `ready` comprueba Postgres, RabbitMQ y la base del Core (`Degraded` si el Core no está configurado).
- **Autorización:** público.

**Response `200`**

```json
{ "status": "Healthy", "checks": [ { "name": "postgres", "status": "Healthy", "ms": 7 }, { "name": "rabbitmq", "status": "Healthy", "ms": 0 }, { "name": "coredb", "status": "Healthy", "ms": 2 } ] }
```

### Panel (pantallas de la Fase 2)

| Pantalla | Quién | Qué hace |
|---|---|---|
| `/` | Todos | Login con la key; una key de consumer o de manager sin pantallas recibe "sin acceso al panel" |
| `/plantillas`, `/plantillas/nueva`, `/plantillas/[id]` | Admin, `TemplateManager` | Lista con búsqueda, orden y paginación; activar/desactivar con confirmación; editor visual (nombre, asunto, transaccional, variables, cuerpo), secciones institucionales, ayuda, vista previa en escritorio y móvil, borradores en el navegador |
| Consumidores → botón "Plantillas" | Admin | Asignación de plantillas al consumidor (listas para 1 a 1 y para grupos) |
| `/contactos` | Admin, `GroupManager` | Población, búsqueda, estado; "Nuevo contacto", "Editar" (todos), "Dar de baja" y "Restaurar" (solo locales); "Exportar" descarga `.xlsx`; "Obtener <población>" abre la revisión del Core con resumen, filtros, selección por sección y "Aplicar cambios" |
| `/grupos`, `/grupos/[id]` | Admin, `GroupManager` | Lista (Vigentes, Todos, Dados de baja), "Nuevo grupo", "Editar" (el selector "Dueño" solo lo ve el Admin), "Dar de baja" y "Restaurar"; miembros con búsqueda, población, selección entre páginas y "Quitar seleccionados"; "Agregar miembros" con Población completa (todos marcados, se desmarcan), De otro grupo y Buscar contactos |
| Consumidores → botón "Grupos" | Admin | Asignación de grupos al consumidor |
| `/salud` | Admin | Estado de la API y sus dependencias |

## Errores y códigos comunes

Las respuestas de error son `ProblemDetails` con `code` y `traceId` (útil para reportar: con el `traceId` desarrollo localiza el log). En `400` de validación viene `errors` por campo.

| Código | Cuándo |
|--------|--------|
| `400` | Entrada inválida (`code: Validation`, `errors` por campo) o regla de entrada (marcador sin variable, sin origen de miembros, `ownerId` de un manager, asignación con ids inválidos) |
| `401` | Sin key, key inválida, vencida, revocada o de dueño inactivo; también cualquier key que no sea Admin en rutas de Admin |
| `403` | Key válida sin el rol de la ruta (`code: RoleNotAllowed`) o manager sobre un grupo ajeno (`code: ResourceNotGranted`) |
| `404` | El recurso o la población no existe |
| `409` | Duplicado o estado que lo impide: nombre de plantilla o de grupo repetido, correo repetido, baja de un contacto del Core por REST, restaurar grupo con nombre tomado, miembros en grupo dado de baja, `CoreEmpty` al aplicar la sincronización |
| `429` | Rate limit (`code: TooManyRequests`, cabecera `Retry-After`): *Sync* 3 por 5 minutos; *Moderate* 60 por minuto |
| `502` / `503` | El Core no responde / no está configurado (sincronización); `503` también en `health/ready` si una dependencia falla |
| `500` | Error interno: mensaje genérico, reportar con el `traceId` |

En el panel, un `401` cierra la sesión y vuelve al login con el aviso "sesión expirada"; un `409` se muestra en el campo del formulario; con la API caída se ve una página 503.

## Nota de pruebas

> [!note] Estado de pruebas por parte de desarrollo
> - **¿Se realizaron pruebas?** Sí, manuales por desarrollo.
> - **Alcance:** en local, matriz con curl de cada ruta de la fase (sin key, key inválida, caso feliz, 400/403/404/409, roles y dueño de grupo) y flujos del panel como Admin y como manager (plantillas con editor y vista previa, revisión y aplicación de la sincronización contra el Core real, contactos locales y exportación, grupos con los tres orígenes de miembros y asignaciones). El slice de sincronización (4) además tuvo E2E en testing el 2026-10-30.
> - **Qué no se probó:** pruebas unitarias e integración automatizadas (pendientes, las escriben el responsable y el equipo); rendimiento (k6); E2E en testing de plantillas, contactos y grupos (se hace con esta entrega); concurrencia de varios managers sobre el mismo grupo; navegadores distintos de Chrome.
> - **Fecha / versión:** 2026-11-03 / testing `0.0.0.04` (Fase 2 completa; las entregas siguientes solo corrigen).

## Relacionado

- [[email-service-plan]]
- [[email-service-plantillas-sub-plan]] · [[email-service-sync-core-sub-plan]] · [[email-service-contactos-sub-plan]] · [[email-service-grupos-sub-plan]]
- [[email-service-cicd-sub-plan]] (cómo llega cada versión a testing)
