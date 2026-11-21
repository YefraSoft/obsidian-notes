---
tipo: manual
titulo: "Integración de clientes — Contrato de API"
autor: Efraín García
creado: 2026-09-18
actualizado: 2026-11-20
estado: vigente
proyecto: rh-cotla
tags:
  - rh-cotla
  - manual
  - api
  - contrato
  - clientes
---
# Integración de clientes — Contrato de API

Contrato de consumo del backend para el panel web y la app móvil. Describe lo que el
API ofrece hoy y lo que todavía no. No describe ninguna aplicación cliente.

- **Base por ambiente:** `http://localhost:8080` en desarrollo.
- **Formato:** JSON en request y response. Fechas en ISO 8601 UTC.
- **Errores:** `application/problem-details` (RFC 9110) con `traceId` para correlación.
  Los errores de validación añaden un objeto `errors` con el campo como clave.

---

## Autenticación

El backend emite tokens **solo a través de `POST /api/v1/auth/login`**. No existe
emisión programática de tokens: una llave de firma compartida queda prohibida.

### `POST /api/v1/auth/login` — público

```json
{ "numeroEmpleado": "201", "password": "…", "dispositivo": "optional" }
```

Respuesta `200`:

```json
{
  "accessToken": "eyJ…",
  "expiraAt": "2026-09-26T04:10:00Z",
  "usuario": {
    "usuarioId": 4, "empleadoId": 4, "sucursalId": 1,
    "numeroEmpleado": "201", "nombreCompleto": "AnaDemo Ventas",
    "rol": "colaborador", "estado": "activo", "correo": "ana@…"
  }
}
```

El `rol` que devuelve el login es el que decide el alcance. No lo calcules en el
cliente: la autorización real vive en el backend.

| Situación | Respuesta |
|---|---|
| Credenciales válidas | `200` |
| Número o contraseña incorrectos | `401` |
| Cuenta `pendiente`, `inactivo` o `rechazado` | `403` |
| Más de 10 intentos por minuto desde una IP | `429` |

Usa el token como `Authorization: Bearer <accessToken>`.

### `GET /api/v1/auth/me` — autenticado

Perfil del usuario del token. Útil al arrancar la app para validar un token
almacenado sin tener que pedir una contraseña.

---

## Recuperación de contraseña

Dos pasos, porque el token viaja por correo y **el backend nunca lo devuelve en una
respuesta**.

### 1. `POST /api/v1/auth/forgot-password` — público

```json
{ "correo": "usuario@correo.com" }
```

Siempre responde `202`, exista o no la cuenta. No permitas inferir de la respuesta si
un correo está registrado: es intencionado.

El backend compone el enlace con `PasswordReset:BaseUrl` y lo envía al correo. **El
cliente no intervene en la construcción del enlace**: recibe el token ya dentro de la
URL y debe navegar a ella.

### 2. `POST /api/v1/auth/reset-password` — público

```json
{ "token": "…", "passwordNueva": "…" }
```

`204` al completar. `400` si el token es inválido, ya se usó o expiró (30 minutos por
defecto, `PasswordReset:TokenMinutes`).

El token es de **un solo uso**: una segunda solicitud de recuperación invalida el
enlace anterior.

> La entrega real por correo depende de `Smtp:*`. Sin configurar, el API responde
> `202` igual, registra un aviso y **descarta el correo**: el usuario nunca recibe el
> enlace. Configura SMTP antes de habilitar el flujo en producción.

### `POST /api/v1/auth/change-password` — autenticado

```json
{ "passwordActual": "…", "passwordNueva": "…" }
```

`204`. `400` si la actual no coincide.

---

## Colaboradores

Todas las rutas requieren token y devuelven `403` cuando el registro queda fuera del
alcance del rol, no `404`.

| Rol | Listado | Expediente ajeno a su alcance | Crear / editar / activar |
|---|---|---|---|
| `rh` | Toda la sucursal | — | Permitido |
| `gerente` | Su equipo | `403` | `403` |
| `colaborador` | Solo él mismo | `403` | `403` |

### `GET /api/v1/employees` — autenticado

Paginada. Devuelve `{ items, page, pageSize, total, totalPages }`.

Filtros opcionales: `page`, `pageSize` (máx. 100), `estadoLaboral`, `departamentoId`,
`equipoId`, `busqueda` (coincide en número de empleado o nombre).

### `GET /api/v1/employees/{id}` — autenticado

`200` con el expediente, `403` si está fuera del alcance.

### `POST /api/v1/employees/self-register` — público

Alta del propio colaborador. `201` con el expediente.

```json
{
  "numeroEmpleado": "555001", "nombreCompleto": "Nombre Apellido",
  "telefono": "5512009900", "correo": "correo@dominio.com", "password": "…",
  "curp": "opcional", "direccion": "opcional", "ciudad": "opcional",
  "estado": "opcional", "codigoPostal": "opcional", "fechaNacimiento": "1990-01-01"
}
```

La cuenta nace `pendiente`: **no puede iniciar sesión hasta que RH la active**. La
respuesta no incluye la contraseña ni su hash.

`telefono` y `numeroEmpleado` son únicos dentro de la sucursal: un duplicado devuelve
`409`.

### Número de empleado — `GET /api/v1/employees/validate-number?numeroEmpleado=…` — solo `rh`

El número de empleado **lo genera el cliente**; el backend solo lo valida. Antes del `POST`,
consulta si el número puede usarse:

```json
{ "numeroEmpleado": "E-0013", "disponible": true, "motivo": null }
```

- Responde **200 siempre**: que el número esté ocupado no es un error HTTP. Mira `disponible`.
- `motivo` explica por qué no está disponible: vacío, más de 50 caracteres o ya existente en la
  sucursal. Muestra ese texto tal cual al usuario.
- Es **solo una consulta**: no reserva el número. `POST /employees` puede seguir respondiendo
  `409` si otro alta se adelanta; maneja ese caso.
- El número recortado (sin espacios) es el que se guarda y el que sirve para iniciar sesión:
  muéstrale al usuario el código que se va a usar.

### Mutadores — solo `rh`

| Ruta | Efecto |
|---|---|
| `POST /api/v1/employees` | Crea colaborador y su cuenta, ya activo |
| `PATCH /api/v1/employees/{id}` | Actualiza el expediente |
| `POST /api/v1/employees/{id}/activate` | Activa cuenta y estado laboral |
| `POST /api/v1/employees/{id}/reject` | Rechaza un auto-registro → `rechazado` |
| `POST /api/v1/employees/{id}/deactivate` | Desactiva y registra la baja |

`activate` es el paso que cierra el alta: sin él la cuenta queda sin acceso.

Para cambiar solo el rol o el estado de la cuenta, usa `/api/v1/users` (sección siguiente):
no toca el expediente y aplica la regla de último RH. Las rutas de arriba siguen vigentes y
ahora aplican las mismas reglas: 409 si el cambio deja la sucursal sin RH activo o si el RH
se afecta a sí mismo.

En `POST` y `PATCH`, `puestoId`, `departamentoId`, `equipoId` y `horarioId` se validan:
`404` si el catálogo no existe en la sucursal, `400` (con el campo en `errors`) si está
dado de baja. En `PATCH`, enviar `null` u omitir el campo **no** desasigna: hoy no hay
forma de dejar vacío uno de estos valores.

---

## Forma de los errores

Todos los mensajes de error vienen en **español** y se pueden mostrar directamente al usuario.
La respuesta sigue `application/problem+json`:

| Caso | `title` | Dónde está el texto |
|---|---|---|
| Campos inválidos (400) | `Uno o más campos no son válidos.` | `errors`: mapa campo → mensajes, p. ej. `{"NumeroEmpleado": ["El número de empleado no puede superar 50 caracteres."]}` |
| Regla de negocio (400/404/409) | `Solicitud inválida`, `Recurso no encontrado`, `Conflicto con el estado actual`… | `detail` |
| Sin permiso (403) | `Acceso denegado` | `detail` |
| Sin sesión (401) | `No autorizado` | Sin cuerpo: el navegador descarta la respuesta por CORS, así que trata la sesión expirada con `expiraAt` (ver arriba) |
| Ruta o método inexistente (404/405) | `Recurso no encontrado` / `Método no permitido` | `detail` |
| Demasiados intentos (429) | `Demasiadas solicitudes` | `detail` |

Para formularios, pinta cada mensaje de `errors` junto a su campo y no muestres el `title` en
ese caso.

## Configuración operativa

Lectura abierta a cualquier rol autenticado, siempre de la sucursal del token. Escritura
solo `rh`: `gerente` y `colaborador` reciben `403`.

### Catálogos: `departments`, `positions`, `schedules`, `teams`

| Ruta | Efecto |
|---|---|
| `GET /api/v1/{recurso}` | Paginado. `activo` (por defecto `true`; `false` lista los dados de baja), `busqueda`. `teams` acepta además `departamentoId` |
| `GET /api/v1/{recurso}/{id}` | Detalle. `404` si no existe |
| `POST /api/v1/{recurso}` | `201`. `409` si el nombre ya existe, también contra uno dado de baja |
| `PATCH /api/v1/{recurso}/{id}` | Parcial: lo omitido o `null` no cambia |
| `DELETE /api/v1/{recurso}/{id}` | `204`. Baja lógica: el registro sigue consultable con `activo: false` |

Un catálogo dado de baja **no** se quita de los colaboradores que ya lo tienen; solo deja
de aceptarse en asignaciones nuevas. Para los selectores del panel, pide `activo=true`.

**Horarios.** Horas en formato `HH:mm:ss`. La comida va completa (`horaSalidaComida` y
`horaRegresoComida`) o no va. Si no envías `toleranciaEntrada` al crear, el horario hereda
la de la tienda; cambiar después la de la tienda no altera los horarios existentes.

**Gerentes de equipo.**

| Ruta | Rol | Efecto |
|---|---|---|
| `GET /api/v1/teams/{id}/managers` | cualquiera | Lista de gerentes |
| `POST /api/v1/teams/{id}/managers` | `rh` | `{ "empleadoId": 7 }`. `201`; `400` si no es un gerente activo; `409` si ya estaba |
| `DELETE /api/v1/teams/{id}/managers/{empleadoId}` | `rh` | `204` |
| `GET /api/v1/teams/{id}/members` | cualquiera | Miembros, filtrados por el alcance del rol |

Un equipo puede tener varios gerentes. El cambio de alcance es inmediato: el gerente ve
o deja de ver a los miembros en su siguiente petición, sin volver a iniciar sesión.

### Tienda y zona — `GET /api/v1/store`

Cualquier rol. **Es lo que la app móvil necesita para el geofencing del checador**:

```json
{
  "id": 1, "nombre": "Ferretería Cotla", "direccion": "…", "ciudad": "…",
  "estado": "…", "codigoPostal": "…",
  "latitud": 19.432608, "longitud": -99.133209,
  "radioPermitidoMetros": 100, "toleranciaEntrada": 5, "activa": true, "…": "…"
}
```

`PATCH /api/v1/store` (solo `rh`) edita los mismos campos. Validaciones: latitud entre
-90 y 90, longitud entre -180 y 180, radio mayor que 0, tolerancia no negativa, RFC de 12
o 13 caracteres. En `razonSocial`, `rfc`, `telefono` y `correo` la cadena vacía borra el
valor.

### Datos públicos de la empresa — `GET /api/v1/public/company` (sin sesión)

Para el aviso de privacidad que exigen las tiendas de aplicaciones (plantilla en
`docs/especificaciones.md`). Anónimo y sin caché. Devuelve solo contacto, sin RFC ni
coordenadas, tomado de lo que captura RH en `PATCH /api/v1/store`:

```json
{
  "nombre": "Sucursal principal",
  "razonSocial": "Ferretería Cotla S.A. de C.V.",
  "telefono": "5555555555",
  "correo": "privacidad@example.com",
  "direccion": "Calle 1 #2",
  "ciudad": "Ciudad",
  "estado": "Estado",
  "codigoPostal": "00000"
}
```

`razonSocial`, `telefono` y `correo` pueden ser `null`: la página debe avisar que faltan en
lugar de publicar un aviso incompleto. `404` si no hay sucursal activa.

### Saldos de vacaciones — `/api/v1/vacation-balances`

| Ruta | Quién | Notas |
|---|---|---|
| `GET /vacation-balances` | `rh` (toda la sucursal), `gerente` (sus equipos) | `colaborador` recibe `403`. Filtros: `departamentoId`, `equipoId`, `busqueda`, `soloConSaldo` (`false` = quien aún no tiene saldo) |
| `GET /vacation-balances/{empleadoId}` | cualquiera, dentro de su alcance | La app móvil consulta aquí el saldo propio con el `empleadoId` del login. `404` si aún no tiene saldo |
| `GET /vacation-balances/{empleadoId}/movements` | cualquiera, dentro de su alcance | Historial paginado, más reciente primero |
| `PUT /vacation-balances/{empleadoId}` | `rh` | `{ "diasIniciales": 12, "descripcion": "…" }` crea o corrige el saldo inicial |
| `POST /vacation-balances/{empleadoId}/adjustments` | `rh` | `{ "dias": -2, "descripcion": "…" }`. Descripción obligatoria. `409` si aún no hay saldo inicial |

`diasDisponibles` lo calcula el servidor: muéstralo, no lo calcules en el cliente.

---

## Checador de asistencia — `/api/v1/attendance` (FASE 4)

Lo usa **cualquier rol**: RH y gerentes también marcan su propia jornada. La hora oficial
la pone el **servidor**; el cliente nunca la envía.

### Marcar — `multipart/form-data`

| Ruta | Marcación |
|---|---|
| `POST /attendance/check-in` | Entrada |
| `POST /attendance/lunch-out` | Salida a comer (opcional) |
| `POST /attendance/lunch-in` | Regreso de comer |
| `POST /attendance/check-out` | Salida |

| Campo | Tipo | Reglas |
|---|---|---|
| `foto` | archivo | Obligatorio. JPEG o PNG reales (se verifica la firma del archivo, no el `Content-Type`). Máx. 5 MB. Tómala con la cámara frontal: el backend no puede impedir la galería |
| `latitud` | decimal | Obligatorio, entre -90 y 90 |
| `longitud` | decimal | Obligatorio, entre -180 y 180 |
| `precisionMetros` | decimal? | Precisión del GPS, 0–999999 |
| `eventoDispositivoId` | string | **Obligatorio**, máx. 100. Un UUID generado por la app **por cada intento de marcación** |
| `registradaEnDispositivoAt` | string? | Solo para **marcaciones capturadas sin conexión**: la hora del teléfono en ISO 8601 **con zona** (p. ej. `2026-10-22T08:58:03-06:00`). Sin este campo la marca es en línea. Sin zona o con formato inválido → `400` |

```bash
curl -X POST $BASE/attendance/check-in -H "Authorization: Bearer $TOKEN" \
  -F "foto=@selfie.jpg;type=image/jpeg" -F latitud=19.432608 -F longitud=-99.133209 \
  -F precisionMetros=8 -F eventoDispositivoId=6f1c2a8e-3b0f-4a4e-9a51-1d2f4c7b9e10
```

Respuestas:

| Código | Cuándo |
|---|---|
| `201` | Marcación creada. `Location: /api/v1/attendance/{id}` |
| `200` | **Reintento**: ya existía una marcación con ese `eventoDispositivoId`, del mismo colaborador y del mismo tipo. Devuelve la original sin duplicar |
| `400` | Campo inválido; `errors` trae la clave del campo (`foto`, `latitud`, `registradaEnDispositivoAt`, …). También si la hora del dispositivo va **adelantada** más de 2 minutos respecto al servidor |
| `403` | El colaborador no está activo |
| `409` | Secuencia inválida (p. ej. salida sin entrada), el tipo ya se registró hoy, el `eventoDispositivoId` ya se usó en otra marcación, **fuera de la ventana de horario** (ver abajo), la marca capturada tiene **más de 24 h**, o es **anterior a otra ya registrada ese día** |
| `503` | El almacenamiento de fotos no respondió; la marcación **no** se guardó. Reintenta con el mismo UUID |

**Reintentos.** Si la petición falla por red o con `503`, reenvía con el **mismo**
`eventoDispositivoId`: si el servidor sí la había guardado responde `200` con la original.
Genera un UUID nuevo solo para una marcación nueva.

**Secuencia del día:** entrada → [salida a comer → regreso de comer] → salida. La comida
es opcional, pero si se abre debe cerrarse antes de la salida. Cada tipo, una vez por día
(día calendario en la zona `America/Mexico_City`).

**Fuera de zona no bloquea.** La marcación se guarda con `estadoZona = fuera_de_zona` y su
`distanciaSucursalMetros`. El retardo se informa en la entrada: `minutosRetardo` (minutos después
de `horaEntrada + toleranciaEntrada`; `null` si no tiene horario).

**Horario (límites provisionales, pendientes de confirmar con el cliente).** Hay una ventana de
marcación por colaborador: desde 30 minutos antes de su hora de entrada hasta 60 minutos después
de su hora de salida. Una marcación fuera de ella responde `409` con un mensaje como
«Fuera de horario: puedes marcar de 08:30 a 19:00.», que puedes mostrar tal cual. `GET /attendance/today`
trae `ventanaMarcacion { desde, hasta }` para que la app muestre el horario permitido antes de
intentarlo; es `null` si el bloqueo está apagado o el colaborador no tiene horario.

**Marcaciones sin conexión.** Si la app guarda una marca mientras no hay red, envía
`registradaEnDispositivoAt` con la hora **de ese momento** y la zona horaria del teléfono, y
reenvía después con el **mismo** `eventoDispositivoId`. Reglas:

- Si la hora del teléfono está a menos de 2 minutos del servidor, se usa la del servidor (la marca
  es en línea).
- Si el teléfono va adelantado, `400`: pide al usuario revisar la fecha y la hora del teléfono.
- Si la marca tiene más de 24 horas, `409`: avisa a RH. Ese límite es provisional.
- La jornada, el retardo y la ventana se calculan con la **hora capturada**, no con la de llegada.
- Las marcas del mismo día se deben enviar **en orden de captura**. Si una marca es anterior a otra
  ya registrada ese día, responde `409`.

Respuesta `AsistenciaResponse`:

```json
{
  "id": 12, "empleadoId": 7, "numeroEmpleado": "205", "nombreCompleto": "Ana García",
  "fechaLocal": "2026-10-02", "tipo": "entrada", "registradaAt": "2026-10-02T15:04:11Z",
  "latitud": 19.432608, "longitud": -99.133209, "precisionMetros": 8.00,
  "distanciaSucursalMetros": 0.00, "estadoZona": "dentro_de_zona", "minutosRetardo": 0,
  "fotoUrl": "/api/v1/attendance/12/photo", "eventoDispositivoId": "6f1c2a8e-…",
  "origen": "en_linea", "sincronizadaAt": "2026-10-02T15:04:11Z", "desfaseSegundos": 0
}
```

`origen` es `en_linea` o `sincronizada`. Una marca `sincronizada` tiene `registradaAt` anterior a
`sincronizadaAt`, y `desfaseSegundos` es la diferencia. RH puede usar esos campos para revisar
marcas que llegaron tarde.

### Consultar

| Ruta | Alcance | Notas |
|---|---|---|
| `GET /attendance/today` | el propio usuario | `marcaciones` del día, `siguientesPermitidas` (qué botones habilitar), `faltantesObligatorias` (`entrada`/`salida`), `horario` asignado y `ventanaMarcacion` (horario permitido). Úsalo en lugar de replicar la secuencia en la app |
| `GET /attendance` | `rh` todo, `gerente` sus equipos, `colaborador` lo suyo | Paginado, más reciente primero. Filtros: `fechaDesde`, `fechaHasta` (`YYYY-MM-DD`), `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estadoZona`. **No envíes filtros vacíos** (`empleadoId=` responde `400`) |
| `GET /attendance/{id}` | mismo alcance | `{ "marcacion": AsistenciaResponse, "tienda": { latitud, longitud, radioPermitidoMetros } }` para pintar el mapa |
| `GET /attendance/{id}/photo` | mismo alcance | La selfie (`image/jpeg` o `image/png`). Requiere el `Bearer`: en web, descárgala con `fetch` y muéstrala como `blob:`; un `<img src>` directo no envía el token |

---

## Incidencias, correcciones y ausencias (FASES 5 y 6)

Todas las rutas requieren `Bearer`. Errores con la forma de la sección «Forma de los errores».
Los listados son paginados (`page`, `pageSize`) y **no deben enviar filtros vacíos**.

### Incidencias — `/api/v1/incidents`

| Ruta | Quién | Notas |
|---|---|---|
| `GET /incidents` | `rh` todo, `gerente` sus equipos (lectura), `colaborador` lo suyo | Filtros: `fechaDesde`, `fechaHasta` (`YYYY-MM-DD`), `empleadoId`, `departamentoId`, `equipoId`, `tipo` (`entrada_faltante`, `salida_faltante`, `fuera_de_zona`), `estado` (`pendiente`, `resuelta`, `descartada`) |
| `GET /incidents/{id}` | mismo alcance | |
| `POST /incidents/{id}/resolve` · `/dismiss` | solo `rh` | Cuerpo opcional `{ "descripcion": "…" }`; `409` si ya fue atendida |
| `POST /incidents/detect` | solo `rh` | Ejecuta la detección ahora; responde `{ "creadas": n }`. Corre sola cada 30 min |

Las incidencias se generan de jornadas **cerradas** (días anteriores a hoy) que tengan marcaciones.

### Correcciones de marcación — `/api/v1/attendance-corrections`

| Ruta | Quién | Notas |
|---|---|---|
| `POST /attendance-corrections` | el dueño de la marcación | JSON `{ asistenciaId, motivo, fechaHoraPropuesta?, tipoPropuesto? }`. Hora ISO 8601 con zona, del mismo día y no futura; al menos hora o tipo. `409` si ya hay una pendiente; `403` si la marcación es ajena |
| `GET /attendance-corrections` · `/{id}` | `rh` todo, `gerente` sus equipos, `colaborador` lo suyo | Filtros: `empleadoId`, `asistenciaId`, `estado` |
| `POST /attendance-corrections/{id}/approve` | solo `rh` | La marcación original **no cambia**; `GET /attendance/{id}` agrega `correccionAprobada { correccionId, fechaHoraPropuesta, tipoPropuesto, revisadoAt }` |
| `POST /attendance-corrections/{id}/reject` | solo `rh` | `{ "motivoRechazo": "…" }` obligatorio |

### Solicitudes de ausencia — `/api/v1/leave-requests`

`POST` es **`multipart/form-data`**: `tipo` (`vacaciones`, `permiso_con_goce`, `permiso_sin_goce`,
`incapacidad`), `fechaInicio` y `fechaFin` (`YYYY-MM-DD`, enteras, no pasadas), `motivo?` y
`documentos` (archivos repetidos; **obligatorio en `incapacidad`**, prohibido en los demás; JPEG,
PNG o PDF, máx. 5 MB y 3 archivos). Los días son naturales y los calcula el servidor
(`diasSolicitados`).

| Ruta | Quién | Notas |
|---|---|---|
| `POST /leave-requests` | cualquier usuario, para sí mismo | `201` con la solicitud en `pendiente`. `400` sin saldo suficiente (vacaciones), fechas inválidas o documento inválido; `409` si traslapa otra solicitud pendiente/aprobada propia. **No descuenta saldo** |
| `GET /leave-requests` | `rh` todo, `gerente` sus equipos, `colaborador` lo suyo | Filtros: `estado`, `tipo`, `fechaDesde`, `fechaHasta`, `empleadoId`, `equipoId`. Incluye `motivoRechazo` y `documentos[]` |
| `GET /leave-requests/{id}` | mismo alcance | |
| `GET /leave-requests/{id}/documents/{archivoId}` | mismo alcance | Requiere `Bearer`: en web, `fetch` + `blob:` |
| `POST /leave-requests/{id}/cancel` | solo el dueño | Solo `pendiente`; si no, `409` |
| `POST /leave-requests/{id}/approve` | solo `rh` | Vacaciones: revalida y descuenta el saldo y registra un movimiento `uso`; `409` si el saldo ya no alcanza o no está pendiente |
| `POST /leave-requests/{id}/reject` | solo `rh` | `{ "motivoRechazo": "…" }` obligatorio |

Al aprobar o rechazar se guarda una notificación in-app; los endpoints para consultarla y el
envío push/correo llegan con la FASE 10.

---

## Horas extra — `/api/v1/overtime` (FASE 7)

| Endpoint | Quién | Notas |
|---|---|---|
| `POST /overtime` | cualquier usuario, para sí mismo | `multipart/form-data`: `fecha` (`yyyy-MM-dd`), `horaInicio`, `horaFin` (`HH:mm`), `concepto`, `comentarios?`, `fotos` (0–5 JPEG/PNG, 5 MB c/u). `201` en `pendiente`. **No se envían horas**: `totalHoras` lo calcula el servidor |
| `GET /overtime` | `rh` todo, `gerente` sus equipos, `colaborador` lo suyo | Filtros: `estado`, `fechaDesde`, `fechaHasta`, `empleadoId`, `departamentoId`, `equipoId` |
| `GET /overtime/{id}` | mismo alcance | Incluye `motivoRechazo` y `fotos[]` |
| `GET /overtime/{id}/photos/{archivoId}` | mismo alcance | Requiere `Bearer`: en web, `fetch` + `blob:` |
| `POST /overtime/{id}/approve` | `rh` | `409` si ya no está pendiente |
| `POST /overtime/{id}/reject` | `rh` | Cuerpo `{ "motivoRechazo": "..." }` obligatorio |

## Reportes — `/api/v1/reports` (FASE 8)

Solo consulta, paginados, con el alcance del token (`rh` todo, `gerente` sus equipos, `colaborador` lo suyo). Fechas `yyyy-MM-dd`.

| Endpoint | Respuesta | Filtros |
|---|---|---|
| `GET /reports/attendance` | `PagedResult<AsistenciaResponse>` | `fechaDesde`, `fechaHasta`, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estadoZona` |
| `GET /reports/incidents` | Filas `{ origen: incidencia\|solicitud\|correccion, referenciaId, tipo, estado, fecha, … }`. Sin `estado`, solo pendientes | fechas, `empleadoId`, `departamentoId`, `equipoId`, `tipo` (`entrada_faltante`, `salida_faltante`, `fuera_de_zona`, `solicitud_pendiente`, `correccion_pendiente`), `estado` |
| `GET /reports/overtime` | `{ registros, totalHorasFiltradas }` | fechas, `empleadoId`, `departamentoId`, `equipoId`, `estado` |
| `GET /reports/vacations` | Por colaborador: saldo + `solicitudes[]` | fechas, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estado` |

`400` si `fechaDesde > fechaHasta` o un `tipo`/`estado` no existe. Fotos: usa `fotoUrl` / `fotos[].url` con `Bearer`.

## Dashboard y calendario (FASE 9)

| Endpoint | Quién | Notas |
|---|---|---|
| `GET /api/v1/dashboard?dias=` | `rh`, `gerente` (colaborador → `403`) | Contadores del día: `colaboradoresActivos`, `presentesHoy`, `ausentes`, `deLicencia`, `marcacionesFueraDeZonaHoy`, pendientes de incidencias, solicitudes, correcciones y horas extra. `proximosCumpleanos` y `proximosAniversarios` solo para RH (`null` para gerente); `dias` 1–90, 30 por defecto |
| `GET /api/v1/calendar?desde=&hasta=&empleadoId=` | cualquier usuario | Rango obligatorio de máx. 62 días. Sin `empleadoId` es el propio; el de otro, solo RH. Eventos: `ausencia` (aprobadas), `asistencia`, `incidencia`, `hora_extra` (aprobadas) |

## Notificaciones — `/api/v1/notifications` (FASE 10)

**Móvil:** no hay push. El colaborador recibe **correo** cuando se aprueba o rechaza una ausencia u hora extra, y puede consultar la bandeja por API. Sugerencia: `GET /notifications/unread-count` al abrir la app y cada ~60 s en primer plano. Los cumpleaños y aniversarios **no** envían correo (solo bandeja). **Panel web:** eventos en tiempo real por SSE (sección siguiente).

| Endpoint | Notas |
|---|---|
| `GET /notifications?leida=&page=&pageSize=` | Solo las del usuario del token, de la más reciente a la más antigua |
| `GET /notifications/unread-count` | `{ "noLeidas": n }` |
| `POST /notifications/{id}/read` · `/unread` | Devuelve la notificación; `404` si no es del usuario |
| `POST /notifications/read-all` | `{ "marcadas": n }` |
| `POST /notifications/devices` | `{ "token", "plataforma": "android\|ios\|web" }`; guarda el token por si se retoma el push (**no se envía nada**; el móvil no necesita llamarlo) |
| `DELETE /notifications/devices/{id}` | Baja del dispositivo propio (`204`) |
| `POST /notifications/alerts/generate` | Solo `rh`: genera ya las alertas del día (idempotente) |

Notificación: `{ id, tipo, titulo, mensaje, leida, leidaAt, createdAt, datos }`. `tipo`: `solicitud_ausencia_aprobada|rechazada`, `hora_extra_aprobada|rechazada` (el rechazo incluye el motivo en `mensaje`), `cumpleanos_propio|equipo`, `aniversario_propio|equipo`. `datos` trae el id para navegar (`solicitudId`, `horaExtraId`, `empleadoId`). Las alertas de cumpleaños/aniversarios del equipo las reciben RH y el gerente del equipo.

## Panel: eventos en tiempo real — `GET /api/v1/panel/events` (SSE)

Solo `rh` y `gerente`. **Extensión fuera de la especificación**, pensada solo para el panel (el móvil no lo usa).

- Conexión: `fetch('https://api.yefrasoft.com/api/v1/panel/events', { headers: { Authorization: 'Bearer <token>', Accept: 'text/event-stream' } })` leyendo el stream; con `@microsoft/fetch-event-source` basta pasar `headers`. **El `EventSource` nativo no sirve** (no permite `Authorization`) y el token **no** se acepta en la URL.
- Eventos SSE: `conectado` (al suscribirse), `latido` (cada 25 s, ignorar), `evento` con `data` `{ "tipo", "ocurridoAt", "datos" }`:
  - `asistencia` → `asistenciaId`, `empleadoId`, `numeroEmpleado`, `nombreCompleto`, `tipo` (`entrada|salida_comida|regreso_comida|salida`), `fechaLocal`, `registradaAt`, `estadoZona`.
  - `solicitud_ausencia` → `solicitudId`, empleado, `tipo`, `fechaInicio`, `fechaFin`, `diasSolicitados`.
  - `correccion_asistencia` → `correccionId`, `asistenciaId`, empleado, `tipoOriginal`, `fechaLocal`, `motivo`.
- RH recibe todo; el gerente solo lo de sus equipos. **No hay repetición**: al abrir o reconectar, vuelve a cargar los listados por REST. Reconectar con espera creciente si se corta. Máximo 5 conexiones por usuario (varias pestañas: la sexta cierra la más antigua).
- Hoy el CORS está abierto (1.0.0). Si en el futuro cambia a una lista de orígenes, debe incluir el del panel y permitir la cabecera `Authorization`.

## Auditoría — `GET /api/v1/audit` (FASE 11, solo `rh`)

Paginado, más reciente primero, de la sucursal del token. Filtros: `fechaDesde`, `fechaHasta` (`yyyy-MM-dd`, días locales), `usuarioId`, `accion`, `entidad`, `entidadId`. Cada item trae `fecha` (UTC), `numeroEmpleado`/`nombreCompleto` del responsable, `accion`, `entidad`, `entidadId`, `ip` y `datosAnteriores`/`datosNuevos` (objetos JSON). Contiene datos personales: mostrar solo a RH. `403` para gerente y colaborador.

## Caché y fotos (FASE 11)

- **Revalidar con `ETag`:** los `GET` de catálogos (`/departments`, `/positions`, `/schedules`, `/teams`, `/store`) y las descargas de fotos (`/attendance/{id}/photo`, `/overtime/{id}/photos/{archivoId}`) devuelven `ETag` y `Cache-Control: private, no-cache`. Enviar `If-None-Match: <etag>` en la siguiente petición: si no cambió responde **`304` sin cuerpo** (reutilizar lo guardado). El navegador lo hace solo con `fetch`; en el móvil hay que guardar el `ETag` junto al dato. Las descargas siguen exigiendo `Authorization: Bearer`.
- **Frescura:** un catálogo puede tardar hasta 10 min en reflejar un cambio hecho por fuera de la API; los cambios hechos por la API se ven al instante. `dashboard` y `reports/*` pueden tener hasta 15 s de retraso.
- **Las fotos cambian de formato:** poco después de subirse, una selfie o evidencia se reemplaza por una versión **WebP** (máx. 1600 px, sin EXIF/GPS). La URL no cambia, pero sí `Content-Type` (`image/webp`) y `ETag`; no asumir `image/jpeg` ni conservar el binario descargado como si fuera el original. Los documentos de incapacidad no se modifican.
- **Cabeceras de seguridad:** todas las respuestas llevan `nosniff`, `X-Frame-Options: DENY`, etc.; no afecta a los clientes.

## Usuarios y roles — `/api/v1/users` (solo `rh`)

Panel de cuentas: asignar rol, activar o desactivar, revocar sesiones y resetear contraseña,
sin tocar el expediente. El `{id}` es el de la **cuenta** (`UsuarioResumen.id`); el
`empleadoId` viene en la misma respuesta.

| Ruta | Uso |
|---|---|
| `GET /api/v1/users?estado=&rol=&busqueda=&page=&pageSize=` | Lista paginada (`PagedResult`) |
| `GET /api/v1/users/{id}` | Detalle, con `sesionesActivas` |
| `PUT /api/v1/users/{id}/role` | Body `{ "rol": "colaborador" \| "gerente" \| "rh" }` |
| `POST /api/v1/users/{id}/activate` | Activa la cuenta |
| `POST /api/v1/users/{id}/reject` | Rechaza un auto-registro pendiente |
| `POST /api/v1/users/{id}/deactivate` | Body `{ "motivo": "..." }`; también revoca sesiones |
| `POST /api/v1/users/{id}/sessions/revoke` | Revoca las sesiones activas |
| `POST /api/v1/users/{id}/password-reset` | 204. Envía el enlace de recuperación al correo del colaborador |

Errores de negocio (409) que conviene mostrar al operador:

- Quitar el rol RH, desactivar o rechazar al **único RH activo** de la sucursal.
- Quitarte a ti mismo el rol RH, desactivarte o rechazarte.
- Resetear la contraseña de una cuenta que no está `activo`, o sin correo registrado.

RH nunca ve ni define la contraseña: el colaborador recibe el enlace por correo.

---

## Sesiones

| Ruta | Rol | Efecto |
|---|---|---|
| `POST /api/v1/auth/logout` | cualquiera | Marca `RevocadaAt` en las sesiones activas |
| `POST /api/v1/auth/sessions/revoke-all?usuarioId={id}` | `rh` | Revoca todas las sesiones de ese usuario (404 si no es de tu sucursal) |
| `POST /api/v1/users/{id}/sessions/revoke` | `rh` | Igual, con el `id` de la cuenta |

> **Ninguna de estas rutas invalida los tokens ya emitidos.** Un `Bearer` sigue funcionando
> hasta su expiración. Desactivar la cuenta impide iniciar sesión de nuevo, pero no corta
> un token vigente antes de que expire.

`revoke-all` toma el `usuarioId` en el **query string**, no en el cuerpo.

---

## CORS

En la versión 1.0.0 el API responde con **CORS abierto** (`AllowAnyOrigin`) en todos los
ambientes: es una deuda aceptada (`progreso-backend.md` §32). `Cors:AllowedOrigins:*` se
exige en la configuración pero todavía no se aplica; los clientes no deben depender de que
un origen sea rechazado. Cuando se active la lista, deberá incluir los orígenes de abajo.

| Ambiente | Orígenes previstos |
|---|---|
| Desarrollo | `http://localhost:3000` (panel web), `http://localhost:8081` (app móvil) |
| Producción | Los del panel y la app (pendiente de activar) |

Un preflight debe devolver `204` con `Access-Control-Allow-Origin` y
`Access-Control-Allow-Methods`.

> `localhost:8081` es el puerto habitual de Metro, pero Expo decide el puerto real en
> cada arranque. Si la app móvil no conecta, es lo primero que hay que revisar: el
> origen cambia entre ejecuciones.

---

## Límite de intentos

El bucket `auth` cubre `login`, `self-register`, `forgot-password` y
`reset-password`: **10 peticiones por minuto y por IP**, y a partir de la siguiente se
responde `429`.

Disparado por IP, no por cuenta. Varias personas detrás de la misma salida de red
comparten el mismo cupo.

**El `429` no trae cabecera `Retry-After`.** El cuerpo es un `ProblemDetails` con el
título `Demasiadas solicitudes` y el detalle `Se excedió el límite de intentos.
Intenta de nuevo en un minuto.`; el tiempo de espera hay que sacarlo del mensaje, no
de una cabecera.

Maneja el `429` reintentando tras un minuto, no acumulando reintentos: cada reintento
consume cuota y prolonga el bloqueo.

---

## Configuración del servidor

Exposición por **Cloudflare Tunnel** (el backend se despliega desde el equipo del proyecto; solo estos dos nombres son públicos y el tunnel se gestiona en Cloudflare, no en el repositorio):

| Nombre público | Service interno | Destino final |
|---|---|---|
| `https://by.yefrasoft.com` | `http://127.0.0.1:3000` | Panel web (Next.js), servicio `frontend` del compose |
| `https://api.yefrasoft.com` | `http://127.0.0.1:8088` | API directo (puerto `8088` del compose → `8080` del contenedor). **Caddy ya no interviene** |

El panel y el API están en **orígenes distintos**, así que el frontend llama a
`NEXT_PUBLIC_API_URL=https://api.yefrasoft.com`. Esa variable se incrusta al compilar
(`bun run build`): cambiarla exige reconstruir. En el compose se controla con `FRONTEND_API_URL`
(por defecto `https://api.yefrasoft.com`). La app móvil puede usar la misma base URL.

### Panel web en el compose

El servicio `frontend` del `docker-compose.yml` construye `app-frontend/Dockerfile` (lo mantiene el equipo web) con `build.args.NEXT_PUBLIC_API_URL` tomado de `FRONTEND_API_URL`, y publica el puerto `3000` que usa el tunnel de `by.yefrasoft.com`. Entra en el `docker compose up -d --build` normal. Contrato que el Dockerfile debe seguir: declarar `ARG NEXT_PUBLIC_API_URL` **antes** de `next build` (y fallar si llega vacío), usar solo Bun (sin Node; imagen `oven/bun`) para instalar desde `bun.lock`, arrancar con `bun --bun next start -p 3000 -H 0.0.0.0`, y un `.dockerignore` con `node_modules` y `.next`. **Versión de bun:** `1.4.4`, en el Dockerfile y en `package.json` (`packageManager`); la imagen se construyó y arrancó con ella el 20-nov-2026 (`/privacidad` y `/` responden 200).

| Variable | Para qué |
|---|---|
| `JwtSettings__SigningKey` | Clave HMAC-SHA256, mínimo 32 bytes. Sin ella el API no arranca |
| `ConnectionStrings__Postgres` | Cadena de conexión. En el compose se arma con `POSTGRES_PASSWORD` del `.env` (obligatoria) |
| `Cors__AllowedOrigins__0`, `__1` | Orígenes permitidos. Se exigen, pero en 1.0.0 el CORS está abierto (deuda aceptada) |
| `Seed__RhNumeroEmpleado`, `Seed__RhPassword` | Cuenta de RH que se crea en base vacía. Obligatorio en producción |
| `Seed__GerenteNumeroEmpleado`, `Seed__GerentePassword`, `Seed__GerenteNombre` | Cuenta de gerente. Obligatorio en producción |
| `Seed__CrearColaboradoresDemo` | Colaboradores de demostración (201–203). Por defecto `false` en el compose |
| `Seed__SucursalLatitud`, `Seed__SucursalLongitud` | Ubicación de la sucursal. Sin ellas el geofence queda en `0,0` |
| `Smtp__Host`, `__Port`, `__User`, `__Password`, `__From` | Transporte del correo de recuperación |
| `PasswordReset__BaseUrl` | URL pública del enlace de recuperación. Obligatoria en el compose |
| `PasswordReset__TokenMinutes` | Vigencia del token. Por defecto 30 |
| `RateLimiting__Enabled` | Permite desactivar el límite de intentos |
| `Storage__Endpoint`, `__AccessKey`, `__SecretKey`, `__Bucket`, `__UseSsl` | Almacenamiento S3 (MinIO) de las selfies. Sin credenciales el API no arranca |
| `Asistencia__ZonaHoraria` | Zona IANA de la jornada. Por defecto `America/Mexico_City` |
| `Asistencia__FotoMaxBytes` | Tamaño máximo de la selfie. Por defecto 5 MB |

Las contraseñas de semilla deben cumplir la política: mínimo 8 caracteres.

---

## Lo que el backend todavía no expone

No consumas estos contratos: no existen.

- **Expediente completo.** `GET /api/v1/employees/{id}` devuelve los datos del
  colaborador y su estado de cuenta, pero no documentos laborales ni historial.
- **Marcación sin conexión.** El checador es solo en línea: no acepta marcaciones con hora
  del dispositivo ni sincronización diferida (§18, pendiente con el cliente).
- **Configuración de la cuenta del propio usuario.** No hay ruta para que un
  colaborador cambie su correo o teléfono; solo puede cambiar su contraseña.
- **Lista de sucursales.** Solo existe la tienda propia (`GET /store`): no hay selector de
  sucursal porque el MVP opera una sola.
- **Devengo automático de vacaciones.** El saldo cambia por saldo inicial, ajustes de RH y
  consumos al aprobar solicitudes; el devengo sigue pendiente (D1).
- **Consulta de notificaciones y horas extra.** FASES 7 y 10.
- **Documentación OpenAPI en producción.** Solo está habilitada en `Development`.

---

## Referencia rápida

```bash
BASE=http://localhost:8080/api/v1

# Login
TOKEN=$(curl -s -X POST $BASE/auth/login -H 'Content-Type: application/json' \
  -d '{"numeroEmpleado":"201","password":"…"}' | python3 -c 'import json,sys;print(json.load(sys.stdin)["accessToken"])')

# Listar colaboradores
curl -s "$BASE/employees?pageSize=100" -H "Authorization: Bearer $TOKEN"

# Alta del propio colaborador
curl -s -X POST $BASE/employees/self-register -H 'Content-Type: application/json' -d '{…}'
```

## Relacionado

- [[rh-cotla-plan]] — plan padre del proyecto.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos y decisiones del backend en detalle, y runbook de despliegue.
- [[rh-cotla-pendientes-sub-plan]] — lo que el API todavía no expone (`F-BE-1` a `F-BE-4`) y otros pendientes.
