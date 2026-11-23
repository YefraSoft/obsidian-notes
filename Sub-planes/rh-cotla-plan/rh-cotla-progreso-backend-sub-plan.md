---
tipo: sub-plan
titulo: "Backend — Progreso y contratos implementados"
autor: Efraín García
creado: 2026-09-11
actualizado: 2026-11-20
estado: vigente
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - backend
  - contratos
  - deuda-tecnica
---
# Backend — Progreso y contratos implementados

> **Plan padre:** [[rh-cotla-plan]]

> Estado del módulo `app-backend/cotla-api/` al 12 de noviembre de 2026.
> Plan vivo y pendientes del backend: [[rh-cotla-backend-sub-plan]].
>
> Este documento describe **solo lo terminado e implementado**. Las FASES 2 a 12 del MVP (versión 1 cerrada, ver §32)
> quedan cerradas en backend. Las pantallas (§2.5, §2.6, §3.6, §4.3, §4.4 y §4.6 del plan)
> corresponden a los equipos web y móvil y quedan fuera de este módulo. La configuración
> operativa de la FASE 3 se describe en las secciones 11 a 15; el almacenamiento de archivos
> y el checador de la FASE 4, en las secciones 16 a 18; incidencias, correcciones y ausencias
> (FASES 5 y 6), en la sección 21.

## 1. Estado actual

- **Compilación:** `dotnet build cotla-api.slnx` → 0 errores, 0 advertencias.
- **Pruebas:** `dotnet test cotla-api.slnx` sobre `postgres:17-alpine` real vía Testcontainers. Conteo por archivo, decisiones de infraestructura y límites en la sección 6.
- **Formato:** `dotnet format cotla-api.slnx --verify-no-changes` en verde. La deuda de espaciado y salto final que arrastraban siete archivos de la FASE 2 se corrigió al cerrar la FASE 3.
- **Pila real:** `docker compose up --build -d` en `Production` responde `Healthy` en `/health` (Postgres y almacenamiento). El 2 de octubre de 2026 se recorrieron **todos** los endpoints con tokens reales de `rh`, `gerente` y `colaborador`: 119 comprobaciones, 0 fallas (sección 6). La colección de Insomnia `RH Cotla API` contiene los 61 requests para repetirlo a mano.
- **Almacenamiento de archivos:** compatible con S3 vía `IFileStorage` (`Common/Storage`), implementado con el SDK de MinIO. El bucket se crea al arrancar y `/health` incluye el check `almacenamiento`. Detalle en la sección 16.
- **Base de datos:** migración EF `Fase2AutenticacionYColaboradores` generada (columna `empleados.correo`, tabla `restablecimientos_password`, estados `rechazado` en los CHECK de `usuarios.estado` y `empleados.estado_laboral`, índice único de CURP normal). Las FASES 3 y 4 **no agregaron migraciones**: su modelo ya existía desde la FASE 1. `infra/db.sql` queda sincronizado con las migraciones (incluye la deriva de la FASE 2 y quita la columna `roles.authorities` que ya no existe).
- **Seed:** sobre una base vacía crea sucursal, roles, departamentos, puestos, horarios y equipos; más dos usuarios configurables desde la sección `Seed`: un `rh` y un `gerente` de demostración con fila en `equipos_gerentes` hacia `Equipo Ventas`. Idempotente. En Production las credenciales son obligatorias y el arranque falla si faltan; en Development usan defaults con `LogWarning`. Detalle en la sección 7.
- **Autenticación:** JWT stateless. `sesiones_usuario` funciona como registro de auditoría; `revocada_at` se escribe pero **no invalida el token** (ver la limitación conocida en la sección 9).
- **Autorización en dos capas.** La policy decide *quién* puede llamar al endpoint; `AlcanceService` decide *sobre qué filas* puede operar. Ambas son necesarias y ninguna basta sola.
  - **Capa 1 — policy.** Vocabulario cerrado en `Common.Config.Security.Models.Policies`: `autenticado`, `rh-only`, `rh-or-gerente`, registradas en `SecurityModule`. Todo endpoint protegido nombra una; no quedan `AuthorizeAttribute` con roles en línea. `rh-or-gerente` tiene hoy un único consumidor, `GET /vacation-balances`, y queda para los flujos de aprobación de las FASES 5–6.
  - **Capa 2 — alcance.** `Common.Security.AlcanceService` (registrado `scoped` en `SecurityModule`) con `RestringirEmpleados`, `RestringirEquipos` y `PuedeAccederAsync`. Aplican, en este orden: filtro por `sucursal_id` **siempre**, luego `rh` = todos, `gerente` = equipos que gestiona, resto = solo su registro. La sucursal siempre proviene del token firmado, nunca del cuerpo de la petición. Centralizarlo impide que las FASES 3–12 reimplementen la regla u olviden el filtro de sucursal.
- **CORS:** en la versión 1.0.0 el API responde con CORS abierto (`AllowAnyOrigin`); es una **deuda aceptada** (ver §32). Las variables `CORS_ALLOWED_ORIGIN_*` se exigen pero aún no se aplican.
- **Documentación OpenAPI** (Swagger UI) habilitada solo en Development.

## 2. Convenciones generales

- Prefijo de rutas: `/api/v1`.
- Autenticación: header `Authorization: Bearer <token>` (JWT).
- Errores: `application/problem+json` (RFC 7807) mediante `GlobalExceptionHandler`. Códigos: 400 validación de dominio, 401 no autenticado/credenciales inválidas, 403 sin permiso o cuenta no activa, 404 no encontrado, 409 conflicto (duplicados o estado), 429 límite de peticiones.
- Rate limiting: política `Auth` = **10 peticiones por minuto y por IP**, sin cola de espera (`QueueLimit = 0`), aplicada a `POST /auth/login`, `POST /auth/forgot-password`, `POST /auth/reset-password` y `POST /employees/self-register`. El umbral y la ventana son **constantes en `Common/RateLimiting/RateLimitingSetup.cs`**; lo único configurable es `RateLimiting:Enabled` (desactivado en las pruebas, ver la sección 6). Al superarse responde `429` con un `ProblemDetails` —título «Demasiadas solicitudes», detalle «Se excedió el límite de intentos. Intenta de nuevo en un minuto.»— y **sin cabecera `Retry-After`**: el cliente debe esperar el minuto por su cuenta.
- Roles: `rh`, `gerente`, `colaborador`. Alcance en colaboradores: **rh** = todos los del catálogo; **gerente** = solo los de sus equipos (`equipos_gerentes`); **colaborador** = únicamente su propio registro.
- Auditoría: cada acción relevante (login, cambio de contraseña, recuperación, crear/actualizar empleado, activar/rechazar/desactivar cuenta, auto-registro, y toda escritura de configuración de la FASE 3) registra evento en `auditoria` con entidad, registro, sucursal, usuario y datos antes/después.
- Catálogos (FASE 3): lectura para cualquier rol autenticado dentro de su sucursal; escritura solo `rh`. `DELETE` es siempre **baja lógica** (`activo = false`), idempotente, y el nombre dado de baja sigue reservado. En `PATCH`, `null` u omitido significa *no tocar*.
- Los campos de texto se normalizan (recortados); valores vacíos opcionales se guardan como `null`.

## 3. Salud y diagnósticos

### `GET /health`
Consulta de salud del servicio (health checks). Sin autenticación. Respuesta definida por ASP.NET Health Checks.

### `GET /api/v1/diagnostics/ping`
Verificación pública del servicio.
- **Auth:** ninguna.
- **200:** `{ "status": "ok", "utc": "<datetime>" }`

### `GET /api/v1/diagnostics/secure`
Endpoint protegido de ejemplo.
- **Auth:** rol `rh`.
- **200:** `{ "status": "ok", "scope": "rh" }`
- Errores: 401, 403.

### `POST /api/v1/diagnostics/echo`
Valida un DTO y lo devuelve.
- **Auth:** ninguna.
- Request: `{ "message": string }` (1–100 caracteres, obligatorio).
- **200:** `{ "message": "<valor>" }`
- Errores: 400 (validación).

## 4. Autenticación (`/api/v1/auth`)

### `POST /auth/login`
Inicia sesión con número de empleado y contraseña.
- **Auth:** ninguna. **Rate limit:** 10/min por IP.
- Request `LoginRequest`:
  ```json
  { "numeroEmpleado": "123", "password": "123", "dispositivo": "iPhone" }
  ```
  | Campo | Tipo | Reglas |
  |---|---|---|
  | `numeroEmpleado` | string | obligatorio, máx 50 |
  | `password` | string | obligatorio, máx 200 |
  | `dispositivo` | string? | opcional, máx 150 |
- **200** `LoginResponse`:
  ```json
  {
    "accessToken": "<jwt>",
    "expiraAt": "2026-09-18T03:25:00Z",
    "usuario": { "usuarioId": 1, "empleadoId": 2, "sucursalId": 1, "numeroEmpleado": "123",
                 "nombreCompleto": "Recursos Humanos", "rol": "rh", "estado": "activo", "correo": null }
  }
  ```
- Errores: 401 (credenciales incorrectas, mensaje genérico), 403 (cuenta no activa), 429.
- Efectos: registra fila en `sesiones_usuario` (dispositivo, user-agent, IP, expiración) y evento `login` en auditoría.

### `POST /auth/logout`
Marca como revocadas las sesiones activas del usuario autenticado.
- **Auth:** Bearer. **Rol:** cualquiera autenticado (`Policies.Autenticado`).
- **204** sin cuerpo. Errores: 401.
- Efectos: actualiza `sesiones_usuario.revocada_at` de todas las filas del usuario con
  `revocada_at IS NULL`, y registra el evento `logout` en auditoría con el conteo de
  sesiones revocadas.
- **La revocación es informativa, no aplicación.** No hay `OnTokenValidated` ni consulta
  por request: un token ya emitido sigue siendo aceptado hasta su expiración
  (`JwtSettings:ExpirationMinutes`, 60 minutos por defecto). La mitigación por diseño es
  esa ventana corta. Ver «Limitación conocida» en la sección 9.

### `POST /auth/sessions/revoke-all`
RH revoca todas las sesiones activas de un usuario concreto.
- **Auth:** Bearer. **Rol:** `rh` únicamente (`Policies.RhOnly`). Un `gerente` recibe 403.
- Query `?usuarioId={entero}` obligatorio.
- **200** `RevocarSesionesResponse`:
  ```json
  { "usuarioId": 7, "sesionesRevocadas": 3 }
  ```
- Errores: 400 (`usuarioId` ausente o no numérico), 401, 403, 404 (la cuenta no existe en la sucursal del solicitante).
- Efectos: igual que `logout` sobre las sesiones del usuario indicado, más el evento
  `revocar_sesiones` en auditoría con el solicitante y el conteo.
- **También informativa:** los tokens ya emitidos siguen siendo válidos hasta expirar.

### `POST /auth/change-password`
Cambia la contraseña del usuario autenticado (valida la contraseña actual).
- **Auth:** Bearer.
- Request `ChangePasswordRequest`:
  ```json
  { "passwordActual": "123", "passwordNueva": "Nueva123!" }
  ```
  | Campo | Tipo | Reglas |
  |---|---|---|
  | `passwordActual` | string | obligatorio, máx 200 |
  | `passwordNueva` | string | obligatorio, mín 8, máx 200 |
- **204** sin cuerpo. Errores: 400 (contraseña actual incorrecta o política), 401.
- Efectos: evento `cambio_password` en auditoría.

### `POST /auth/forgot-password`
Solicita el envío de un enlace de restablecimiento.
- **Auth:** ninguna. **Rate limit:** 10/min por IP.
- Request `ForgotPasswordRequest`:
  ```json
  { "correo": "colaborador@correo.com" }
  ```
  | Campo | Tipo | Reglas |
  |---|---|---|
  | `correo` | string | obligatorio, formato email, máx 254 |
- **202** siempre que el formato sea válido (no revela si el correo existe). Errores: 400, 429.
- Comportamiento: si existe un usuario **activo** con ese correo, invalida tokens pendientes previos, genera un token de un solo uso (32 bytes aleatorios, Base64Url) y lo envía por correo con la plantilla `RecuperacionPassword.html`. El token se **guarda como hash SHA-256**, expira en `PasswordReset:TokenMinutes` (30 por defecto). Si el correo no existe o el usuario no está activo, no hace nada (permite enumeración nula). Si el envío falla, registra el error en logs y continúa (en Development sin SMTP el correo se loguea). Efectos: evento `solicitud_recuperacion` en auditoría.

### `POST /auth/reset-password`
Restablece la contraseña usando el token enviado por correo.
- **Auth:** ninguna. **Rate limit:** 10/min por IP.
- Request `ResetPasswordRequest`:
  ```json
  { "token": "<token>", "passwordNueva": "Nueva123!" }
  ```
  | Campo | Tipo | Reglas |
  |---|---|---|
  | `token` | string | obligatorio, máx 200 |
  | `passwordNueva` | string | obligatorio, mín 8, máx 200 |
- **204** sin cuerpo. Errores: 400 (token inválido/vencido/usado, o política de contraseña), 429.
- Comportamiento: marca el token como usado (un solo uso) y actualiza el hash de la contraseña. Efectos: evento `restablecimiento_password` en auditoría.

### `GET /auth/me`
Perfil del usuario autenticado.
- **Auth:** Bearer.
- **200** `PerfilUsuario`:
  ```json
  {
    "usuarioId": 1, "empleadoId": 2, "sucursalId": 1,
    "numeroEmpleado": "123", "nombreCompleto": "Recursos Humanos",
    "rol": "rh", "estado": "activo", "correo": null
  }
  ```
  | Campo | Tipo |
  |---|---|
  | `usuarioId` | int |
  | `empleadoId` | int |
  | `sucursalId` | int |
  | `numeroEmpleado` | string |
  | `nombreCompleto` | string |
  | `rol` | string |
  | `estado` | string |
  | `correo` | string? |
- Errores: 401.

### `GET /auth/sessions`
Historial de inicios de sesión del usuario autenticado (auditoría, últimos 50).
- **Auth:** Bearer.
- **200** `SesionResponse[]`:
  ```json
  [{
    "id": 1, "dispositivo": "iPhone", "userAgent": "Mozilla/5.0 ...", "ip": "127.0.0.1",
    "expiraAt": "2026-09-18T03:25:00Z", "revocadaAt": null, "createdAt": "2026-09-18T02:25:00Z"
  }]
  ```
  | Campo | Tipo |
  |---|---|
  | `id` | int |
  | `dispositivo` | string? |
  | `userAgent` | string? |
  | `ip` | string? |
  | `expiraAt` | datetime |
  | `revocadaAt` | datetime? (sin uso hoy) |
  | `createdAt` | datetime |
- Errores: 401.

## 5. Colaboradores (`/api/v1/employees`)

Todas las rutas exigen Bearer salvo `self-register`. La sucursal siempre se resuelve del usuario autenticado, nunca del cliente. El alcance por rol lo aplica `AlcanceService` (ver la sección 1), de modo que la lista y el detalle devuelven exactamente las mismas filas que un `gerente` puede ver en su expediente.

### `GET /api/v1/employees`
Lista paginada de colaboradores según el alcance del rol.
- **Auth:** cualquier rol autenticado (alcance: rh = todos, gerente = sus equipos, colaborador = solo él mismo).
- Query params: `page` (default 1), `pageSize` (default 20), `estadoLaboral` (`pendiente | activo | baja | rechazado`), `departamentoId`, `equipoId`, `busqueda` (número de empleado o nombre, contiene).
- **200** `PagedResult<EmpleadoResumen>`:
  ```json
  {
    "items": [{
      "id": 2, "numeroEmpleado": "123", "nombreCompleto": "Recursos Humanos",
      "telefono": "0000000000", "correo": null, "estadoLaboral": "activo",
      "estadoUsuario": "activo", "rol": "rh", "puestoId": null, "departamentoId": null,
      "equipoId": null, "horarioId": null, "fechaIngreso": null, "fechaBaja": null
    }],
    "page": 1, "pageSize": 20, "total": 1, "totalPages": 1
  }
  ```
  | Campo (resumen) | Tipo |
  |---|---|
  | `id`, `numeroEmpleado`, `nombreCompleto`, `telefono` | string/int |
  | `correo` | string? |
  | `estadoLaboral` | string |
  | `estadoUsuario` | string? |
  | `rol` | string? |
  | `puestoId`, `departamentoId`, `equipoId`, `horarioId` | int? |
  | `fechaIngreso`, `fechaBaja` | date? |
- Errores: 401.

### `GET /api/v1/employees/validate-number?numeroEmpleado={valor}`
Indica si un número de empleado puede usarse en un alta. **El número lo genera el cliente**: el backend solo valida. Es una consulta previa, no reserva el número; `POST /employees` puede responder 409 si otro alta se adelanta.
- **Auth:** rol `rh`.
- **200** `NumeroEmpleadoValidacion`, siempre (la indisponibilidad no es un error HTTP):
  ```json
  { "numeroEmpleado": "E-0013", "disponible": false, "motivo": "Ya existe un colaborador con ese número de empleado." }
  ```
- Reglas, en este orden, sobre el valor recortado: vacío o ausente → «El número de empleado es obligatorio.»; más de 50 caracteres → «El número de empleado no puede superar 50 caracteres.»; ya existe en la sucursal → «Ya existe un colaborador con ese número de empleado.». Si no hay motivo, `disponible` es `true`.
- Errores: 401, 403.

### `GET /api/v1/employees/{id}` (`{id}` = entero)
Expediente detallado de un colaborador.
- **Auth:** cualquier rol (dentro de su alcance; rh siempre).
- **200** `EmpleadoDetalle`: campos del resumen más `sucursalId`, `usuarioId?`, `fechaNacimiento?`, `curp?`, `rfc?`, `numeroImss?`, `direccion?`, `ciudad?`, `estado?`, `codigoPostal?`, nombres de `puesto?`, `departamento?`, `equipo?`, `horario?`, `motivoBaja?`, `createdAt`, `updatedAt`.
- Errores: 401, 403 (fuera de alcance), 404.

### `POST /api/v1/employees`
Alta de colaborador con su cuenta de acceso. Crea el empleado (estado laboral `activo`) y el usuario (estado `activo`) en transacción.
- **Auth:** rol `rh`.
- Request `CreateEmployeeRequest` (opcionales: `curp`, `rfc`, `numeroImss`, `direccion`, `ciudad`, `estado`, `codigoPostal`, `fechaNacimiento`, `fechaIngreso`, `puestoId`, `departamentoId`, `equipoId`, `horarioId`):
  ```json
  {
    "numeroEmpleado": "205", "nombreCompleto": "Ana García", "telefono": "5544332211",
    "correo": "ana@correo.com", "password": "Inicial123!", "rol": "colaborador"
  }
  ```
  | Campo | Tipo | Reglas |
  |---|---|---|
  | `numeroEmpleado` | string | obligatorio, máx 50 |
  | `nombreCompleto` | string | obligatorio, máx 200 |
  | `telefono` | string | obligatorio, máx 30 |
  | `correo` | string? | formato email, máx 254 |
  | `curp` | string? | máx 18 |
  | `rfc` | string? | máx 13 |
  | `numeroImss` | string? | máx 20 |
  | `direccion` | string? | máx 200 |
  | `ciudad` | string? | máx 100 |
  | `estado` | string? | máx 100 |
  | `codigoPostal` | string? | máx 10 |
  | `fechaNacimiento` | date? | — |
  | `fechaIngreso` | date? | — |
  | `puestoId`, `departamentoId`, `equipoId`, `horarioId` | int? | — |
  | `password` | string | obligatorio, mín 8, máx 200 |
  | `rol` | string? | `colaborador` por defecto; debe ser `colaborador\|gerente\|rh` |
- **201** `EmpleadoDetalle` (con `Location: /api/v1/employees/{id}`). Errores: 400 (rol inválido/no configurado, política), 401, 403, 409 (número, CURP, teléfono o correo duplicados), 404 (si el rol configurado no existe).
- Efectos: evento `crear_empleado` en auditoría.

### `PATCH /api/v1/employees/{id}`
Actualiza el expediente y, opcionalmente, la contraseña o el rol de la cuenta.
- **Auth:** rol `rh`.
- Request `UpdateEmployeeRequest` (todos opcionales; `password` mín 8 si se envía; `rol` igual que en alta; `correo`/`curp`/`rfc`/`numeroImss` se normalizan, `null` no altera, cadena vacía limpia):
  ```json
  { "telefono": "5522223344", "correo": "nuevo@correo.com" }
  ```
- **200** `EmpleadoDetalle`. Errores: 400, 401, 403, 404, 409.
- `puestoId`, `departamentoId`, `equipoId`, `horarioId` y las fechas: `null` **no altera** la asignación (decisión D5, sección 15). Hoy no existe forma de *desasignar* uno de estos valores.
- Referencias (también en `POST`): un catálogo inexistente o de otra sucursal → **404**; uno existente pero dado de baja → **400** con el campo como clave. Ya no producen 500.
- Efectos: evento `actualizar_empleado` en auditoría (la contraseña nunca se guarda en el resumen; se registra como oculta o cambiada).

### `POST /api/v1/employees/{id}/activate`
Reactiva la cuenta (`usuarios.estado = activo`) y el estado laboral (`activo`), limpiando `fechaBaja`/`motivoBaja`. También es la vía de aprobación del auto-registro.
- **Auth:** rol `rh`.
- **200** `EmpleadoDetalle`. Errores: 401, 403, 404.
- Efectos: evento `activar_cuenta` en auditoría.

### `POST /api/v1/employees/{id}/reject`
Rechaza un auto-registro (estado laboral y de cuenta `rechazado`). Solo aplica a colaboradores en estado `pendiente` (o ya `rechazado`).
- **Auth:** rol `rh`.
- **200** `EmpleadoDetalle`. Errores: 401, 403, 404, 409 (no está pendiente).
- Efectos: evento `rechazar_cuenta` en auditoría.

### `POST /api/v1/employees/{id}/deactivate`
Da de baja: cuenta `inactivo`, estado laboral `baja`, fecha de baja = hoy, motivo obligatorio.
- **Auth:** rol `rh`.
- Request `DeactivateRequest`:
  ```json
  { "motivo": "Renuncia voluntaria" }
  ```
  | Campo | Tipo | Reglas |
  |---|---|---|
  | `motivo` | string | obligatorio, máx 255 |
- **200** `EmpleadoDetalle`. Errores: 400, 401, 403, 404.
- Efectos: evento `desactivar_cuenta` en auditoría (incluye motivo y fecha de baja).

### `POST /api/v1/employees/self-register`
Auto-registro público de colaborador. Crea el empleado (`pendiente`) y el usuario (`pendiente`) sobre la **primera sucursal activa**; el rol siempre es `colaborador`; la cuenta no puede iniciar sesión hasta que RH la active.
- **Auth:** ninguna. **Rate limit:** 10/min por IP.
- Request `SelfRegisterRequest` (opcionales: `curp`, `direccion`, `ciudad`, `estado`, `codigoPostal`, `fechaNacimiento`):
  ```json
  {
    "numeroEmpleado": "300", "nombreCompleto": "Luis Pérez", "telefono": "5512345678",
    "correo": "luis@correo.com", "password": "Registro123!"
  }
  ```
  | Campo | Tipo | Reglas |
  |---|---|---|
  | `numeroEmpleado` | string | obligatorio, máx 50 |
  | `nombreCompleto` | string | obligatorio, máx 200 |
  | `telefono` | string | obligatorio, máx 30 |
  | `correo` | string | obligatorio, formato email, máx 254 |
  | `curp` | string? | máx 18 |
  | `direccion` | string? | máx 200 |
  | `ciudad` | string? | máx 100 |
  | `estado` | string? | máx 100 |
  | `codigoPostal` | string? | máx 10 |
  | `fechaNacimiento` | date? | — |
  | `password` | string | obligatorio, mín 8, máx 200 |
- **201** `EmpleadoDetalle`. Errores: 400, 409 (duplicados o sin sucursal activa), 429.
- Efectos: evento `auto_registro` en auditoría (sin usuario autenticado).

## 6. Pruebas

```bash
dotnet test cotla-api.slnx   # requiere Docker: levanta postgres:17-alpine
dotnet format cotla-api.slnx --verify-no-changes
```

**277 casos en verde** (los `[Theory]` cuentan un caso por fila). La tabla lista los archivos; los recuentos anteriores a la FASE 5 no se recalcularon:

| Archivo | Casos | Cubre |
|---|---|---|
| `LeaveRequestsTests.cs` | 11 | Solicitudes de ausencia: días naturales, saldo, traslape, aprobar con descuento y movimiento, revalidación, rechazo, cancelación, incapacidad con documento, alcance |
| `IncidentsEndpointsTests.cs` | 4 | Detección idempotente, fuera de zona, jornada en curso o sin marcas, resolver/descartar y alcance |
| `AttendanceCorrectionsTests.cs` | 4 | Corrección con original intacta, una pendiente por marcación, rechazo con motivo, validaciones y permisos |
| `AttendanceEndpointsTests.cs` | 37 | Marcaciones con selfie y GPS, zona, secuencia, idempotencia, validación de la foto, retardo, fecha local, alcance de consulta y descarga de la foto |
| `SchedulesEndpointsTests.cs` | 25 | CRUD de horarios, reglas de comida, tolerancia por defecto (D2) |
| `DepartmentsEndpointsTests.cs` | 20 | CRUD, duplicados, baja lógica, permisos |
| `VacationBalancesTests.cs` | 18 | Saldo inicial, ajustes con signo, alcance |
| `TeamsEndpointsTests.cs` | 17 | Equipos, gerentes y miembros |
| `EmployeesEndpointsTests.cs` | 14 | Alcance por rol, alta, auto-registro, activación, hash de contraseña |
| `UsersEndpointsTests.cs` | 11 | Cuentas `/users`: listado, cambio de rol, desactivar y revocar sesiones, reseteo de contraseña, aislamiento entre sucursales, último RH |
| `EmployeesRhGuardTests.cs` | 5 | Guardas de último RH y autodegradación desde `/employees` |
| `EmployeeNumberValidationTests.cs` | 8 | `validate-number`: libre, existente, espacios, vacío, longitud, tras un alta, alcance por rol, sin token |
| `ErroresEnEspanolTests.cs` | 6 | Errores en español (campo largo, 404, 405, JSON mal formado, 401) y guarda que exige `ErrorMessage` en todo atributo de validación de un contrato |
| `PositionsEndpointsTests.cs` | 14 | CRUD de puestos |
| `StoreEndpointsTests.cs` | 13 | Tienda y zona |
| `EmployeeReferenceValidationTests.cs` | 13 | Referencias de catálogo 404/400 |
| `AuthEndpointsTests.cs` | 12 | Login, credenciales inválidas, cuentas no activas, `logout`, `revoke-all`, `change-password` |
| `PasswordResetTests.cs` | 7 | Recuperación completa, token de un solo uso, invalidación del anterior, enumeración nula |
| `DiagnosticsEndpointsTests.cs` | 6 | Ping, `secure`, `echo` |
| `HealthTests.cs` | 1 | `/health` |

### Decisiones de la infraestructura de pruebas

- **Una sola base compartida y pruebas en serie** (`PostgresCollection`). El fixture
  levanta un `postgres:17-alpine` real con Testcontainers y aplica las migraciones;
  no hay base en memoria ni emuladores.
- **El limitador de intentos se desactiva en el host de pruebas**
  (`RateLimiting:Enabled=false`). El bucket son 10 por minuto y por IP, y desde el
  servidor todas las peticiones comparten IP: una suite con 40 casos que autentican
  lo superaría siempre y fallaría por motivos ajenos a lo que verifica. El
  comportamiento real se comprueba contra `docker compose`.
- **El correo se captura, no se envía.** `RecordingEmailSender` sustituye a
  `IEmailSender` y guarda los mensajes emitidos. El token de recuperación solo
  existe en texto plano dentro del mensaje, así que leerlo de la base es imposible;
  la prueba lo toma del correo capturado. La lógica real —generación, hasheo
  SHA-256, invalidación del token previo y un solo uso— sí se ejercita; lo único
  que se finge es el transporte SMTP.
- **Las pruebas no tocan los datos de semilla.** Toda prueba que modifica estado
  crea una cuenta desechable con `TestAccounts` (número, teléfono y correo únicos) y
  emite el token con `TestTokens.CreateFor`, que carga el `Usuario`, `Empleado` y
  `Rol` reales de la base.
- **Nada de totales absolutos.** El listado de colaboradores está paginado y la base
  es compartida, así que las aserciones comprueban contención o conjunto exacto de
  identificadores, nunca un número global de filas.

- **Sin MinIO en la suite.** `InMemoryFileStorage` sustituye a `IFileStorage`, igual que
  el capturador de correo: la validación de la foto, la clave, la fila de `archivos` y la
  descarga con alcance son las reales. El transporte S3 se verifica contra Compose.
- **Reloj controlable.** `RelojControlable` sustituye a `TimeProvider` para fijar la hora en
  las pruebas de retardo y de cambio de día. Se fija en el pasado porque la vigencia del
  JWT usa la hora real.
- **`TestAccounts` toma la sucursal de menor id y el rol de esa sucursal.** Antes tomaba
  «la primera» sin orden: cuando `StoreEndpointsTests` actualizaba la sucursal sembrada,
  PostgreSQL devolvía la sucursal ajena que crea `EmployeeReferenceValidationTests` y el
  alta chocaba con el FK compuesto de `usuarios` (12 pruebas de equipos y saldos fallaban
  según el orden). Corregido al cerrar la FASE 4.

### Qué no cubren las pruebas

- El **limitador de intentos**, desactivado arriba.
- La **entrega de correo** contra un servidor SMTP real.
- El **avance entre sucursales**: solo hay una sucursal en la base.
- El **almacenamiento real** (MinIO) y la zona horaria del contenedor: se verifican contra Compose.
- La **concurrencia** de dos marcaciones simultáneas (candado por colaborador y choque de
  UUID): la suite corre en serie.

### Verificación contra la pila real

Con `docker compose up --build -d` en `Production` y peticiones reales: `/health`
`Healthy`; login correcto en los tres roles y `401` con contraseña incorrecta;
preflight CORS permitido `204` con `Access-Control-Allow-Origin` y denegado sin él; el
alcance por rol coincide con lo que verifique la suite; el ciclo completo
auto-registro → `pendiente` → activación por RH → login; el restablecimiento de
contraseña con el hash SHA-256 persistido; y el umbral del limitador medido en
**10 permisos por minuto y por IP**, con `429` a partir del siguiente.

La entrega del enlace de recuperación no se recorre por correo contra la pila real:
sin `Smtp:*` configurado el token se descarta. Para ejercitar `reset-password` se
insertó directamente un token cuyo SHA-256 se calculó por adelantado, lo que prueba
el hasheo, la validación y el consumo de un solo uso contra la base real. Esa parte
del flujo queda cubierta solo por las pruebas, con el capturador.

## 7. Configuración relevante (`appsettings.json`)

```jsonc
"JwtSettings": {
  "Issuer": "cotla-api",
  "Audience": "cotla",
  "ExpirationMinutes": 60
},
"PasswordReset": {
  "BaseUrl": "http://localhost:3000/reset-password",
  "TokenMinutes": 30
},
"Smtp": {
  "Host": "", "Port": 587, "User": "", "Password": "",
  "From": "", "FromName": "RH Cotla", "EnableSsl": true
},
"RateLimiting": { "Enabled": true }
```

- `Storage.*` y `Asistencia.*`: ver las secciones 16 y 17.
- `RateLimiting.Enabled`: es la **única** opción del limitador que se puede
  configurar (`RateLimiting__Enabled`). El umbral de 10 peticiones por minuto, la
  ventana de un minuto y la ausencia de cola de espera son constantes de
  `Common/RateLimiting/RateLimitingSetup.cs`, no claves de configuración; no hay
  forma de cambiarlas por `appsettings.json` ni por variable de entorno.

- `JwtSettings.ExpirationMinutes`: vigencia del token y de la sesión registrada. Por seguridad (cuenta inactiva/rechazada no bloquea tokens existentes) el valor por defecto es **60 minutos**; se puede ajustar por variable de entorno (`JwtSettings__ExpirationMinutes`).
- `Smtp.*`: configuración genérica de correo (puerto, credenciales, remitente, SSL). En Development sin host configurado, el envío de recuperación se simula registrándolo en el log. Variables de entorno: `Smtp__Host`, `Smtp__Port`, `Smtp__User`, `Smtp__Password`, `Smtp__From`, `Smtp__FromName`, `Smtp__EnableSsl`.
- `PasswordReset.BaseUrl`: URL base del enlace (se le añade `?token=...`). `PasswordReset.TokenMinutes`: vigencia del token (30).
- El ajuste `ExpirationMinutes` corto es la mitigación por diseño ante la ausencia de revocación de sesiones; la limitación queda documentada de forma explícita.

### `Seed` (usuarios de siembra)

Sección nueva, obligatoria en **Production**. Si falta cualquiera de sus variables el
servicio se niega a arrancar. Variables de entorno: `Seed__RhNumeroEmpleado`,
`Seed__RhPassword`, `Seed__GerenteNumeroEmpleado`, `Seed__GerentePassword`,
`Seed__GerenteNombre`, `Seed__SucursalLatitud`, `Seed__SucursalLongitud`,
`Seed__SucursalNombre`, `Seed__SucursalRazonSocial`, `Seed__SucursalDireccion`,
`Seed__SucursalCiudad`, `Seed__CrearColaboradoresDemo`.

- En Development, los valores ausentes caen a un default de desarrollo y se emite un
  `LogWarning`; el servicio arranca igual.
- Las contraseñas configuradas se validan contra `PasswordPolicy` (mín. 8) **antes** de
  hashear. Un valor corto aborta el arranque con un mensaje que identifica al usuario.
- Se crea un usuario `rh` y un usuario `gerente` de demostración. El gerente recibe una
  fila en `equipos_gerentes` hacia el equipo `Equipo Ventas`: sin ella el alcance del rol
  `gerente` devolvería conjunto vacío siempre.
- `Seed__SucursalLatitud` / `Seed__SucursalLongitud` sustituyen al `(0, 0)` histórico. Si
  se dejan vacías se usa `0` con aviso en el log, y el geocerca de FASE 4 no será funcional.
- `Seed__CrearColaboradoresDemo` controla los cinco colaboradores de demostración que
  hizo posible el alcance del rol `gerente` en las pruebas. En `false` no se crean y el
  aviso correspondiente queda registrado en el log; conviene `false` en producción.

## 8. CORS

- **Estado real en 1.0.0:** `Program.cs` registra la política con `AllowAnyOrigin`,
  `AllowAnyHeader` y `AllowAnyMethod` en todos los ambientes. `Cors:AllowedOrigins`
  (`Cors__AllowedOrigins__0`, `__1`) **no se lee**: el compose exige `CORS_ALLOWED_ORIGIN_0`
  solo para conservar el contrato de configuración.
- Es una **deuda aceptada** por el usuario (§32). La API usa JWT en la cabecera
  `Authorization` (no cookies), por lo que CORS abierto no permite robar sesiones por sí
  solo; sí deja de ser una defensa contra orígenes no previstos.
- Para cerrarlo en una versión futura: leer `Cors:AllowedOrigins` con `WithOrigins(...)`
  y fallar al arrancar en Production si la lista queda vacía.

## 9. Limitación conocida: revocación de sesiones

`logout` y `sessions/revoke-all` escriben `revocada_at` y evento de auditoría, pero
**no invalidan nada del lado del servidor**: no existe `OnTokenValidated` ni consulta a
`sesiones_usuario` durante la validación del token. Un JWT ya emitido se sigue aceptando
hasta `JwtSettings:ExpirationMinutes`.

Causa: el token emite un `jti` (`JwtService.GenerateToken`) pero `SesionUsuario` no lo
persiste, así que no hay forma de identificar la sesión concreta de una petición.

Decisión tomada: la revocación queda **informativa** de forma deliberada, coherente con
un diseño JWT stateless, y la mitigación es la ventana de expiración corta. Si en el
futuro se exige revocación real por dispositivo, el camino es añadir una columna
`token_id` a `sesiones_usuario` y validar ese `jti` en `OnTokenValidated`.

## 10. Flujos relevantes

- **Login → sesión:** `login` valida credenciales con BCrypt, exige cuenta `activo`, registra la sesión (auditoría) y emite el JWT (claims: `sub`, `jti`, rol, `numero_empleado`, `empleado_id`, `sucursal_id`, `rol`). El cliente conserva el token; `logout` marca `revocada_at` en las sesiones activas del usuario (204), sin invalidar el token en el servidor.
- **RH revoca sesiones de un usuario:** `POST /auth/sessions/revoke-all?usuarioId={id}` (solo `rh`) marca `revocada_at` en todas las sesiones activas de ese usuario y registra `revocar_sesiones` en auditoría.
- **Recuperación de contraseña:** `forgot-password` (202 incondicional) → correo con enlace `?token=...` → `reset-password` consume el token (único uso, expiración 30 min, hash SHA-256 en BD). También aplica en Development sin SMTP (el correo se loguea).
- **Cambio de contraseña:** valida la contraseña actual y la política (mín. 8) y re-hashea con BCrypt.
- **Auto-registro:** `self-register` (público, 10/min por IP) crea el expediente y la cuenta en `pendiente`; RH la **activa** (`activate`) o la **rechaza** (`reject`, estado `rechazado`). Una cuenta `pendiente`, `rechazada` o `inactiva` no puede iniciar sesión (403).
- **Alcance por rol:** las consultas de colaboradores se filtran en el backend según `rh` (todos), `gerente` (equipos asignados vía `equipos_gerentes`) o `colaborador` (solo su registro). La sucursal siempre proviene del token.
- **Bajas:** `deactivate` exige motivo, registra fecha de baja y deja la cuenta `inactivo`; `activate` reactiva y limpia la baja.

## 11. Catálogos: departamentos, puestos y horarios

Mismo esqueleto en los tres recursos. Lectura con `autenticado` (cualquier rol, siempre
dentro de la sucursal del token); escritura con `rh-only`.

| Ruta | Respuesta |
|---|---|
| `GET /api/v1/{recurso}` | `PagedResult<…Resumen>`. Query: `page`, `pageSize`, `activo` (por defecto `true`), `busqueda` (contiene, sobre el nombre) |
| `GET /api/v1/{recurso}/{id}` | `…Detalle`. 404 si no existe en la sucursal |
| `POST /api/v1/{recurso}` | 201 + `Location`. 400 validación, 409 nombre duplicado |
| `PATCH /api/v1/{recurso}/{id}` | 200. Omitido o `null` no altera |
| `DELETE /api/v1/{recurso}/{id}` | 204. Baja lógica, idempotente |

Recursos: `departments`, `positions`, `schedules`.

- **Nombre:** obligatorio, 3–100 caracteres, único por sucursal **incluyendo los dados de
  baja** (el índice `(sucursal_id, nombre)` no filtra por `activo`). `descripcion` máx. 250.
- **Departamentos:** el resumen trae `equiposActivos` y `empleadosActivos`; el detalle trae
  además la lista `equipos` (`id`, `nombre`, `activo`).
- **Puestos:** resumen y detalle traen `empleadosActivos`. No existe filtro por
  departamento: la tabla `puestos` no tiene esa columna.
- **Horarios:** `horaEntrada`, `horaSalida` (obligatorias, `HH:mm:ss`),
  `horaSalidaComida?`, `horaRegresoComida?`, `toleranciaEntrada?` (0–240 minutos) y
  `empleadosAsignados`.
  - Sin `toleranciaEntrada` en el `POST`, se copia la de la sucursal (decisión D2). El
    horario es la única fuente que leerá el checador.
  - Reglas replicadas de los `CHECK` de la base, con 400 por campo: `entrada < salida`; la
    comida va completa o no va; si va, `entrada < salidaComida < regresoComida < salida`.
    En `PATCH` se valida el **resultado combinado** con lo que ya tenía el horario.
- Los conteos se obtienen con una consulta agregada por página, nunca una por fila.

## 12. Equipos y gerentes (`/api/v1/teams`)

Mismo contrato que los catálogos de la sección 11, más:

- `GET /teams` acepta además `departamentoId`. Resumen: `id`, `nombre`, `descripcion`,
  `departamentoId`, `departamento` (nombre), `activo`, `empleadosActivos`, `gerentes`
  (conteo). Detalle: lo mismo con la lista `gerentes` y `createdAt`/`updatedAt`.
- `departamentoId` en `POST`/`PATCH` se valida con `ReferenciaCatalogoService`: 404 si no
  existe en la sucursal, 400 si está dado de baja.
- **Dar de baja un equipo no desactiva a sus miembros ni corta el alcance de sus
  gerentes**: `equipos_gerentes` no mira `activo`. Lo único que cambia es que el equipo
  deja de aceptar nuevas asignaciones de colaboradores (400 en `PATCH /employees`).

### Gerentes

| Ruta | Policy | Respuesta |
|---|---|---|
| `GET /teams/{id}/managers` | `autenticado` | `GerenteEquipo[]`: `id` (empleado), `numeroEmpleado`, `nombreCompleto`, `asignadoAt` |
| `POST /teams/{id}/managers` | `rh-only` | Cuerpo `{ "empleadoId": 7 }`. **201** con la lista actualizada |
| `DELETE /teams/{id}/managers/{empleadoId}` | `rh-only` | 204. 404 si no estaba asignado |

Al asignar: el colaborador debe existir en la sucursal (404), tener `estado_laboral =
activo` y una cuenta con rol `gerente` (400 con clave `empleadoId` nombrando el requisito).
Reasignar el mismo gerente → **409**. Un equipo admite **varios gerentes** y puede quedarse
sin ninguno; en ese caso ningún `gerente` ve a sus miembros. El efecto es inmediato sobre
el alcance: el siguiente `GET /employees` del gerente ya incluye o excluye al equipo.
Auditoría: `asignar_gerente_equipo` y `retirar_gerente_equipo` sobre `equipos_gerentes`.

### Miembros

`GET /teams/{id}/members` (`autenticado`) devuelve `PagedResult<MiembroEquipo>` (`id`,
`numeroEmpleado`, `nombreCompleto`, `puestoId`, `puesto`, `estadoLaboral`), filtrado por
`AlcanceService`: RH ve a todos, el gerente solo si gestiona ese equipo (si no, lista
vacía), el colaborador solo a sí mismo. Query opcional `estadoLaboral`.

## 13. Tienda y zona (`/api/v1/store`)

Recurso **singular**: la tienda es la sucursal del token. No existen `POST` ni `DELETE`
(§27 excluye múltiples sucursales operativas).

- `GET /store` — **cualquier rol autenticado**. Es el insumo del geofencing de la FASE 4:
  el móvil lee `latitud`, `longitud` y `radioPermitidoMetros` de su propia sucursal sin
  ser RH. Respuesta `TiendaResponse`: `id`, `nombre`, `razonSocial`, `rfc`, `telefono`,
  `correo`, `direccion`, `ciudad`, `estado`, `codigoPostal`, `latitud`, `longitud`,
  `radioPermitidoMetros`, `toleranciaEntrada`, `activa`, `updatedAt`.
- `PATCH /store` — `rh-only`. Todos los campos anteriores salvo `id`, `activa` y
  `updatedAt`. `null` u omitido no altera. En los opcionales (`razonSocial`, `rfc`,
  `telefono`, `correo`) la cadena vacía limpia el valor; en los obligatorios (`nombre`,
  `direccion`, `ciudad`, `estado`, `codigoPostal`) se rechaza con 400.
- Validaciones (400 por campo): `latitud` ∈ [-90, 90], `longitud` ∈ [-180, 180],
  `radioPermitidoMetros > 0`, `toleranciaEntrada >= 0`, `correo` con formato válido y
  `rfc` de **12 o 13** caracteres (persona moral o física; se guarda en mayúsculas). RFC
  repetido en otra sucursal → 409.
- Coordenadas redondeadas a 6 decimales, que es la precisión de la columna
  (`numeric(9,6)`), para que la respuesta coincida con lo guardado.
- `toleranciaEntrada` de la tienda es solo el **valor por defecto** de los horarios nuevos;
  cambiarla no modifica los horarios existentes (decisión D2).
- Auditoría: `actualizar_tienda` sobre `sucursales` con datos antes y después.
- **Nota para FASE 4:** la base solo exige `radio > 0`, así que un radio de 1 m es válido
  aunque en la práctica marque casi todo como *fuera de zona*. Conviene acordar un mínimo
  razonable antes de liberar el checador.

## 14. Saldos de vacaciones (`/api/v1/vacation-balances`)

Solo saldo inicial y ajustes manuales. El devengo automático está fuera de fase
(decisión D1) y los consumos (`uso`) los registrará la FASE 6 al aprobar solicitudes.

| Ruta | Policy | Alcance / respuesta |
|---|---|---|
| `GET /vacation-balances` | `rh-or-gerente` | `PagedResult<SaldoVacacionesResumen>`. RH: toda la sucursal; gerente: sus equipos. Un colaborador recibe 403 y consulta su saldo con la ruta de detalle |
| `GET /vacation-balances/{empleadoId}` | `autenticado` | `SaldoVacacionesDetalle`. 403 fuera de alcance; 404 si el colaborador no existe o no tiene saldo |
| `GET /vacation-balances/{empleadoId}/movements` | `autenticado` | `PagedResult<MovimientoVacacionesResponse>`, del más reciente al más antiguo. Mismo alcance |
| `PUT /vacation-balances/{empleadoId}` | `rh-only` | Define o corrige el saldo inicial. 200 con el detalle |
| `POST /vacation-balances/{empleadoId}/adjustments` | `rh-only` | Ajuste manual con signo. 200 con el detalle |

- **Listado.** Filtros: `departamentoId`, `equipoId`, `busqueda` (número o nombre),
  `soloConSaldo` (`true` solo con saldo, `false` solo **sin** saldo —para encontrar a
  quién falta configurar—, omitido todos). Cada fila trae `empleadoId`, `numeroEmpleado`,
  `nombreCompleto`, `estadoLaboral`, `departamentoId`, `equipoId`, `tieneSaldo` y, si lo
  tiene, `diasIniciales`, `diasGenerados`, `diasUtilizados`, `diasDisponibles`, `updatedAt`.
- **Detalle.** `id`, `empleadoId`, `numeroEmpleado`, `nombreCompleto`, `diasIniciales`,
  `diasGenerados`, `diasUtilizados`, `diasDisponibles`, `updatedAt`. `diasDisponibles` es
  la columna calculada `iniciales + generados − utilizados`: se expone, nunca se escribe.
- **Movimiento.** `id`, `tipo` (`saldo_inicial | devengo | uso | ajuste`), `dias` (con
  signo), `descripcion`, `solicitudAusenciaId`, `creadoPorUsuarioId`, `createdAt`.

### Saldo inicial — `PUT`

Cuerpo `{ "diasIniciales": 12, "descripcion": "opcional, máx. 255" }`, `diasIniciales`
entre 0 y 365.

- Sin saldo previo: crea la fila y un movimiento `saldo_inicial` por `diasIniciales`. Con
  `diasIniciales = 0` crea la fila **sin** movimiento (la base prohíbe movimientos de 0).
- Con saldo previo: registra un movimiento `ajuste` por la **diferencia**, nunca por el
  valor absoluto, de modo que la suma del historial siempre cuadra con el saldo. Si la
  diferencia es 0 no se escribe nada.
- 400 (`diasIniciales`) si dejaría `iniciales + generados < utilizados`. 404 si el
  colaborador no existe en la sucursal.

### Ajuste manual — `POST /adjustments` (decisión D4)

Cuerpo `{ "dias": -2, "descripcion": "Justificación obligatoria" }`, `dias` entre -365 y
365 y distinto de 0; `descripcion` 3–255 caracteres.

- `dias > 0` suma a `diasGenerados`. `dias < 0` resta a `diasIniciales`.
  `diasUtilizados` nunca se toca desde aquí.
- 400 (`dias`) si es 0, si dejaría `diasIniciales < 0` o si dejaría el saldo por debajo de
  lo utilizado. **409** si el colaborador aún no tiene saldo: primero se define el inicial.
- Saldo y movimiento se escriben en la misma transacción; el movimiento guarda
  `creadoPorUsuarioId`. Auditoría: `definir_saldo_vacaciones` y `ajustar_saldo_vacaciones`
  sobre `saldos_vacaciones`.

## 15. Decisiones de la FASE 3

Tomadas con el usuario el 25 de septiembre de 2026; no se reabren sin actualizar
`docs/especificaciones.md` §31.

| # | Decisión |
|---|---|
| D1 | **Devengo de vacaciones fuera de fase.** Solo saldo inicial y ajuste manual. El motor de devengo y su política (anual al aniversario o proporcional mensual) esperan confirmación del cliente. |
| D2 | **La tolerancia manda en el horario.** `horarios.tolerancia_entrada` es la fuente; la de la sucursal es el valor por defecto que se copia al crear un horario. |
| D3 | **Lectura de catálogos abierta, escritura restringida.** Departamentos, puestos, horarios, equipos y tienda se leen con cualquier rol; se escriben solo con `rh`. Los saldos sí se restringen por alcance. |
| D4 | **Ajuste de saldo con signo.** Positivo suma a generados; negativo resta a iniciales sin dejarlos negativos. |
| D5 | **`null` en `PATCH /employees` no desasigna.** Corrige el código para cumplir lo que ya prometía esta documentación. Consecuencia aceptada: no hay forma de limpiar un FK o una fecha del expediente; si el panel lo necesita hará falta una acción dedicada. |

Otras resoluciones de la fase:

- **404 para cualquier referencia que no resuelve en la sucursal** (inexistente o de otra
  sucursal), sin distinguir los dos casos para no revelar catálogos ajenos; **400** solo
  para «existe pero está dado de baja». Aplica también a `PUT /vacation-balances`, donde
  el plan pedía 409.
- **El listado de saldos es `rh-or-gerente`.** El plan pedía esa policy y a la vez que el
  colaborador viera «solo el suyo» en el listado; se mantuvo la policy y el colaborador
  usa la ruta de detalle.
- **La guarda del saldo inicial replica el `CHECK` exacto** (`utilizados <= iniciales +
  generados`) en vez de la versión más estricta del plan (`iniciales >= utilizados`), que
  habría rechazado casos válidos con días generados.
- **Límite de 365 días** en el saldo inicial y en cada ajuste: no es regla de negocio, es
  un tope defensivo para que la suma no desborde la columna entera.
- **La siembra no asigna departamento** a los colaboradores de demostración (sí equipo y
  puesto), así que los conteos por departamento empiezan en 0 en una base recién sembrada.

## 16. Almacenamiento de archivos

`Common/Storage/` define `IFileStorage` (`PutAsync`, `GetAsync`, `DeleteAsync`,
`EnsureBucketAsync`, `PingAsync`) y su implementación `MinioFileStorage` con el SDK `Minio`
(decisión D9). Cambiar de proveedor S3 en producción es cambiar esta clase, no los módulos.

| Variable | Valor en Compose | Notas |
|---|---|---|
| `Storage__Endpoint` | `minio:9000` | `localhost:9000` en Development |
| `Storage__AccessKey` / `Storage__SecretKey` | `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` | Obligatorias: sin ellas el API no arranca |
| `Storage__Bucket` | `STORAGE_BUCKET` (`cotla`) | Se crea al arrancar si no existe |
| `Storage__UseSsl` | `false` | |

- `StorageInitializer` crea el bucket en segundo plano con 15 reintentos cada 2 s, porque
  Compose solo garantiza `service_started` para `minio`. Si se agotan, el API sigue arriba,
  `/health` lo reporta y la siguiente subida vuelve a intentarlo.
- El check `almacenamiento` de `/health` tiene un tope de 3 s: sin él, con MinIO caído,
  `/health` tardaba ~14 s en responder.
- La base solo guarda la referencia: `archivos.almacenamiento_clave` y
  `url_doc = s3://{bucket}/{clave}`. Las fotos se sirven **a través del API** con el alcance
  de la marcación, no con URLs directas al bucket; la política de URLs firmadas sigue
  pendiente (§31).
- MinIO AIStor exige `MINIO_ROOT_PASSWORD` de al menos 8 caracteres; con menos, el
  contenedor sale con `Invalid credentials` y `/health` queda `Unhealthy`.

## 17. Checador de asistencia (`/api/v1/attendance`)

| Ruta | Policy | Respuesta |
|---|---|---|
| `POST /attendance/check-in` · `lunch-out` · `lunch-in` · `check-out` | `autenticado` | `multipart/form-data`. 201 `AsistenciaResponse`; 200 si es un reintento |
| `GET /attendance` | `autenticado` + alcance | `PagedResult<AsistenciaResponse>`, del más reciente al más antiguo |
| `GET /attendance/today` | `autenticado` | `JornadaHoyResponse` del propio usuario |
| `GET /attendance/{id}` | `autenticado` + alcance | `AsistenciaDetalle` (`marcacion` + `tienda`) |
| `GET /attendance/{id}/photo` | `autenticado` + alcance | La selfie con su `Content-Type` |

**Formulario de marcación:** `foto` (JPEG o PNG, máx. `Asistencia:FotoMaxBytes` = 5 MB, tipo
decidido por la firma del archivo), `latitud`, `longitud`, `precisionMetros?` y
`eventoDispositivoId` (obligatorio, máx. 100).

Orden de las reglas en `AttendanceService.RegistrarAsync`:

1. Validación de campos → 400 con la clave del campo.
2. **Idempotencia (D7).** Si `(sucursal, eventoDispositivoId)` ya existe para el mismo
   colaborador y tipo → 200 con la original, sin subir nada. Con otro colaborador o tipo → 409.
3. Colaborador con `estado_laboral` distinto de `activo` → 403.
4. `fechaLocal` = hoy en `Asistencia:ZonaHoraria` (`America/Mexico_City`).
5. **Secuencia** (`SecuenciaJornada`): entrada → [salida a comer → regreso] → salida; la
   comida es opcional, pero abierta debe cerrarse antes de la salida; cada tipo una vez por
   día. Violación → 409 con el motivo.
6. **Zona:** distancia Haversine a la tienda; `dentro_de_zona` si `distancia ≤ radio`. Nunca
   bloquea (§6.5).
7. Subida de la foto a `sucursales/{id}/asistencias/{yyyy}/{MM}/{guid}.{ext}`. Si falla → 503
   y no se guarda nada.
8. Transacción con `pg_advisory_xact_lock(sucursal, empleado)`, nueva validación de la
   secuencia, fila en `archivos` (`selfie_asistencia`) y fila en `asistencias`. Si algo
   falla se borra el objeto subido. Un choque de unicidad con el mismo UUID se resuelve
   como el reintento del paso 2, nunca como 500.
9. Auditoría `registrar_asistencia` sobre `asistencias`.

- **`evento_dispositivo_id` es obligatorio** porque el índice es `UNIQUE NULLS NOT DISTINCT
  (sucursal_id, evento_dispositivo_id)`: con `null` la sucursal solo admitiría una marcación.
- **Retardo (D6):** solo en `entrada` y si el colaborador tiene horario,
  `max(0, hora local − (horaEntrada + toleranciaEntrada))` en minutos. Se calcula al
  responder con el horario **vigente**; no se guarda. Si RH cambia el horario, cambia el
  retardo mostrado de marcaciones pasadas.
- **`GET /today`** devuelve `siguientesPermitidas` y `faltantesObligatorias` con la misma
  `SecuenciaJornada`, para que el móvil no replique la regla. No crea incidencias (FASE 5).
- Los filtros de `GET /attendance` (`fechaDesde`, `fechaHasta`, `empleadoId`,
  `departamentoId`, `equipoId`, `tipo`, `estadoZona`) se aplican después del alcance; un
  `tipo` o `estadoZona` desconocido, o `fechaDesde > fechaHasta`, responde 400.

## 18. Decisiones de la FASE 4

Tomadas con el usuario el 2 de octubre de 2026.

| # | Decisión |
|---|---|
| D6 | **Revisada el 23 de octubre de 2026:** bloqueo por horario con `409` y registro en auditoría. Límites provisionales (ver abajo). |
| D7 | **Revisada el 23 de octubre de 2026:** sincronización con la hora capturada por el dispositivo, dentro de límites. Provisional (ver abajo). |
| D8 | **Selfie en multipart dentro de la marcación.** Una sola llamada crea archivo y marcación. |
| D9 | **SDK de MinIO** detrás de `IFileStorage`, para poder cambiar de proveedor S3 en producción. |

**Zona real configurada (22 de octubre de 2026).** La tienda quedó en `20.638516, -103.302586`,
radio 100 m y tolerancia 5 min, con `PATCH /store` por RH. Antes estaba en `0,0`, así que toda
marcación real habría salido *fuera de zona*. Una base nueva sigue arrancando en `0,0` si no se
definen `Seed__SucursalLatitud` / `Seed__SucursalLongitud`.

**Verificación del checador en la pila real** (`docker compose`, Production, cuentas de prueba
`9002` y `9003` con horario Matutino; ambas desactivadas al terminar):

| Comprobación | Resultado |
|---|---|
| `9002` entrada con GPS de la tienda | 201, `dentro_de_zona`, `minutosRetardo` 776 (la prueba corrió de noche; el retardo es real contra las 09:00) |
| Reintento de la entrada con el mismo UUID | 200, misma marcación (id 6), sin duplicado |
| Segunda entrada con otro UUID | 409 «Ya registraste tu entrada hoy.» |
| Salida a comer, regreso y salida | 201 cada una, `dentro_de_zona` |
| `GET /attendance/today` al final | sin `siguientesPermitidas` ni `faltantesObligatorias` |
| `9003` salida sin entrada | 409 «Primero registra tu entrada.» |
| `9003` entrada a ~1,1 km de la tienda | 201, se guarda, `fuera_de_zona`, 1111,95 m |
| Foto que no es JPEG/PNG, sin foto, sin latitud | 400 «Solicitud inválida» con `errors` en español |
| `9003` consulta la marcación o la foto de `9002` | 403 |
| RH descarga la foto de `9002` | 200, 646 bytes |
| RH filtra `estadoZona=fuera_de_zona` | devuelve la marcación de `9003` |

**D6 y D7 implementadas (23 de octubre de 2026).** Los límites son provisionales hasta que el
cliente los confirme (§31). Todos se ajustan por variable de entorno, sin cambiar código.

| Ajuste (`Asistencia:*`) | Variable en `.env` | Por defecto | Qué hace |
|---|---|---|---|
| `BloquearFueraDeHorario` | `ASISTENCIA_BLOQUEAR_FUERA_DE_HORARIO` | `true` | Rechaza con 409 las marcaciones fuera de la ventana. `false` vuelve a solo etiquetar. |
| `MinutosAntesDeEntrada` | `ASISTENCIA_MINUTOS_ANTES_DE_ENTRADA` | 30 | Minutos que se permiten marcar antes de la entrada. |
| `MinutosDespuesDeSalida` | `ASISTENCIA_MINUTOS_DESPUES_DE_SALIDA` | 60 | Minutos que se permiten marcar después de la salida. |
| `ToleranciaRelojSegundos` | `ASISTENCIA_TOLERANCIA_RELOJ_SEGUNDOS` | 120 | Diferencia con el reloj del servidor que se ignora: se usa la hora del servidor. |
| `MaxHorasSinConexion` | `ASISTENCIA_MAX_HORAS_SIN_CONEXION` | 24 | Antigüedad máxima de una marca capturada sin conexión. |

- **Ventana (D6):** `[entrada − MinutosAntes, salida + MinutosDespues]`, una sola para las cuatro
  marcaciones. Sin horario asignado no hay bloqueo. Fuera de la ventana responde `409` con
  «Fuera de horario: puedes marcar de 08:30 a 19:00.»; el intento queda en `auditoria` con la
  acción `marcacion_rechazada_fuera_de_horario`. No se guardan ni la marcación ni la foto.
- **Hora efectiva (D7):** sin `registradaEnDispositivoAt` la marca es en línea. Con él: si la
  diferencia con el servidor está dentro de `ToleranciaRelojSegundos`, se usa la del servidor; si
  el dispositivo va adelantado, `400` en `errors.registradaEnDispositivoAt`; si la marca tiene más
  de `MaxHorasSinConexion` horas, `409`; en otro caso se guarda la hora capturada, que define la
  fecha de la jornada, el retardo y la ventana. Debe incluir zona horaria (p. ej.
  `2026-10-22T08:58:03-06:00`).
- **Orden (D7):** una marca capturada no puede quedar antes de otra ya registrada ese día (`409`).
- **Origen (sin columna nueva):** `origen` es `sincronizada` si `sincronizadaAt > registradaAt`,
  y `en_linea` en otro caso; `desfaseSegundos` es la diferencia en segundos. Las marcas anteriores
  son `en_linea`.
- **Radio de la zona:** se cambia con `PATCH /store` (`radioPermitidoMetros`, mínimo 1). Las marcas
  ya guardadas no se recalculan.
- **Contrato aditivo:** `GET /attendance/today` incluye `ventanaMarcacion`; `AsistenciaResponse`
  incluye `origen`, `sincronizadaAt` y `desfaseSegundos`.

Pruebas: `ReglaHorarioTests` y `ReglaSincronizacionTests` (unitarias), `AttendanceHorarioBloqueoTests`
y `AttendanceSincronizacionTests` (integración). Suite completa: 258 de 258.

**Verificación en la pila real (23 de octubre de 2026)**, con las cuentas `9004` y `9005` (horario
Matutino), desactivadas al terminar. La zona se leyó de la base en el momento de la prueba
(`21.638516, -105.302586`, radio 10 m):

| Comprobación | Resultado |
|---|---|
| `9004` entrada capturada a las 09:30 locales, en el centro de la zona | 201, `dentro_de_zona`, `origen` sincronizada, `desfaseSegundos` 46682, `minutosRetardo` 25 |
| Reintento con el mismo UUID | 200, misma marcación |
| `9004` comida capturada a las 09:00, antes que la entrada | 409 «La hora de la marcación es anterior a otra ya registrada ese día.» |
| `GET /attendance/today` de `9004` | `ventanaMarcacion` 08:30–19:00 |
| `9005` entrada en tiempo real (de madrugada) | 409 «Fuera de horario: puedes marcar de 08:30 a 19:00.»; registros en auditoría 0 → 1 |
| `9005` entrada capturada a las 10:30, a ~1,1 km del centro | 201, `fuera_de_zona`, 1111,95 m |
| `9004` consulta la marca de `9005` | 403 |
| Hora capturada 30 min en el futuro | 400 en `errors.registradaEnDispositivoAt` |
| RH filtra `estadoZona=fuera_de_zona` | devuelve la marca de `9005` |

No se probó el cambio de radio en vivo, porque la tienda la modificó el RH sembrado a las 04:27 UTC
(ver abajo) y no se tocó esa configuración. El cambio de radio tiene prueba automatizada en
`StoreEndpointsTests`.

**Estado de la tienda y de la configuración (23 de octubre de 2026).** Tras un `docker compose
down -v` y `up`, la base se resembró desde `.env` (`20.638516 / -103.302586`, radio 100 y
tolerancia 5 de los valores fijos de la semilla). A las 04:35 UTC el RH ajustó la tienda a
`20.638487 / -103.302512`, radio 10 m y tolerancia 10 min (auditoría `actualizar_tienda`). Hoy la
base y `.env` no coinciden: la semilla solo aplica a una tabla vacía, así que la base no se
sobrescribe, pero si se vuelve a resembrar la tienda arrancará con las coordenadas de `.env` y el
radio y la tolerancia de la semilla. Hay que alinear ambos.

**Almacenamiento tras el reinicio.** MinIO (`aistor-server`) monta `rh-cotla_minio-data` en
`/mnt/data`, el bucket `cotla` existe y `live`/`ready` responden 200. El API crea o verifica el bucket
al arrancar y `/health` da Healthy en `postgres` y `almacenamiento`. El `down -v` borró las
asistencias, archivos, sesiones, restablecimientos de contraseña y cuentas de prueba anteriores.
Quedan volúmenes de configuraciones viejas (`monolito-numex_postgres_data`, `rh-cotla_api-storage`,
`rh-cotla_postgres18-data`) que `down -v` no borra y el compose actual no usa.

Pendientes del checador: límites reales de la ventana (§31), antigüedad máxima y tolerancia de
reloj (D7), precisión mínima del GPS y semántica del retardo.

## 19. Usuarios y roles (`/api/v1/users`)

Recurso propio para la cuenta de acceso (rol, estado, sesiones y reseteo de contraseña),
separado del expediente del colaborador. Todas las rutas exigen `rh` (`Policies.RhOnly`) y
se limitan a la sucursal del token: una cuenta de otra sucursal responde **404**.

El `{id}` de estas rutas es el de la **cuenta** (`usuarios.id`), no el de `empleados`.

| Ruta | Efecto | Errores |
|---|---|---|
| `GET /users` | Lista paginada. Filtros: `estado`, `rol`, `busqueda` (número o nombre). | 401, 403 |
| `GET /users/{id}` | Cuenta, estado laboral y `sesionesActivas`. | 404 |
| `PUT /users/{id}/role` | Body `{ "rol": "gerente" }`. Cambia solo la cuenta; el expediente no se toca. Auditoría `cambiar_rol`. | 400, 404, 409 |
| `POST /users/{id}/activate` | Delegado en `EmployeeService.ActivateAsync` (`activar_cuenta`). | 404 |
| `POST /users/{id}/reject` | Delegado en `EmployeeService.RejectAsync`; solo si el estado laboral es `pendiente` o `rechazado`. | 404, 409 |
| `POST /users/{id}/deactivate` | Body `{ "motivo": "..." }`. Desactiva, registra la baja y revoca sesiones. | 400, 404, 409 |
| `POST /users/{id}/sessions/revoke` | Revoca las sesiones activas (`revocar_sesiones`). | 404 |
| `POST /users/{id}/password-reset` | Envía al correo del colaborador el enlace de recuperación. 204. Auditoría `solicitar_reset_password_rh`. | 404, 409 |

Reglas de negocio nuevas (en `Common/Security/RhGuard.cs`, las mismas que aplica `/employees`):

- **Último RH:** no se puede cambiar el rol de un RH activo, ni desactivarlo o rechazarlo, si
  es el único RH activo de la sucursal → 409.
- **Sin autodegradación:** un RH no puede quitarse su propio rol, desactivarse ni rechazarse → 409.
- **Reseteo:** solo para cuentas `activo` con correo registrado (409 en otro caso). RH nunca ve
  la contraseña; el enlace caduca según `PasswordResetSettings:TokenMinutes`. Sin SMTP
  configurado responde 204 y no envía correo (queda una advertencia en el log).
- **Desactivar no invalida tokens:** la validación de estado ocurre solo al iniciar sesión, así
  que un JWT ya emitido sigue aceptándose hasta expirar (§9).

Cambios en rutas existentes:

- `POST /auth/sessions/revoke-all` ahora responde **404** si el `usuarioId` no existe en la
  sucursal del solicitante. Antes aceptaba cualquier id.
- `PATCH /employees/{id}` (campo `Rol`), `deactivate` y `reject` de colaboradores aplican ahora
  las mismas guardas que `/users`: 409 si el cambio deja la sucursal sin RH activo o si el RH se
  afecta a sí mismo. `activate` no reduce RH y no cambia.

## 20. Errores en español

Todo texto que llega al frontend (y por tanto al usuario final) está en español. Son tres fuentes, todas en `Common/Errors`:

- **Atributos de validación de los contratos.** Cada `ValidationAttribute` declara `ErrorMessage` en español, con el campo en lenguaje natural («El número de empleado no puede superar 50 caracteres.»). La prueba `ErroresEnEspanolTests.Todo_atributo_de_validacion_de_un_contrato_declara_su_mensaje` recorre los tipos `*.Contracts` por reflexión y falla si un contrato nuevo deja un atributo sin mensaje; así no vuelve el texto por defecto de .NET.
- **Títulos y detalles del framework** (`ProblemDetailsEnEspanol.Personalizar`, conectado en `Program.cs` con `AddProblemDetails`). Sustituye el título en inglés de 400/401/403/404/405/409/415/429/500/503 y rellena el `detail` cuando falta. Un error de validación muestra el título «Uno o más campos no son válidos.» y sin detalle, para que el cliente pinte solo los mensajes de campo.
- **Excepciones de dominio** (`GlobalExceptionHandler`): leen el título de la misma tabla, así que los textos no se duplican.

Forma de la respuesta de error: `title`, `status`, y según el caso `detail` o `errors` (mapa campo → mensajes). Ejemplo real, `POST /api/v1/auth/login` con un número de 60 caracteres:

```json
{ "title": "Uno o más campos no son válidos.", "status": 400,
  "errors": { "NumeroEmpleado": ["El número de empleado no puede superar 50 caracteres."] } }
```

Un contrato nuevo debe declarar `ErrorMessage` en todo atributo. Los 401 de autenticación JWT llegan sin cuerpo; el cliente los trata por `expiraAt` (ver [[rh-cotla-integracion-clientes-manual]]).

## 21. Incidencias, correcciones y ausencias (FASES 5 y 6)

Contratos para clientes en [[rh-cotla-integracion-clientes-manual]]. Sin cambios de dependencias.

**Base de datos.** Una migración, `Fase5IncidenciasIdempotentes`: índices únicos parciales
`UX_incidencias_dia_tipo` (incidencias sin marcación: una por colaborador, día y tipo) y
`UX_incidencias_asistencia_tipo` (una por marcación y tipo). `infra/db.sql` sincronizado. La
FASE 6 no agregó migraciones.

**Configuración** (`appsettings.json`): `Incidencias:{Habilitado=true, IntervaloMinutos=30,
DiasRevision=7}` y `Ausencias:{DocumentoMaxBytes=5242880, MaxDocumentos=3}`. Sin variables
nuevas en Docker Compose: usan estos valores por defecto.

### FASE 5

- **Detector** (`Modules/Incidents/Services/IncidenciaDetector`): revisa jornadas cerradas
  (`fecha_local < hoy`) de los últimos `DiasRevision` días. `entrada_faltante` /
  `salida_faltante` solo si la jornada tiene alguna marcación; `fuera_de_zona` por cada marcación
  fuera de zona. Un día sin marcaciones no genera incidencia y la comida no genera incidencia
  (§6.7). Es idempotente (no repite una incidencia aunque esté resuelta) y serializa las corridas
  con `pg_advisory_xact_lock`. Lo ejecuta `IncidenciasHostedService` y `POST /incidents/detect`.
- **Incidencias** (`/api/v1/incidents`): listar, detalle, `resolve`, `dismiss`; escritura solo RH,
  `409` si ya fue atendida.
- **Correcciones** (`/api/v1/attendance-corrections`): el colaborador solicita sobre su propia
  marcación, una sola pendiente por marcación (candado por marcación). **Aprobar aplica el
  cambio sin tocar `asistencias`**: `GET /attendance/{id}` agrega `correccionAprobada`. RH no
  edita marcaciones directamente (decisión de §31 aplicada, por confirmar con el cliente).
- **Auditoría:** `solicitar_correccion_asistencia`, `aprobar_correccion_asistencia`,
  `rechazar_correccion_asistencia`, `resolver_incidencia`, `descartar_incidencia`.
- **Límites conocidos:** no se corrige una marca inexistente (la tabla exige `asistencia_id`);
  la comida no genera incidencias; no hay días laborales definidos.

### FASE 6

- **Solicitudes** (`/api/v1/leave-requests`): `multipart/form-data`, días naturales, fechas
  enteras y no pasadas. Vacaciones exigen saldo y que alcance (`400`); incapacidad exige documento
  (JPEG/PNG/PDF por firma del archivo, 5 MB, 3 archivos), otros tipos no admiten documentos. Se
  rechaza traslape con solicitudes propias pendientes o aprobadas (`409`; defensa propia, no está
  en la especificación). Si la transacción falla se borran los documentos ya subidos.
- **Aprobación** (solo RH): transacción con `pg_advisory_xact_lock` por colaborador; revalida el
  saldo (`409` si ya no alcanza), suma a `dias_utilizados` y registra `movimientos_vacaciones`
  tipo `uso` con **días negativos** ligado a la solicitud. El saldo no se reserva al crear (§8).
- **Cancelación:** solo el dueño y solo `pendiente`.
- **Notificaciones:** al aprobar/rechazar se inserta una fila en `notificaciones`
  (`solicitud_ausencia_aprobada` / `solicitud_ausencia_rechazada`). Consulta y canal: FASE 10.
- **Auditoría:** `crear_`, `cancelar_`, `aprobar_` y `rechazar_solicitud_ausencia`, con el saldo
  antes y después al aprobar vacaciones.
- `/vacation-balances` se conserva como consulta de saldos y movimientos.

### Validación

- `dotnet build`: 0 errores, 0 advertencias. `dotnet test cotla-api.slnx`: 277 casos en verde,
  con la migración aplicada sobre `postgres:17-alpine` real (Testcontainers).
- `dotnet format --verify-no-changes`: sin hallazgos en los archivos de las FASES 5 y 6; queda un
  aviso previo de orden de imports en `AttendanceHorarioBloqueoTests.cs`.
- **No verificado:** el recorrido sobre `docker compose` (Production): este equipo no tiene `.env`
  con las variables obligatorias, y no se inventaron secretos. Debe repetirse con el `.env` real.

## 22. Horas extra (FASE 7, `/api/v1/overtime`)

Módulo `Modules/Overtime` (`HoraExtraService`). Sin migraciones: usa `horas_extra` y `horas_extra_archivos` de la FASE 1.

- `POST /overtime` (multipart; cualquier usuario, para sí mismo) → `201`, estado `pendiente`. Campos: `fecha` (`yyyy-MM-dd`, no futura), `horaInicio`/`horaFin` (`HH:mm`, mismo día, fin > inicio), `concepto` (obligatorio), `comentarios`, `fotos` (0–5, JPEG/PNG por firma, 5 MB c/u). `400` por validación; `503` si falla el almacenamiento (se borran las fotos ya subidas si falla el guardado).
- **Horas calculadas** (§9.2): `totalHoras` es la columna generada de PostgreSQL; el cliente no puede enviarla (un campo `totalHoras` en el formulario se ignora).
- `GET /overtime` (filtros `estado`, `fechaDesde`, `fechaHasta`, `empleadoId`, `equipoId`; paginado), `GET /overtime/{id}`, `GET /overtime/{id}/photos/{archivoId}`: alcance `rh` todo, `gerente` sus equipos (solo lectura), `colaborador` lo suyo.
- `POST /overtime/{id}/approve` y `/reject` (`motivoRechazo` obligatorio, 3–255): solo `rh`; `409` si ya no está pendiente (candado `pg_advisory_xact_lock`). Auditoría en `crear/aprobar/rechazar_hora_extra`.
- Configuración (`HorasExtra` en `appsettings.json`): `FotoMaxBytes`, `MaxFotos`.
- **Fuera de alcance por vacíos de la especificación:** cancelar/editar tras enviar, horas que cruzan medianoche, notificaciones (FASE 10) y reportes (FASE 8).
- **Pruebas:** `OvertimeEndpointsTests` (9) verdes. Suite completa: 285/286; falla `AttendanceCorrectionsTests.Aprobar_conserva_la_marcacion_original…` por precisión de microsegundos al comparar fechas (ajena a la FASE 7). **No verificado:** `docker compose` en Production (sin `.env` real).

## 23. Reportes (FASE 8, `/api/v1/reports`)

Módulo `Modules/Reports` (`ReportService`). Sin migraciones. Todos son `GET`, solo consulta (sin exportación, §11), autenticados, con alcance en el servicio: `rh` todo, `gerente` sus equipos, `colaborador` lo suyo. Paginación estándar (`page`, `pageSize`); fechas `yyyy-MM-dd`; `fechaDesde > fechaHasta` → `400`.

| Endpoint | Contenido | Filtros |
|---|---|---|
| `/reports/attendance` | Delega en el listado de marcaciones (`AsistenciaResponse`: coordenadas, zona, `fotoUrl`) | `fechaDesde`, `fechaHasta`, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estadoZona` |
| `/reports/incidents` | Filas unificadas `{origen, referenciaId, empleado, fecha, tipo, estado, descripcion}`: incidencias (`entrada_faltante`, `salida_faltante`, `fuera_de_zona`), `solicitud_pendiente`, `correccion_pendiente` | fechas, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estado` (por defecto `pendiente`) |
| `/reports/overtime` | `{ registros, totalHorasFiltradas }`; la suma cubre todo el filtro, no solo la página | fechas, `empleadoId`, `departamentoId`, `equipoId`, `estado` |
| `/reports/vacations` | Por colaborador: saldo (iniciales, generados, utilizados, disponibles) y `solicitudes[]` | fechas, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estado` |

- `GET /overtime` y `/reports/overtime` ganan el filtro `departamentoId` (compatible hacia atrás).
- Decisiones: sin `estado`, el reporte de incidencias muestra solo lo pendiente; el `estado` aplica igual a incidencias, solicitudes y correcciones. A diferencia de `/vacation-balances` (rh/gerente), el colaborador sí consulta su propio reporte de vacaciones. Incidencias se une en memoria (20–50 colaboradores).
- **Pruebas:** `ReportsEndpointsTests` (7) verdes: alcance por rol, filtros, 400 por rango/tipo/estado, autenticación, suma de horas y solicitud pendiente que sale del reporte al aprobarse. No se prueba aún el origen `correccion_pendiente` (la consulta existe y compila, sin caso de prueba). Suite completa: 292/293; sigue fallando `AttendanceCorrectionsTests.Aprobar_conserva…` (precisión de microsegundos, previo). **No verificado:** `docker compose` en Production (sin `.env` real).

## 24. Dashboard y calendario (FASE 9)

Módulo `Modules/Dashboard` (`DashboardService`, `CalendarService`, `FechasProximas`). Sin migraciones.

### `GET /api/v1/dashboard?dias=` (`rh` y `gerente`; colaborador → `403`)
Indicadores del día local: `colaboradoresActivos`, `presentesHoy`, `ausentes`, `deLicencia`, `marcacionesFueraDeZonaHoy`, `incidenciasPendientes`, `solicitudesPendientes`, `correccionesPendientes`, `horasExtraPendientes`, `fecha`, `dias`. El alcance lo aplica `AlcanceService`: el gerente solo cuenta su equipo.
- **Presente** = activo con `entrada` hoy. **Ausente** = activo sin entrada y sin ausencia aprobada que cubra hoy. **DeLicencia** = activo sin entrada con ausencia aprobada hoy. Invariante: `activos = presentes + ausentes + deLicencia`. No se considera horario ni descanso (decisión del cliente en esta fase).
- `proximosCumpleanos[]` y `proximosAniversarios[]` (`{empleadoId, numeroEmpleado, nombreCompleto, fecha, anios}`): **solo RH**; el gerente recibe `null` (§12.2 no los lista). Ventana `dias` 1–90, 30 si se omite; fuera de rango → `400`. Un 29-feb se celebra el 28-feb en años no bisiestos; aniversarios solo desde 1 año; sin fecha → se omite.

### `GET /api/v1/calendar?desde=&hasta=&empleadoId=` (autenticado)
`desde` y `hasta` obligatorios, máx. 62 días (`400` si faltan, se invierten o exceden). Sin `empleadoId`, el propio; ajeno → solo RH (`403` para gerente y colaborador, `404` si no existe). Eventos ordenados por fecha: `ausencia` (aprobadas, con `fechaFin`), `asistencia` (uno por día), `incidencia` (cualquier estado), `hora_extra` (solo aprobadas).

- **Vacíos de la especificación:** §13 no autoriza al gerente a ver el calendario de su equipo (no se implementó); §12.1 no acota la ventana de «fuera de zona» (se usa el día de hoy); pendientes (incidencias, solicitudes, correcciones, horas extra) se cuentan sin límite de fecha.
- **Pruebas:** `DashboardEndpointsTests` (15, incluida la lógica de fechas) y `CalendarEndpointsTests` (7). Suite completa: 315/315 verdes. `dotnet format`: solo el aviso previo de `AttendanceHorarioBloqueoTests.cs`. **No verificado:** `docker compose` en Production (sin `.env` real).

## 25. Alertas y notificaciones (FASE 10, `/api/v1/notifications`)

Módulo `Modules/Notifications`. Sin migraciones (usa `notificaciones` y `dispositivos_push` de la FASE 1). Canal: **bandeja consultable + correo SMTP de resoluciones + stream SSE para el panel** (§26). Push/FCM **descartado por ahora**.

### Bandeja (autenticado, siempre las propias)
- `GET /notifications?leida=&page=&pageSize=` · `GET /notifications/unread-count` · `POST /notifications/{id}/read` · `POST /notifications/{id}/unread` · `POST /notifications/read-all`. Ajena o inexistente → `404`.
- Respuesta: `{ id, tipo, titulo, mensaje, leida, leidaAt, createdAt, datos }`; `datos` trae las referencias (`solicitudId`, `horaExtraId`, `empleadoId`, `fecha`).
- Tipos: `solicitud_ausencia_aprobada|rechazada`, `hora_extra_aprobada|rechazada` (con motivo), `cumpleanos_propio|equipo`, `aniversario_propio|equipo`.
- Se crean en la misma transacción que el cambio de estado (`NotificacionService`); las correcciones de marcación **no** notifican (no están en §10).

### Dispositivos
`POST /notifications/devices` `{token, plataforma: android|ios|web}` (upsert por token; si lo registra otra cuenta se reasigna) y `DELETE /notifications/devices/{id}` (baja lógica; `404` si no es del usuario). **No hay envío push todavía.**

### Correo
Solo salen por correo las **resoluciones** (`solicitud_ausencia_aprobada|rechazada`, `hora_extra_aprobada|rechazada`: `TiposNotificacion.ConCorreo`). Las alertas de cumpleaños y aniversarios quedan solo en la bandeja y en el dashboard, sin correo. `NotificacionesHostedService` (cada `Notificaciones:IntervaloSegundos`, 60) envía las no enviadas de las últimas 24 h al `correo` del expediente y marca `enviada_at`. Sin correo → se marca atendida; fallo SMTP → queda pendiente y se reintenta; contenido escapado con `HtmlEncode`. Si SMTP no está configurado, `SmtpEmailSender` omite el envío con aviso y la notificación igual se marca enviada (comportamiento heredado de la FASE 2). `EnvioCorreoHabilitado=false` lo apaga.

### Alertas del día
`AlertasHostedService` (revisa cada 15 min; a partir de `Notificaciones:Alertas:HoraLocal` genera una vez al día y se pone al corriente si arrancó tarde) y `POST /notifications/alerts/generate` (solo RH, su sucursal; `403` para los demás). Para cada activo con cumpleaños o aniversario hoy (aniversario desde 1 año; 29-feb → 28-feb): aviso propio, y a **RH** y a los **gerentes de su equipo** (el propio no recibe duplicado). Idempotente por (usuario, tipo, empleado, fecha). Los «próximos» no se notifican: están en `GET /dashboard`.

- **Vacíos de la especificación:** §10 no fija la hora de las alertas ni si los «próximos» notifican (decisión: se notifica el día y los próximos van por dashboard); el gerente recibe las de **su** equipo; canal push/correo (§31) sin confirmar por el cliente.
- **Pruebas:** `NotificationsEndpointsTests` (10) y `AlertasTests` (6). **No verificado:** envío SMTP real.

## 26. Eventos en tiempo real para el panel (`GET /api/v1/panel/events`, SSE)

> **No está en la especificación** (§10 y §12 no definen avisos en tiempo real): extensión pedida por el usuario. El móvil **no** lo usa; se entera por correo.

- **Quién:** `rh` y `gerente` (policy `rh-or-gerente`); colaborador → `403`, sin token → `401`. Autenticación con `Authorization: Bearer` (no se acepta token en la URL). El `EventSource` nativo no permite esa cabecera: el panel debe usar `fetch` con stream (p. ej. `@microsoft/fetch-event-source`).
- **Formato** `text/event-stream`: `event: conectado` al suscribirse; `event: latido` cada `Notificaciones:PanelLatidoSegundos` (25 s) para que Cloudflare no cierre la conexión ociosa; `event: evento` con `data` = `{ "tipo", "ocurridoAt", "datos" }`. `id` incremental por conexión.
- **Tipos y `datos`:**
  | `tipo` | Cuándo | `datos` |
  |---|---|---|
  | `asistencia` | Se registra una marcación nueva (entrada, salida, comida) | `asistenciaId`, `empleadoId`, `numeroEmpleado`, `nombreCompleto`, `tipo`, `fechaLocal`, `registradaAt`, `estadoZona` |
  | `solicitud_ausencia` | Se crea una solicitud de ausencia | `solicitudId`, empleado, `tipo`, `fechaInicio`, `fechaFin`, `diasSolicitados` |
  | `correccion_asistencia` | Se crea una solicitud de corrección | `correccionId`, `asistenciaId`, empleado, `tipoOriginal`, `fechaLocal`, `motivo` |
- **Alcance:** RH recibe todo lo de su sucursal; el gerente, solo lo de colaboradores de los equipos que gestiona. Los destinatarios se resuelven al publicar.
- **Efímero:** no se guarda ni pasa por la bandeja. Sin repetición de eventos perdidos: al abrir o reconectar, el panel refresca por REST (`/attendance`, `/leave-requests`, `/attendance-corrections`). Una marcación reenviada con el mismo `eventoDispositivoId` o rechazada **no** publica. Un fallo al publicar nunca rompe la operación de negocio.
- **Límites:** 5 conexiones por usuario (la sexta cierra la más antigua). Pub/sub **en memoria**, válido para una sola instancia del API; con varias haría falta un bus (p. ej. Redis) detrás de `PanelBroker`. Horas extra no emiten evento (no se pidió).
- **Pruebas:** `PanelEventsTests` (8): RH recibe cada tipo, gerente solo su equipo, 403/401, reenvío y rechazo no publican, latido, liberación al cerrar y límite de 5 conexiones. Suite completa: 338/339; falla solo `AttendanceCorrectionsTests.Aprobar_conserva…` (precisión de microsegundos, previa). `dotnet format`: solo el aviso previo de `AttendanceHorarioBloqueoTests.cs`. **No verificado:** el stream a través del tunnel de Cloudflare hasta reconstruir la imagen.

## 27. Auditoría: consulta (`GET /api/v1/audit`, solo RH)

Filtros: `fechaDesde`, `fechaHasta` (días **locales**, inclusivos; `desde > hasta` → `400`), `usuarioId`, `accion`, `entidad`, `entidadId`; paginado, más reciente primero; siempre acotado a la sucursal del token. Item: `{ id, fecha (UTC), usuarioId, numeroEmpleado, nombreCompleto, accion, entidad, entidadId, ip, datosAnteriores, datosNuevos }` (los datos como objeto JSON; sus claves van en PascalCase tal como las escribe cada servicio). Colaborador y gerente reciben `403`. Se comprobó que la bitácora no guarda contraseñas ni hashes. Contiene datos personales (p. ej. CURP del expediente): por eso es solo RH.

## 28. Cabeceras de seguridad e IP real

- **Cabeceras** (`SecurityHeadersMiddleware`): `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()`; fuera de Development, `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'` y HSTS cuando la petición es HTTPS. No toca `Cache-Control`.
- **IP real:** `UseForwardedHeaders` es el primer middleware. Confía en `X-Forwarded-For/Proto` solo si la conexión llega de `ForwardedHeaders:TrustedNetworks` (CIDR; por defecto loopback y rangos privados/Docker) y toma **solo la última entrada** (`ForwardLimit = 1`): una IP falsa antepuesta por el cliente se ignora. Lo usan el rate limit y `auditoria.ip`.
- **Riesgo conocido:** el compose publica `8080` y `8088` en el host. Una petición directa desde la LAN también llega como red de confianza y podría falsear `X-Forwarded-For`. Si el móvil no necesita la LAN, publicar solo en `127.0.0.1` (`"127.0.0.1:8088:8080"`) o reducir `TrustedNetworks`. No se cambió sin pedirlo.
- Para que el middleware de caché no sirva respuestas protegidas, `Program.cs` llama a `UseAuthentication`/`UseAuthorization` **explícitamente antes** de `UseOutputCache`.

## 29. Redis

Servicio `redis` (`redis:7-alpine`) en `docker-compose.yml`: **sin puertos publicados**, `--requirepass ${REDIS_PASSWORD}` (variable obligatoria; en `.env.example`), `--appendonly yes` con volumen `redis-data` (la cola sobrevive reinicios), `--maxmemory 256mb --maxmemory-policy volatile-lru` (solo desaloja claves con TTL, es decir el caché; la cola no se desaloja). El API usa `Redis__ConnectionString` con `abortConnect=false`.
- `/health` incluye `redis`: `Healthy`, o **`Degraded`** (HTTP 200) si no está configurado o no responde; nunca `Unhealthy`.
- **Degradación:** sin Redis el API responde igual. El store de caché envuelto (`ResilientOutputCacheStore`) trata cualquier error como *miss*; el encolado de imágenes captura el error y deja la foto original (la barrida la recoge cuando Redis vuelva).

## 30. Caché

Activo con `Cache:Habilitado` (true) y Redis configurado. La suite general lo apaga; `CacheTests` lo activa en un host derivado.
- **Servidor (Output Cache sobre Redis):** solo `GET` con `200` y sesión válida; la clave incluye ruta, query, `Origin` y el alcance.
  | Política | Rutas | Vigencia | Varía por |
  |---|---|---|---|
  | `catalogo` | `GET` de `/departments`, `/positions`, `/schedules`, `/teams`, `/store` (y `/{id}`, `/teams/{id}/managers`) | 10 min (`Cache:CatalogoSegundos`) | sucursal + rol |
  | `tablero` | `GET /dashboard` y `/reports/*` | 15 s (`Cache:TableroSegundos`) | usuario |
  `GET /teams/{id}/members` **no** se cachea (depende del alcance del gerente).
- **Invalidación:** tras un `POST/PUT/PATCH/DELETE` exitoso bajo catálogos **o `/employees`** (los catálogos traen conteos de empleados) se evicta la etiqueta de catálogos de la sucursal. Una escritura rechazada no invalida. Un cambio hecho por fuera de la API no invalida: caduca a los 10 min.
- **Cliente (ETag):** los `GET` de catálogo devuelven `ETag` (hash del cuerpo) y `Cache-Control: private, no-cache`; `If-None-Match` (valor, lista, `W/` o `*`) coincidente → `304` sin cuerpo. Los errores no llevan ETag. Funciona con y sin caché de salida.
- **Fotos:** `GET /attendance/{id}/photo` y `GET /overtime/{id}/photos/{archivoId}` devuelven `ETag` derivado de la **clave de almacenamiento** y `Cache-Control: private, no-cache`; `If-None-Match` vigente → `304` **sin leer MinIO**. La sesión y el alcance se validan antes que el ETag (sin alcance → `403` aunque traiga el ETag). No aplica a documentos de incapacidad.
- **Pruebas:** `CacheTests` (10), `ETagTests` (13).

## 31. Compresión de fotos a WebP (cola)

- **Qué:** solo fotos (`archivos.tipo` `selfie_asistencia` y `foto_hora_extra`, JPEG/PNG). Los documentos de incapacidad **no** se tocan. **El original se reemplaza** por su WebP (decisión del usuario; excepción consciente a «no eliminar historial» de §15: el registro y su historial se conservan, solo cambia la versión comprimida).
- **Encolar sin esperar:** tras guardar, `AttendanceService` y `HoraExtraService` hacen `XADD` a `cotla:imagenes`; la petición responde sin esperar. Mientras la cola no la procesa, la foto sigue siendo la original y se descarga normal.
- **Trabajadores** (`ImagenesHostedService`, `Imagenes:Paralelismo` = 2): `XREADGROUP` (sondeo ~0,5 s), `XACK` al terminar; mensajes colgados se reclaman con `XAUTOCLAIM` pasados `Imagenes:ReclamarSegundos` (300 s); tras `Imagenes:MaxEntregas` (3) entregas fallidas se descartan con log y la foto queda original. Un fallo transitorio (almacenamiento/BD) no se confirma y se reintenta.
- **Procesado** (`ProcesadorImagenes` + `ImagenCompresor`, SkiaSharp): rechaza más de `MaxPixeles` (40 MP); decodifica **ya reducido** cuando puede; **aplica la orientación EXIF** (los selfies de móvil suelen venir girados); reduce el lado mayor a `LadoMaximo` (1600 px); WebP calidad `Calidad` (80), **sin EXIF/GPS**. Si el WebP no pesa menos, se conserva el original. Clave nueva **determinista** (misma ruta con `.webp`), por lo que el reintento es idempotente; actualiza `archivos` (`almacenamiento_clave`, `tipo_mime`, `tamano_bytes`, `url_doc`, extensión de `nombre_original`) solo si la fila aún apunta a la clave vieja, y luego borra el objeto original (mejor esfuerzo; si falla queda un huérfano y se registra).
- **Barrida** (`Imagenes:BarridaHoras` = 6, y al arrancar): encola fotos JPEG/PNG de más de 2 min sin convertir. Cubre fallos de encolado y **comprime la foto histórica** ya guardada. Una foto que no se puede decodificar o no ahorra espacio se reintenta en cada barrida (coste mínimo).
- **Efecto visible para los clientes:** la misma URL de una foto puede pasar de JPEG a WebP poco después de subirla; cambian `Content-Type` y `ETag`. Una descarga que cae justo en el reemplazo relee la clave vigente y reintenta una vez.
- **Pruebas:** `ImagenCompresorTests` (11: reducción, proporciones, PNG, orientación EXIF 1/3/6/8 con verificación de píxeles, corrupta, demasiados píxeles) y `ImagenesProcesoTests` (13: la respuesta no espera, reemplazo y descarga, idempotencia, corrupta, sin ahorro, incapacidad intacta, horas extra, ETag/304/403, clave obsoleta, trabajador de punta a punta con Redis real, reclamo de mensajes, descarte tras demasiadas entregas, barrida, Redis caído).
- **Verificación pendiente en el servidor:** se comprobó que la imagen Docker compila y que `libSkiaSharp.so` carga sin dependencias faltantes; **no** se reconstruyó ni reinició el stack en uso, así que la compresión real contra MinIO y el caché a través del tunnel quedan por confirmar tras `docker compose up --build -d`. Para ello hace falta `REDIS_PASSWORD` en el `.env` (se añadió un valor local generado; no se commitea).

**Otras pruebas de la fase:** `AuditEndpointsTests` (7: solo RH, responsable/acción/fecha/cambio, filtros, orden y paginación, aislamiento de sucursal, 400, sin contraseñas ni hashes) y `SeguridadHttpTests` (8: cabeceras en 200/401/404, sin `Cache-Control` impuesto, IP real desde red de confianza, anti-falsificación y red no confiable).

**Suite:** 400/401 verdes; falla solo `AttendanceCorrectionsTests.Aprobar_conserva…` (precisión de microsegundos, previa). `dotnet format`: solo el aviso previo de `AttendanceHorarioBloqueoTests.cs`.

## 32. Cierre de la versión 1 (13 de noviembre de 2026)

La versión 1 del backend queda **cerrada**. Lo no implementado no se da por hecho: está en el [backlog de la versión 2](plan-backend.md#backlog-de-la-versión-2).

### Qué cubre
| Fase | Resultado |
|---|---|
| 1–3 | Fundaciones, autenticación y colaboradores, configuración y tienda |
| 4–6 | Checador (con modo sin conexión), incidencias y correcciones, ausencias y vacaciones |
| 7–9 | Horas extra con evidencias, reportes, dashboard y calendario |
| 10 | Notificaciones: bandeja, correo de resoluciones, alertas diarias y eventos en tiempo real (SSE) para el panel; **sin push** |
| 11 | Consulta de auditoría, cabeceras de seguridad, IP real tras Cloudflare, Redis, caché con `ETag`, compresión de fotos a WebP por cola |
| 12 | Despliegue por Cloudflare Tunnel desde el equipo del proyecto, panel en el compose, política de reinicio |

### Estado de la suite
`dotnet test` (20-nov-2026, tras el endurecimiento de 1.0.0): **401 de 401** verdes, 0 advertencias de compilación. Antes del endurecimiento, la única falla conocida era `AttendanceCorrectionsTests.Aprobar_conserva_la_marcacion_original…` (comparaba fechas con precisión de microsegundos), ya corregida con tolerancia de 1 ms. `dotnet format`: solo el aviso previo de `AttendanceHorarioBloqueoTests.cs`.

### Estado de despliegue (honesto)
- **Desplegado y verificado por `curl`** (13-nov-2026, `https://api.yefrasoft.com`): fases 1–9 — salud, autenticación RH y gerente, alcance por rol, reportes, dashboard, calendario, validaciones y 401/403/400. Sin escrituras sobre datos reales.
- **En el código pero NO desplegado:** fases 10 y 11 (notificaciones, SSE del panel, auditoría, cabeceras, IP real, Redis, caché, compresión). Para publicarlas:
  ```bash
  docker tag rh-cotla-api rh-cotla-api:previous      # respaldo para rollback
  docker compose up -d --build                        # recrea los contenedores; breve corte (≈1–2 min)
  curl -s https://api.yefrasoft.com/health            # espera postgres, almacenamiento y redis en Healthy
  ```
  Requiere `REDIS_PASSWORD` en `.env` (ya añadido localmente; no se commitea). Los volúmenes (`postgres-data`, `minio-data`, `redis-data`) persisten. El API aplica migraciones al arrancar; después de la Fase 6 **no hubo migraciones nuevas**, así que este despliegue no cambia el esquema.
- **No verificado:** compresión real de fotos contra MinIO, caché y cabecera de IP a través del tunnel, envío SMTP real, reinicio del equipo (política `restart`), y el panel dentro del compose (construcción y arranque).

### Rollback
1. `docker tag rh-cotla-api:previous rh-cotla-api:latest`
2. `docker compose up -d --no-build api`

Es seguro mientras no existan migraciones nuevas (en la versión 1, ninguna posterior a la Fase 6). Antes de **cualquier** versión con migraciones, mientras no haya backups automáticos (v2), hacer un `pg_dump` manual: `docker exec cotla-postgres pg_dump -U cotla cotla_db > respaldo.sql`.

### Endurecimiento aplicado antes de etiquetar 1.0.0
- **Contraseña de Postgres** fuera del compose y de `appsettings.json`: sale de `POSTGRES_PASSWORD` en `.env` (obligatoria, `:?`).
- **Puertos** de Postgres (`5432`) y MinIO (`9000`, `9001`) enlazados solo a `127.0.0.1`.
- **Usuarios de semilla:** los colaboradores demo (201–203) ya **no** se crean por defecto (`SEED_CREAR_COLABORADORES_DEMO=false` en el compose y en `.env.example`). La cuenta inicial es la de RH, sembrada desde `SEED_RH_NUMERO_EMPLEADO` y `SEED_RH_PASSWORD`; el gerente inicial desde `SEED_GERENTE_*`. `.env.example` ya no trae contraseñas ni claves de ejemplo: el compose falla si faltan.
- **`PASSWORD_RESET_BASE_URL`** es obligatoria (antes apuntaba a `localhost`).
- Versión `1.0.0` en `cotla-api.csproj`, `.dockerignore` en el API y prueba intermitente de correcciones estabilizada (compara con tolerancia de 1 ms).

### Registro de deuda técnica al cierre de 1.0.0
**Deuda aceptada** (decisión explícita del usuario; no se corregirá salvo que cambie el contexto):

| # | Deuda | Riesgo | Mitigación vigente |
|---|---|---|---|
| A1 | CORS abierto (`AllowAnyOrigin`, `Program.cs`) | Cualquier sitio puede llamar al API desde un navegador | Autenticación por JWT en cabecera (sin cookies) y autorización por rol/alcance en el backend |
| A2 | Puerto `8080`/`8088` del API publicado y `X-Forwarded-For` confiado desde redes privadas (`ForwardedHeadersSetup.cs`) | Desde la LAN se puede falsear la IP y saltarse el límite de 10 intentos/min en login | Red de confianza; el límite sigue aplicando a clientes que no falsean la cabecera. Cerrar el `8080` o limitar `TrustedNetworks` al proxy real lo elimina |

**Pendientes para futuros fixes** (no bloquean 1.0.0; ver backlog en `plan-backend.md`):

| # | Pendiente | Prioridad sugerida |
|---|---|---|
| P1 | Backups automáticos de PostgreSQL con restauración probada (hoy: `pg_dump` manual) | Alta: hacerlo antes de acumular datos reales |
| P2 | Revocación real de JWT (`jti` en `sesiones_usuario` + `OnTokenValidated`); hoy un usuario dado de baja conserva el token hasta 60 min | Alta (1.0.1) |
| P3 | CI/CD, registry de imágenes y smoke tests | Media |
| P4 | `HEALTHCHECK` en Dockerfile/compose; fijar el tag de la imagen de MinIO (hoy `latest`) | Media |
| P5 | Rate limit general (hoy solo en autenticación) y proteger o retirar `/diagnostics/*` | Media |
| P6 | Exportar el JSON de OpenAPI como artefacto del release (en Production no se publica) | Baja |
| P7 | Una sola instancia del API: el pub/sub del panel está en memoria | Baja hasta escalar |
| P8 | Si ya existen en el servidor los colaboradores demo 201–203 (sembrados antes de este cambio), desactivarlos manualmente: el cambio solo evita crearlos | Operativa: revisar en el despliegue |

## Relacionado

- [[rh-cotla-plan]] — plan padre con el estado por fase y las decisiones vigentes.
- [[rh-cotla-integracion-clientes-manual]] — contrato de consumo del API para web y móvil; este documento describe lo implementado, aquel cómo consumirlo.
- [[rh-cotla-pendientes-sub-plan]] — deuda técnica centralizada; los P1–P8 y A1–A2 de arriba están reflejados como `FX-…`/`F-…`.
- [[rh-cotla-backend-sub-plan]] — checklist por fases del backend (respaldo).
