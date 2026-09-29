---
tipo: manual
titulo: "Guía de integración frontend — API tsj-core"
autor: "Efraín García"
creado: 2026-09-29
actualizado: 2026-09-29
estado: vigente
proyecto: tsj-web
tags:
  - tsj-web
  - api
  - frontend
  - manual
  - aspirantes
---

# Guía de integración frontend — API `tsj-core`

> Contratos vigentes y guía de pruebas para la aplicación frontend del portal TSJ/TecMM. No usar ejemplos con datos personales reales.

## Resumen

- Base local: `http://localhost/tsj-api`.
- Base productiva: `https://tecmm.mx/tsj-api`.
- El contenido usa JSON con propiedades `camelCase`; enviar `Content-Type: application/json` cuando exista cuerpo.
- Errores de dominio usan `application/problem+json`.
- Esta guía cubre las rutas HTTP expuestas actualmente. `CustomizeUa` aún no tiene controlador, por lo que no es integrable como API.

## Requisitos previos

- Para probar localmente: infraestructura levantada y `baseUrl` configurada a `http://localhost/tsj-api`.
- Conservar la cookie pública y, cuando corresponda, el JWT retornado por login.
- Usar datos sintéticos en desarrollo; no guardar CURPs, contraseñas, tokens o JWTs en repositorios, capturas o tickets.

## CORS y autenticación

### CORS

La API acepta solicitudes de cualquier dominio y permite credenciales. El backend devuelve el `Origin` recibido en `Access-Control-Allow-Origin` y `Access-Control-Allow-Credentials: true`; no usa `*`, porque los navegadores no permiten `*` con credenciales.

Para peticiones del mismo sitio que deban conservar la cookie pública, el cliente debe incluir credenciales:

```ts
fetch(`${baseUrl}/aspirantes/validar`, {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload)
});
```

La política CORS permite la llamada desde cualquier origen. Sin embargo, la cookie pública actual usa `SameSite=Lax`: un navegador puede no guardarla o reenviarla cuando frontend y API pertenecen a sitios distintos. Esto no afecta el uso de JWT Bearer; si se requiere cookie pública entre sitios, backend debe cambiar explícitamente su política `SameSite` y usar HTTPS.

> [!warning]
> Aceptar credenciales desde cualquier origen es una apertura amplia solicitada para el entorno actual. No usar esta configuración como supuesto de seguridad para operaciones sensibles; las rutas protegidas siguen requiriendo JWT y rol.

### Mecanismos

| Mecanismo | Uso | Cliente |
|---|---|---|
| Cookie `tesj-public-user-id` | Rutas con política `Public`; se crea automáticamente en la primera petición. | `credentials: "include"`; el navegador puede limitarla entre sitios por `SameSite=Lax`. |
| JWT Google | Gestión y `GET /auth/me`. | `Authorization: Bearer <jwt>` |
| JWT Aspirante | Cambio de procedimiento de su propio aspirante. | `Authorization: Bearer <jwt>` |

Los JWT expiran según configuración (actualmente 1 día). `POST /auth/logout` devuelve `204`, pero no invalida un JWT en servidor: frontend debe eliminarlo de su almacenamiento.

### Roles y alcance

| Rol | Hereda políticas de | Capacidades principales |
|---|---|---|
| `Public` | — | Catálogo público, aspirantes y cookie pública. |
| `Vinculador` | `Public` | Lectura sobre recursos propios; no tiene ruta específica expuesta aún. |
| `CampusManager` | `Public` | Gestión de unidades; escritura/edición solo en su unidad asignada. |
| `Director` | `CampusManager`, `Public` | Gestión global de unidades y creación. |
| `Admin` | `Director`, `CampusManager`, `Public` | Gestión global y eliminación. |
| `Aspirante` | — | Editar únicamente su propio procedimiento. |

## Límites y errores

| Política | Límite actual |
|---|---|
| `get-public` | 10 solicitudes / 60 s |
| `auth-login` | 10 solicitudes / 300 s |
| `registro-aspirante` | 5 solicitudes / 300 s |
| `write` | Cubeta de 5 tokens; repone 1 cada 10 s |

Todos los errores de aplicación siguen esta forma:

```json
{
  "status": 422,
  "title": "Business Error",
  "detail": "Descripción legible del problema.",
  "instance": "/tsj-api/ruta"
}
```

Tratar `400` como JSON/método inválido, `401` como sesión ausente o JWT inválido, `403` como rol o alcance insuficiente, `404` como recurso inexistente, `409` como conflicto de estado, `422` como regla/validación incumplida, `429` como límite y `503` como dependencia temporalmente no disponible.

## Catálogo de endpoints

### Autenticación

| Método | Ruta | Autorización | Respuestas |
|---|---|---|---|
| `POST` | `/auth/google` | Anónimo, `auth-login` | 200, 400, 401, 429 |
| `GET` | `/auth/me` | JWT Google (`GoogleUser`) | 200, 401 |
| `POST` | `/auth/logout` | Anónimo | 204 |

`POST /auth/google`

```json
{ "code": "google-authorization-code", "redirectUri": "http://localhost:5173/callback", "codeVerifier": "pkce-verifier" }
```

`200 OK`

```json
{
  "jwt": "<jwt-google>",
  "user": {
    "id": 1,
    "email": "usuario@tecmm.edu.mx",
    "displayName": "Usuario de prueba",
    "avatarUrl": "https://example.test/avatar.png",
    "role": "CampusManager",
    "unidadAcademicaId": 3
  }
}
```

El cliente inicia OAuth con PKCE en Google, envía el `code`, `redirectUri` permitido y `codeVerifier`; backend intercambia y valida el ID token, dominio permitido y URI. `GET /auth/me` retorna el mismo objeto `user`.

### Unidades académicas

| Método | Ruta | Política / límite | Respuestas |
|---|---|---|---|
| `GET` | `/unidad-academica` | `Public` / `get-public` | 200, 401, 429 |
| `GET` | `/unidad-academica/{id}` | `Public` / `get-public` | 200, 401, 404, 429 |
| `GET` | `/unidad-academica/management` | `CampusManager` / `write` | 200, 401, 403, 429 |
| `GET` | `/unidad-academica/management/{id}` | `CampusManager` / `write` | 200, 401, 403, 404, 429 |
| `POST` | `/unidad-academica/management` | `Director` / `write` | 201, 401, 403, 422, 429 |
| `PUT` | `/unidad-academica/management/{id}` | `CampusManager` / `write` | 200, 401, 403, 404, 422, 429 |
| `PATCH` | `/unidad-academica/management/{id}` | `CampusManager` / `write` | 200, 401, 403, 404, 422, 429 |
| `POST` | `/unidad-academica/management/{id}/disable` | `CampusManager` / `write` | 200, 401, 403, 404, 429 |
| `POST` | `/unidad-academica/management/{id}/enable` | `CampusManager` / `write` | 200, 401, 403, 404, 429 |
| `DELETE` | `/unidad-academica/management/{id}` | `Admin` / `write` | 200, 401, 403, 404, 422, 429 |

Contrato de recurso y cuerpo completo de `POST`/`PUT`:

```json
{
  "id": 3,
  "name": "Unidad Académica de prueba",
  "coverPhoto": "https://example.test/cover.jpg",
  "iconPhoto": "https://example.test/icon.jpg",
  "address": "Calle de prueba 1",
  "phone": "7121234567",
  "email": "contacto@example.test",
  "whatsapp": "5217121234567",
  "disabled": false
}
```

En creación y reemplazo, enviar los siete campos editables (`name` a `whatsapp`); `id` y `disabled` son solo respuesta. `PATCH` acepta una combinación no vacía de esos campos: `null` equivale a no modificar y no puede limpiar un valor. El catálogo público omite unidades deshabilitadas; las rutas `management` las incluyen. `CampusManager` solo puede modificar su unidad; `Director` y `Admin` pueden cualquier unidad. Deshabilitar, habilitar y eliminar no llevan cuerpo.

### Aspirantes

| Método | Ruta | Política / límite | Respuestas |
|---|---|---|---|
| `GET` | `/aspirantes/verificar/{curp}` | `Public` / `get-public` | 200, 401, 429 |
| `POST` | `/aspirantes/validar` | `Public` / `registro-aspirante` | 200, 409, 422, 429, 503 |
| `POST` | `/aspirantes/registro` | `Public` / `registro-aspirante` | 201, 409, 422, 429, 503 |
| `POST` | `/aspirantes/login` | `Public` / `auth-login` | 200, 401, 422, 429 |
| `POST` | `/aspirantes/contrasena/recuperacion` | `Public` / `auth-login` | 202, 422, 429, 503 |
| `POST` | `/aspirantes/contrasena/restablecer` | `Public` / `auth-login` | 204, 422, 429, 503 |
| `PATCH` | `/aspirantes/procedimiento` | JWT `Aspirante` / `write` | 200, 400, 401, 404, 422, 429 |

#### Consulta por CURP

`GET /aspirantes/verificar/{curp}` devuelve:

```json
{
  "existe": true,
  "tipo": "aspirante",
  "aspirante": {
    "id": 21,
    "curp": "TEST000101HDFXXX09",
    "nombre": "NOMBRE",
    "primerApellido": "PATERNO",
    "segundoApellido": "MATERNO",
    "fechaNacimiento": "2000-01-01T00:00:00",
    "lugarNacimiento": null,
    "sexo": "H",
    "correo": "persona@example.test",
    "celular": null,
    "proceso": "Licenciaturas",
    "procedimiento": "Nuevo Ingreso",
    "status": "Activo",
    "programaId": null
  }
}
```

Cuando no exista, `existe` será `false` y los campos de tipo/aspirante pueden ser `null`.

#### Prevalidar y registrar

Prevalidación de ingreso ordinario:

```json
{ "curp": "TEST000101HDFXXX09", "proceso": "Licenciaturas", "procedimiento": "Nuevo Ingreso" }
```

Prevalidación de alumno de Licenciatura que solicita Posgrados:

```json
{ "curp": "TEST000101HDFXXX09", "proceso": "Posgrados", "procedimiento": null }
```

Respuesta permitida:

```json
{
  "puedeRegistrarse": true,
  "usuarioRegistrado": false,
  "mensaje": null,
  "tokenValidacion": "<token-opaco-de-un-solo-uso>",
  "expiraEnSegundos": 720,
  "datosCurp": {
    "estatus": "correcto",
    "curp": "TEST000101HDFXXX09",
    "apellidoPaterno": "PATERNO",
    "apellidoMaterno": "MATERNO",
    "nombre": "NOMBRE",
    "sexo": "H",
    "fechaNacimiento": "2000-01-01",
    "statusCurp": "RCN"
  }
}
```

Si ya tiene usuario, responde `200` con `usuarioRegistrado: true` y sin token. Si hay aspirante/alumno incompatible, procedimiento distinto o inconsistencia, responde `409` o `422` según el caso. El token expira en 12 minutos y se consume incluso cuando el registro posterior falla.

Registro:

```json
{
  "tokenValidacion": "<token-de-prevalidacion>",
  "curp": "TEST000101HDFXXX09",
  "nombre": "NOMBRE",
  "primerApellido": "PATERNO",
  "segundoApellido": "MATERNO",
  "fechaNacimiento": "2000-01-01",
  "lugarNacimiento": null,
  "sexo": "H",
  "correo": "persona@example.test",
  "celular": null,
  "contrasena": "<contrasena-inicial>",
  "proceso": "Licenciaturas",
  "procedimiento": "Nuevo Ingreso"
}
```

`curp`, `nombre`, `primerApellido`, `correo`, `contrasena`, `proceso` y el procedimiento efectivo son obligatorios. CURP se normaliza a mayúsculas; teléfono y correo deben tener formato válido. `procedimiento` puede omitirse únicamente para la ruta especial de alumno de Licenciatura hacia Posgrados; el token es la autoridad para proceso y procedimiento.

`201 Created` y `POST /aspirantes/login` devuelven:

```json
{ "aspirante": { "id": 21, "curp": "TEST000101HDFXXX09", "nombre": "NOMBRE", "primerApellido": "PATERNO", "segundoApellido": "MATERNO", "fechaNacimiento": "2000-01-01T00:00:00", "lugarNacimiento": null, "sexo": "H", "correo": "persona@example.test", "celular": null, "proceso": "Licenciaturas", "procedimiento": "Nuevo Ingreso", "status": "Activo", "programaId": null }, "jwt": "<jwt-aspirante>" }
```

Login recibe `{ "curp": "...", "contrasena": "..." }`. Guardar el JWT solo durante la sesión necesaria y enviarlo como Bearer.

#### Procedimiento y contraseña

Para cambiar procedimiento, usar el JWT de aspirante y enviar:

```json
{ "procedimiento": "Pase Directo" }
```

Solo admite `Nuevo Ingreso` o `Pase Directo` y solo modifica al aspirante representado por el JWT.

Recuperación solicita `{ "curp": "TEST000101HDFXXX09" }` y siempre intenta responder `202` sin revelar si existe la CURP. Restablecimiento recibe `{ "token": "<token-del-correo>", "contrasena": "<nueva-contrasena>" }` y responde `204`. Los tokens de recuperación son opacos, de un solo uso y su vigencia depende de la configuración del backend.

### Flujo de aspirante y bloqueo actual

1. Enviar `POST /aspirantes/validar`.
2. Si `puedeRegistrarse` es `true`, prellenar los datos disponibles de `datosCurp`, solicitar correo/contraseña y conservar el token solo en memoria.
3. Enviar `POST /aspirantes/registro` antes de la expiración.
4. Conservar el JWT de la respuesta para cambiar procedimiento; nunca usar el token de prevalidación como autenticación.

> [!danger] Bloqueo conocido — Licenciatura a Posgrados
> La prevalidación del alumno de Licenciatura con `proceso: "Posgrados"` y `procedimiento: null` responde correctamente con `puedeRegistrarse: true`. El registro actual vuelve a evaluar EDCORE con el procedimiento efectivo `Nuevo Ingreso` y responde `409` (`La CURP cambió de estado en EDCORE`). No se crea el aspirante. Frontend debe mostrar el error, no reintentar automáticamente y no considerar finalizado el registro hasta que backend corrija la revalidación.

## Uso / Pasos de prueba

1. Definir `baseUrl` y conservar cookies para pruebas públicas.
2. Probar catálogo público, luego login Google y una ruta de gestión con Bearer JWT.
3. Probar aspirantes con una CURP autorizada y datos sintéticos; no reutilizar un token de prevalidación ni de recuperación.
4. Para verificar CORS desde cualquier frontend, enviar un preflight `OPTIONS` con `Origin` arbitrario y `Access-Control-Request-Method`; debe responder con ese mismo origen y credenciales permitidas.

## Troubleshooting

| Síntoma | Causa probable | Acción frontend |
|---|---|---|
| La cookie pública no persiste entre sitios | La cookie actual usa `SameSite=Lax`. | Usar JWT o solicitar un ajuste específico de política de cookies con HTTPS. |
| `401` en gestión | JWT ausente, vencido o no Google. | Renovar sesión con Google. |
| `403` en unidad académica | Rol o unidad fuera de alcance. | Ocultar la acción y mostrar acceso insuficiente. |
| `422` al registrar | Campo inválido, token usado/vencido o regla de negocio. | Mostrar `detail`; volver a prevalidar si el token no existe. |
| `409` en Posgrados | Bloqueo conocido de revalidación. | No registrar ni reintentar automáticamente. |
| `429` | Límite de la política alcanzado. | Esperar la ventana indicada y evitar reintentos agresivos. |
| `503` | RENAPO, Redis, correo o EDCORE no disponible. | Informar indisponibilidad temporal y permitir reintento manual. |

## Relacionado

- [[backend-manual]]
