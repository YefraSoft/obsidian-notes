---
tipo: sub-plan
titulo: "Backend cotla-api — Plan por fases"
autor: Efraín García
creado: 2026-09-11
actualizado: 2026-11-20
estado: completado
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - backend
  - aspnet-core
  - fases
---
# Backend cotla-api — Plan por fases

> **Plan padre:** [[rh-cotla-plan]]
> **Nota (2026-11-20):** este plan por app queda como respaldo. El estado vigente vive en los sub-planes por fase (tabla «Estado Actual» de [[rh-cotla-plan]]) y la deuda en [[rh-cotla-pendientes-sub-plan]].

> Alcance: `app-backend/`, `docker-compose.yml`, `.env.example` y `infra/db.sql`.
> Fuente funcional: `docs/especificaciones.md` (v1.4). Las pantallas móviles y web **no** están aquí: ver [[rh-cotla-movil-sub-plan]] y [[rh-cotla-frontend-sub-plan]].
> Estado detallado de contratos, pruebas y decisiones: [[rh-cotla-progreso-backend-sub-plan]]. Consumo por clientes: [[rh-cotla-integracion-clientes-manual]].
> Plan general y decisiones de stack: [[rh-cotla-plan]].
>
> **Versión 1 cerrada el 13 de noviembre de 2026.** Las fases 1–12 están implementadas o resueltas con el alcance acordado con el usuario; lo que no se hizo **no** está marcado como hecho y vive en el [backlog de la versión 2](#backlog-de-la-versión-2). **Despliegue:** `api.yefrasoft.com` corre hoy las fases 1–9; las fases 10 y 11 están en el código y requieren `docker compose up -d --build` (ver `progreso-backend.md` §32).

---

## Índice

Estado: ✅ completa · 🟡 parcial · ⏳ pendiente.

| Fase | Estado | Qué ataca (especificación) | Detalle |
|------|:------:|----------------------------|---------|
| 1 — Fundaciones y modelo de datos | ✅ | §23 stack, §25 arquitectura, §26 modelo de datos | [→](#fase-1--fundaciones-técnicas-y-modelo-de-datos) |
| 2 — Autenticación, cuentas y colaboradores | ✅ | §3 roles, §4 autenticación y cuentas, §5 expediente y auto-registro, §28 seguridad | [→](#fase-2--autenticación-cuentas-y-colaboradores) |
| 3 — Configuración operativa y tienda | ✅ | §6.5 zona, §6.6 horario, §8 saldos, §14 mapas, §27 fuera de alcance (una sola sucursal) | [→](#fase-3--configuración-operativa-y-tienda) |
| 4 — Checador de asistencia | ✅ | §6.1–6.6 marcaciones, zona, secuencia y horario; §18 sin conexión; §19 errores. Límites de D6 y D7 provisionales, pendientes del cliente | [→](#fase-4--checador-de-asistencia) |
| 5 — Incidencias y correcciones | ✅ | §6.7 marcaciones faltantes, §6.8 corrección, §17 auditoría | [→](#fase-5--incidencias-y-correcciones) |
| 6 — Ausencias y vacaciones | ✅ | §7 solicitudes, §7.4 aprobación, §8 saldos y libro mayor | [→](#fase-6--ausencias-y-vacaciones) |
| 7 — Horas extras | ✅ | §9.1–9.3 registro, cálculo, aprobación; §15 evidencias | [→](#fase-7--horas-extras) |
| 8 — Reportes | ✅ | §11.1–11.4 reportes, §3 alcance por rol | [→](#fase-8--reportes) |
| 9 — Dashboards y calendario | ✅ | §12.1–12.2 dashboards, §13 calendario | [→](#fase-9--dashboards-y-calendario) |
| 10 — Alertas y notificaciones | ✅ | §10 alertas, cumpleaños, aniversarios, notificaciones; canal pendiente §31 | [→](#fase-10--alertas-y-notificaciones) |
| 11 — Archivos, auditoría, seguridad y rendimiento | ✅ | §15 archivos, §16 seguridad, §17 auditoría | [→](#fase-11--archivos-auditoría-seguridad-y-rendimiento) |
| 12 — Despliegue en producción | ✅ (alcance acotado) | §29 criterios de aceptación, §30 recomendaciones. Cerrada en v1 con despliegue por Cloudflare Tunnel; endurecimiento, CI/CD y backups pasan a v2 | [→](#fase-12--despliegue-en-producción) |

**Decisiones pendientes que bloquean ítems** (detalle en §31 de la especificación): D1 devengo de vacaciones, hosting y retención de archivos, asignación de gerentes, corrección directa de marcación. D6 y D7 ya están implementadas con valores provisionales; el cliente fija los definitivos (minutos de la ventana, antigüedad máxima sin conexión y margen de reloj).

---

## Backlog de la versión 2

Todo lo de esta lista **no está hecho**. Se agrupa por motivo; el detalle de cada ítem sigue en la fase donde nació.

**Decisiones del usuario (se difieren a propósito)**
- [ ] Backups automatizados de PostgreSQL con destino fuera del servidor y restauración probada.
- [x] Endurecimiento previo a 1.0.0: contraseña de Postgres a `POSTGRES_PASSWORD` (`.env`), puertos `5432`, `9000` y `9001` solo en `127.0.0.1`, colaboradores demo apagados por defecto, versión `1.0.0`, `.dockerignore`.
- [ ] **Deuda aceptada (no se corregirá en 1.0.0):** CORS abierto (`AllowAnyOrigin`; `Cors:AllowedOrigins` no se lee) y `8080` publicado con `X-Forwarded-For` confiado desde la LAN, que permite saltarse el límite de intentos. Detalle en `progreso-backend.md` §32.
- [ ] `HEALTHCHECK` en el Dockerfile y fijar el tag de la imagen de MinIO (hoy `latest`).
- [x] `GET /api/v1/public/company` (anónimo): datos de contacto de la empresa para el aviso de privacidad de las tiendas. La página pública `/privacidad` del panel (`app-frontend/app/privacidad`) ya consume el endpoint; la app móvil debe enlazarla desde las fichas de las tiendas y, de preferencia, desde la pantalla de inicio de sesión (dependencia del equipo móvil).
- [ ] CI/CD, registry de imágenes con etiquetado por versión, smoke tests automatizados, plan de rollback formal, hosting definitivo y TLS propio.
- [ ] Carga de configuración inicial en producción documentada (tienda, zona, horarios, saldos, usuarios).
- [ ] Push móvil (FCM): descartado por ahora; el registro de tokens (`dispositivos_push`) se conserva.
- [ ] Revocación real de sesiones: `token_id` (`jti`) en `sesiones_usuario` y validación en `OnTokenValidated` (migración; limitación en `progreso-backend.md` §9).
- [ ] Eventos de horas extra hacia el panel (SSE) y un bus (Redis) detrás de `PanelBroker` si hubiera más de una instancia del API.

**Dependen de otro equipo**
- [ ] Dockerfile del panel en `app-frontend/` (el `Dockerfile` ya existe; falta alinear la versión de bun con la acordada, ver contrato en `integracion-clientes.md`).

**Dependen del cliente (§31 de la especificación)**
- [ ] Política de devengo de vacaciones (D1), valores definitivos de D6 y D7 (ventana horaria, antigüedad máxima sin conexión, margen de reloj), corrección directa de marcación, asignación de gerentes, proveedor de almacenamiento y conservación de archivos, y el canal de notificaciones definitivo.

**Deuda menor**
- [ ] Contacto de emergencia en el expediente: la entidad existe, pero no hay endpoint ni campo en `POST/PATCH /employees` (§5.1).
- [ ] Decidir si `POST /users/{id}/password-reset` responde 503 cuando SMTP no está configurado (hoy 204 sin enviar correo).
- [x] Estabilizar `AttendanceCorrectionsTests.Aprobar_conserva_la_marcacion_original…` (resuelto con tolerancia de 1 ms; suite 401/401).
- [ ] El aviso previo de orden de imports en `AttendanceHorarioBloqueoTests.cs`.
- [ ] Verificar en el servidor, tras desplegar las fases 10 y 11: compresión real de fotos contra MinIO, caché a través del tunnel y envío SMTP real.

---

## Fase 1 — Fundaciones técnicas y modelo de datos

> **Estado: ✅ Completada.**

### 1.1 Infraestructura de desarrollo (Docker)

- [x] Revisar `docker-compose.yml` raíz: servicios `api` + `postgres` + `minio` con healthchecks y volúmenes persistentes
- [x] Almacenamiento de archivos: compatible con S3, servido en dev por MinIO AIStor, con volúmenes `minio-data` y `minio-certs`. La BD solo guarda clave y URL, nunca el binario
- [x] Credenciales de MinIO (`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`) documentadas en `.env.example`; la licencia y `mc` quedan fuera del repo vía `.gitignore`
- [x] Variables sensibles vía `.env` (no se commitean valores reales)
- [x] `docker compose up --build` reproducible desde cero
- [x] `curl http://localhost:8080/health` responde `Healthy` (HTTP 200)

### 1.2 Backend base

- [x] `GET /health` y OpenAPI integrado solo en desarrollo
- [x] Estructura por módulo (Endpoints + Contracts; entidades en `Entities/`)
- [x] Middleware global de errores (ProblemDetails RFC 7807)
- [x] Logging estructurado (JSON console)
- [x] CORS para los orígenes de dev de web y móvil
- [x] Contratos de request/response explícitos y validados con DataAnnotations

### 1.3 Modelo de datos (EF Core + migraciones)

> Basado en §26 de la especificación y `infra/db.sql`.

- [x] `Npgsql.EntityFrameworkCore.PostgreSQL` y `Microsoft.EntityFrameworkCore.Design`
- [x] `AppDbContext` registrado con la cadena `Postgres`
- [x] Entidades base: `Sucursal`, `Rol`, `Empleado`, `Departamento`, `Puesto`, `Equipo`, `Horario`, `Usuario`
- [x] Entidades de operación: `Asistencia`, `CorreccionAsistencia`, `SolicitudAusencia`, `SaldoVacaciones`, `MovimientoVacaciones`, `HoraExtra`, `HoraExtraArchivo`, `Notificacion`, `Incidencia`, `Archivo`, `SesionUsuario`, `DispositivoPush`, `EquipoGerente`, `ContactoEmergencia`, `SolicitudAusenciaArchivo`, `Auditoria`
- [x] Constraints de unicidad de CURP, teléfono e identificador de usuario (§5.3)
- [x] Enums de estado: cuenta, zona, solicitud, marcación
- [x] Migración inicial versionada aplicada con `dotnet ef database update`
- [x] Migraciones versionadas como único mecanismo de cambios al esquema
- [x] Tabla `restablecimientos_password` (tokens de recuperación, creada en la FASE 2; la especificación §26 no la lista)

### 1.4 Seeds y catálogos base

- [x] Seed de roles: `colaborador`, `gerente`, `rh`
- [x] Seed de usuario RH inicial (`Seed__Rh*`, BCrypt, validado con `PasswordPolicy`)
- [x] Seed de tienda `Ferretería Cotla`: ubicación desde `SEED_SUCURSAL_LATITUD/LONGITUD` (0,0 si no se definen); radio (100 m) y tolerancia (5 min) son valores fijos de la semilla
- [x] Seed de departamentos, puestos, horarios y equipos vía `DbSeeder` idempotente
- [x] Seeds solo si no existen

### 1.5 Validación

- [x] `dotnet build` sin errores ni advertencias
- [x] Contenedores `api` y `postgres` healthy
- [x] Migraciones aplicadas en BD limpia
- [x] OpenAPI en desarrollo (`/openapi/v1.json`)

---

## Fase 2 — Autenticación, cuentas y colaboradores

> **Objetivo:** registro, activación y administración del colaborador, con autenticación segura y RBAC (§4, §5).
> **Estado: ✅ Backend completo**, validado con pruebas automatizadas y contra `docker compose`. Consumo en [[rh-cotla-integracion-clientes-manual]].
> Las pantallas de esta fase viven en `plan-movil.md` y `plan-frontend.md`, Fase 2.

### 2.1 Autenticación

- [x] JWT Bearer para API móvil y web
- [x] `POST /auth/login` (número de empleado + contraseña)
- [x] `POST /auth/logout` (revocación informativa, ver §2.4)
- [x] `POST /auth/change-password`
- [x] `POST /auth/forgot-password` (token por correo)
- [x] `POST /auth/reset-password`
- [x] SMTP de desarrollo configurado (Gmail) y `PASSWORD_RESET_BASE_URL` pasado por `docker-compose.yml`
- [ ] **(v2)** `POST /users/{id}/password-reset` responde 204 sin enviar correo si SMTP no está configurado; decidir si pasa a 503 (ver backlog de la versión 2)
- [x] Contraseñas hasheadas con BCrypt
- [x] Claims de identidad, rol y scope (empleado/gerente/RH)
- [x] Bloquear login a cuentas `Pendiente`/`Inactivo`/`Rechazado`
- [x] `GET /auth/me`

### 2.2 Roles y permisos (RBAC)

| Recurso | Colaborador | Gerente | RH |
|---|---|---|---|
| Datos propios | ✔ | — | ✔ |
| Equipo/área asignada | — | ✔ (solo lectura) | ✔ |
| Todos los colaboradores | — | — | ✔ |
| Aprobaciones / config | — | — | ✔ |

- [x] Policies `autenticado`, `rh-only`, `rh-or-gerente` (`Common/Config/Security/Models/Policies.cs`)
- [x] Endpoints protegidos con policy; sin `AuthorizeAttribute` con roles en línea
- [x] Alcance validado en backend por `AlcanceService`: filtro por sucursal siempre; `rh` todos; `gerente` por `equipos_gerentes`; `colaborador` solo su registro. La sucursal sale del token

### 2.3 Colaboradores y expediente

- [x] `GET/POST /employees`, `GET/PATCH /employees/{id}`
- [x] `POST /employees/{id}/activate`, `/deactivate`, `/reject`
- [x] `POST /employees/self-register` (estado `Pendiente`; solo móvil)
- [x] Duplicados validados: CURP, teléfono, identificador de usuario (§5.3)
- [x] Al desactivar: conserva historial y registra fecha/motivo de baja
- [x] Vínculo colaborador ↔ usuario, departamento, equipo y horario
- [x] `GET /api/v1/employees/validate-number` (el número lo genera el cliente)
- [x] Errores en español (validación, framework, dominio)
- [x] Gestión de cuentas y roles en `/api/v1/users` con guardas de último RH (`progreso-backend.md` §19)
- [ ] **(v2)** Contacto de emergencia en el expediente (`contacto_emergencia`, §5.1 de la especificación): la entidad existe, pero no hay endpoint ni campo en `POST/PATCH /employees`

### 2.4 Gestión de sesiones

- [x] `GET /auth/sessions` y `POST /auth/logout`
- [x] `POST /auth/sessions/revoke-all` (RH, acotado a su sucursal) y `POST /api/v1/users/{id}/sessions/revoke`
- [ ] **(v2) Revocación real** (limitación transversal, ver backlog de la versión 2; no bloquea el cierre de la fase): hoy `revocada_at` es informativa; el token sigue válido hasta `ExpirationMinutes` (60). Requiere `token_id` y validación en `OnTokenValidated`

### 2.7 Validación

- [x] Auto-registro → `Pendiente` → RH activa → colaborador opera
- [x] Login de cuenta `Inactiva`/`Pendiente` rechazado (403)
- [x] Colaborador no accede a datos ajenos (403)
- [x] Cambio y recuperación de contraseña completos
- [x] Contraseñas solo como hash

> | Criterio | Prueba | Evidencia |
> |---|---|---|
> | Auto-registro → RH activa → opera | `EmployeesEndpointsTests.Colaborador_pendiente_no_puede_iniciar_sesion_hasta_ser_activado` | `self-register` → 201; login → 403; `activate` → 200; login → 200 |
> | El colaborador solo accede a lo suyo | `Colaborador_autoregistrado_solo_ve_su_propio_expediente` | Su listado contiene exactamente su número de empleado |
> | Login de `Pendiente` / `Rechazado` / `Inactivo` | `AuthEndpointsTests.Login_con_cuenta_no_activa_devuelve_403` | Teoría sobre los tres estados → 403 |
> | Colaborador sin acceso a datos ajenos | `Gerente_no_puede_ver_el_expediente_de_otro_equipo`, `Colaborador_no_puede_ver_el_expediente_de_otro`, `Colaborador_no_puede_crear_colaboradores`, `AuthEndpointsTests.Revoke_all_requiere_rh` | 403 |
> | Alcance del gerente por equipo | `Gerente_ve_solo_a_su_equipo` | Exacto `{201, 202}`; el 203 de Almacén queda excluido |
> | Contraseñas solo como hash | `La_contrasena_se_guarda_solo_como_hash_bcrypt`, `El_hash_bcrypt_es_distinto_en_cada_cuenta` | BCrypt `$2…`; hash distinto por cuenta |
> | Cambio de contraseña | `Cambio_de_password_habilita_la_nueva_y_anula_la_anterior` | 204; login nueva 200, anterior 401 |
> | Recuperación de contraseña | `PasswordResetTests.Recuperacion_de_password_completa_el_flujo_hasta_el_login` | Token del correo capturado → 204 → login 200 |
> | Token de un solo uso | `El_token_es_de_un_solo_uso`, `Una_nueva_solicitud_invalida_el_token_anterior` | Reutilizar → 400 |
> | Token solo como hash | `El_token_de_recuperacion_se_guarda_solo_como_hash` | 64 hex, distinto del token en claro |
>
> El enlace de recuperación no se ejercita por HTTP con SMTP real: las pruebas sustituyen `IEmailSender` por un capturador. El límite de intentos (10/min por IP) se verifica por `curl` contra `docker compose`, porque la suite lo desactiva ([[rh-cotla-integracion-clientes-manual]]).

---

## Fase 3 — Configuración operativa y tienda

> **Objetivo:** catálogos y configuración que el checador y las solicitudes requieren (§6.5, §6.6, §8).
> **Estado: ✅ Backend completa.** Sin migraciones: el modelo existe desde la FASE 1. Contratos en `progreso-backend.md` §11–§15.

### 3.1 Departamentos, áreas y equipos

- [x] CRUD de departamentos — `/api/v1/departments`, escritura solo RH, baja lógica
- [x] CRUD de equipos y gerentes — `/api/v1/teams`, `/teams/{id}/managers` (m2m, varios gerentes por equipo)
- [x] Asignar colaborador a departamento/equipo/gerente — `PATCH /employees/{id}` con `ReferenciaCatalogoService` (404/400, nunca 500)

### 3.2 Puestos

- [x] Catálogo de puestos — `/api/v1/positions`

### 3.3 Horarios laborales

- [x] CRUD de horarios: `start_time`, `lunch_start`, `lunch_end`, `end_time`, `tolerance_minutes` — `/api/v1/schedules`
- [x] Tolerancia configurable por horario; la tienda aporta el valor por defecto (D2)
- [x] Comida de referencia de 1 hora; la comida es opcional por horario
- [x] Asignar horario a colaborador — `horarioId` en `POST`/`PATCH /employees`

### 3.4 Tienda y zona geográfica

- [x] `GET`/`PATCH /api/v1/store` (recurso singular, §27)
- [x] Latitud, longitud y radio — lectura abierta a todos los roles para el geofencing

### 3.5 Saldos iniciales de vacaciones

- [x] CRUD de `VACATION_BALANCE` — `/api/v1/vacation-balances`
- [ ] Política de devengo (anual o proporcional) — **diferido (D1)**, por confirmar con el cliente; se retoma en FASES 6/10
- [x] Ajustes manuales por RH con auditoría — `POST /vacation-balances/{id}/adjustments` (signo, D4)

### 3.7 Validación

- [x] Solo RH escribe configuración; gerente y colaborador reciben 403
- [x] Horarios y zona persisten y son consultables
- [x] Saldo inicial disponible para validar solicitudes

> **Decisiones cerradas el 25 de septiembre de 2026** (detalle en `progreso-backend.md` §15):
>
> | # | Decisión |
> |---|---|
> | D1 | Devengo fuera de fase: solo saldo inicial y ajuste manual |
> | D2 | Tolerancia en el horario; la de la tienda es el valor por defecto |
> | D3 | Catálogos y tienda legibles por cualquier rol; escritura solo RH; saldos por alcance |
> | D4 | Ajuste con signo: positivo a generados, negativo a iniciales |
> | D5 | `null` en `PATCH /employees` no desasigna |
>
> Pruebas de respaldo: `*_no_puede_*` en `DepartmentsEndpointsTests`, `PositionsEndpointsTests`, `SchedulesEndpointsTests`, `TeamsEndpointsTests`, `StoreEndpointsTests`, `VacationBalancesTests`; `TeamsEndpointsTests.Asignar_un_gerente_amplia_su_alcance_y_retirarlo_lo_reduce`; `EmployeeReferenceValidationTests`.
>
> Anotado fuera de fase: motor de devengo (D1), límites del horario (§31), múltiples sucursales y turnos rotativos (§27). Para la FASE 4 conviene acordar un radio mínimo: la base solo exige `> 0`.

---

## Fase 4 — Checador de asistencia

> **Objetivo:** marcaciones con hora de servidor, selfie, GPS, zona y secuencia (§6).
> **Estado: ✅ Backend completo** (D6 y D7 implementadas el 23 de octubre de 2026 con límites provisionales). Pendiente de confirmar con el cliente (§31): los minutos de la ventana, la antigüedad máxima sin conexión y el margen de reloj. Sin migraciones.
> Las pantallas (cámara, historial, consulta web) están en `plan-movil.md` y `plan-frontend.md`, Fase 4.

### 4.1 Endpoints de marcación

- [x] `POST /attendance/check-in` — `multipart/form-data` con selfie, GPS y `eventoDispositivoId`
- [x] `POST /attendance/lunch-out`, `lunch-in`, `check-out`
- [x] `GET /attendance` (filtros fecha, colaborador, área, `tipo`, `estadoZona`; alcance por rol)
- [x] `GET /attendance/{id}` (incluye zona de la tienda); foto en `GET /attendance/{id}/photo`
- [x] Hora oficial generada y almacenada por el **servidor** (§6.2)
- [x] `GET /attendance/today` con siguientes marcaciones permitidas y faltantes obligatorias

### 4.2 Reglas de secuencia y zona

- [x] Secuencia entrada → comida → salida; inválida → 409
- [x] Mismo tipo no se repite en el día → 409, con candado por colaborador
- [x] Zona `Dentro` / `Fuera de zona` por Haversine contra el radio
- [x] **No bloquear** fuera de zona; solo etiquetar (§6.5)
- [x] Bloqueo por horario laboral (D6): ventana `[entrada − 30 min, salida + 60 min]`, 409 y registro del intento en auditoría; límites provisionales, ajustables por variable de entorno (`progreso-backend.md` §18)
- [x] Radio de la zona modificable con `PATCH /store` (`radioPermitidoMetros`); cubierto por `StoreEndpointsTests`

### 4.5 Sincronización sin conexión (D7)

> **D7 implementada con límites provisionales:** el dispositivo puede enviar `registradaEnDispositivoAt` (hora con zona). Se usa si está dentro de 120 s del servidor o se rechaza si va adelantada, y se rechaza con 409 si tiene más de 24 h. Sin cambios de esquema: `origen` y `desfaseSegundos` se derivan de `registrada_at` y `sincronizada_at`.

- [x] Hora capturada por el dispositivo, con validación de adelanto, antigüedad y orden del día
- [x] Origen derivado (`en_linea` / `sincronizada`) y desfase en la respuesta
- [ ] Confirmar con el cliente la antigüedad máxima (24 h provisional) y el margen de reloj (120 s provisional) (§18)
- [ ] Política de conflictos que falta: GPS desactualizado y fotos pendientes de subir. Un dispositivo con reloj atrasado puede reclamar una hora anterior; hoy se controla por orden y antigüedad, y RH ve el desfase

### 4.7 Validación

- [x] Entrada guarda hora (servidor), GPS y selfie
- [x] Detección de `Fuera de zona`
- [x] Comida opcional y salida operan
- [x] Secuencia inválida y duplicados rechazados
- [x] RH consulta detalle por colaborador y día
- [x] Verificación de D6 y D7 en la pila real (23 de octubre de 2026): ventana con 409 y auditoría, hora capturada, orden, fuera de zona y alcance (`progreso-backend.md` §18)
- [x] Zona de la tienda configurada el 22 de octubre; hoy la base tiene `20.638487, -103.302512` (radio 10 m, tolerancia 10 min) tras el reajuste del RH (`progreso-backend.md` §18)
- [x] Recorrido de extremo a extremo en la pila real: entrada, comida, salida, reintento idempotente, fuera de zona y alcance (`progreso-backend.md` §18)

> **Decisiones cerradas el 2 de octubre de 2026** (detalle en `progreso-backend.md` §18):
>
> | # | Decisión |
> |---|---|
> | D6 | Revisada el 23-oct-2026: rechazo 409 fuera de la ventana, con auditoría (provisional) |
> | D7 | Revisada el 23-oct-2026: hora capturada por el dispositivo dentro de límites (provisional) |
> | D8 | La selfie viaja en `multipart/form-data` dentro de la marcación |
> | D9 | SDK de MinIO detrás de `IFileStorage` (`Common/Storage`) |
>
> Pruebas: `AttendanceEndpointsTests` (entrada completa, fuera de zona, comida opcional, secuencia 409 en 5 casos, duplicados, reintento idempotente, detalle y foto, alcance por rol). Suite completa en verde; pila real `docker compose` (Production) con `/health` Healthy y recorrido de endpoints con tokens reales (119 comprobaciones).
>
> La zona de desarrollo está así: la base tiene `20.638487, -103.302512` (radio 10 m, tolerancia 10 min) y `.env` sigue con `20.638516, -103.302586`. Están desalineados (`progreso-backend.md` §18). Los límites de D6 y D7 son provisionales hasta que el cliente los confirme.

---

## Fase 5 — Incidencias y correcciones

> **Objetivo:** detectar marcas faltantes y gestionar correcciones (§6.7, §6.8, §17).
> **Estado: ✅ Backend completo** (12-nov-2026). Una migración (`Fase5IncidenciasIdempotentes`: dos índices únicos parciales en `incidencias`). Contratos y decisiones en `progreso-backend.md` §21.
> Las pantallas están en `plan-movil.md` y `plan-frontend.md`, Fase 5.

### 5.1 Detección de incidencias

- [x] Detectar marcas obligatorias faltantes (sin entrada, sin salida) — `IncidenciaDetector`, solo jornadas cerradas con al menos una marcación
- [x] Detectar marcaciones `Fuera de zona`, ligadas a la marcación
- [x] Clasificar incidencias `pendiente` / `resuelta` / `descartada`
- [x] Consulta para reportes — `GET /api/v1/incidents` (filtros fecha, colaborador, departamento, equipo, tipo, estado; alcance por rol)
- [x] Detección idempotente en segundo plano (`IncidenciasHostedService`, cada 30 min, 7 días atrás) y bajo demanda con `POST /incidents/detect` (RH)

### 5.2 Corrección de marcaciones

- [x] Solicitud de corrección: marcación relacionada, motivo, hora y/o tipo propuestos, solicitante, estado — `/api/v1/attendance-corrections`
- [x] Revisión por RH (aprobar / rechazar con motivo)
- [x] **Conservar la marcación original**: aprobar no modifica `asistencias`; `GET /attendance/{id}` expone `correccionAprobada` y cada decisión queda en auditoría (§6.8)
- [x] Decisión de §31: RH **no** edita marcaciones directamente; solo aprueba solicitudes (aplicada, pendiente de confirmar con el cliente)
- [x] Resolver / descartar incidencias — `POST /incidents/{id}/resolve|dismiss` (RH), con auditoría

### 5.4 Validación

- [x] Se detectan marcas obligatorias faltantes
- [x] La corrección no sobreescribe el histórico original
- [x] Auditoría registra quién, acción y fecha

> **Decisiones de implementación (confirmar con el cliente):** un día sin ninguna marcación no es incidencia (la especificación no define días laborales); no se corrige una marca inexistente (`correcciones_asistencia.asistencia_id` es obligatorio): ese caso lo atiende RH sobre la incidencia; la comida no genera incidencia (§6.7).
> Pruebas: `IncidentsEndpointsTests`, `AttendanceCorrectionsTests`.

---

## Fase 6 — Ausencias y vacaciones

> **Objetivo:** solicitudes de vacaciones, permisos e incapacidades con validación de saldo y aprobación RH (§7, §8).
> **Estado: ✅ Backend completo** (12-nov-2026). Sin migraciones. Contratos y decisiones en `progreso-backend.md` §21.
> Las pantallas están en `plan-movil.md` y `plan-frontend.md`, Fase 6.

### 6.1 Solicitudes

- [x] `POST /leave-requests` (`multipart/form-data`)
- [x] `GET /leave-requests` (filtros por estado, tipo, fechas, colaborador, equipo)
- [x] `GET /leave-requests/{id}`
- [x] `POST /leave-requests/{id}/cancel` (solo el dueño y solo `pendiente`)
- [x] `POST /leave-requests/{id}/approve`
- [x] `POST /leave-requests/{id}/reject` (motivo obligatorio)
- [x] Tipos: `vacaciones`, `permiso_con_goce`, `permiso_sin_goce`, `incapacidad`

### 6.2 Reglas de negocio

- [x] Vacaciones: validar saldo antes de crear (400 si no existe saldo o no alcanza)
- [x] Rechazar medio día (solo fechas enteras) y fechas pasadas con error claro; los días son naturales (`fin − inicio + 1`)
- [x] Saldo no se descuenta mientras esté `Pendiente`; se **revalida y descuenta al aprobar**, en transacción con candado por colaborador (409 si ya no alcanza)
- [x] Cancelar solo `Pendiente`; las aprobadas se conservan en histórico
- [x] Defensa adicional (no está en la especificación): no se permiten solicitudes `pendiente`/`aprobada` traslapadas del mismo colaborador (409)

### 6.3 Documentos de incapacidad

- [x] Adjuntar documento a la incapacidad (§7.3) y descargarlo con `GET /leave-requests/{id}/documents/{archivoId}` (mismo alcance que la solicitud)
- [x] Obligatoriedad: documento **obligatorio** (confirmado)
- [x] Restringir tipos (JPEG, PNG, PDF por firma del archivo), tamaño (5 MB) y cantidad (3), configurables en `Ausencias:*`

### 6.4 Libro mayor de vacaciones

- [x] Consulta de saldo: inicial, devengado, utilizado, disponible (`/vacation-balances`)
- [x] Descontar al aprobar
- [x] Registrar movimientos (`uso` en negativo ligado a la solicitud) con auditoría
- [x] `/vacation-balances` se conserva como la consulta de saldos y movimientos; no se crea `/vacations`
- [ ] Devengo automático — diferido (D1)

### 6.5 Notificaciones de cambio de estado

- [x] Al aprobar/rechazar se inserta una fila en `notificaciones` (in-app); el canal push/correo y los endpoints de consulta siguen en Fase 10
- [x] Motivo de rechazo visible en el historial del colaborador (`GET /leave-requests`)

### 6.7 Validación

- [x] Vacaciones sin saldo no se envían
- [x] RH aprueba/rechaza con motivo y trazabilidad (auditoría)
- [x] El saldo refleja ajustes y consumos
- [x] Cancelación solo en `Pendiente`

> Pruebas: `LeaveRequestsTests`.

---

## Fase 7 — Horas extras

> **Objetivo:** captura, cálculo automático, evidencia y aprobación (§9, §15).
> **Estado: ✅ Backend completa.** Sin migraciones (el modelo existe desde la FASE 1). Contratos en `progreso-backend.md` §22.

### 7.1 Registro

- [x] `POST /overtime`, `GET /overtime`, `GET /overtime/{id}`
- [x] Campos: fecha, hora inicial, hora final, concepto, estado, comentarios, aprobación
- [x] `total_hours = hora_final - hora_inicial` calculado en backend
- [x] Impedir alteración manual de horas (§9.2)
- [x] `POST /overtime/{id}/approve` y `/reject` (motivo obligatorio)
- [x] Estados: `Pendiente`, `Aprobada`, `Rechazada`

### 7.2 Evidencias

- [x] Recibir hasta **5 fotografías** por solicitud y guardar en `horas_extra_archivos` (límite validado en backend)
- [x] Subir al almacenamiento y guardar referencias, no binarios

### 7.4 Validación

- [x] Horas calculadas automáticamente y no editables
- [x] Hasta 5 fotografías consultables por RH
- [x] Aprobadas/rechazadas con motivo; las aprobadas aparecen en reportes

---

## Fase 8 — Reportes

> **Objetivo:** consultas con filtros, sin exportación (§11).
> **Estado: ✅ Backend completa.** Sin migraciones. Contratos en `progreso-backend.md` §23.

- [x] `GET /reports/attendance` — colaborador, fecha, tipo, hora, coordenadas, zona, fotografía; filtros fecha, colaborador, área (§11.1)
- [x] `GET /reports/incidents` — marcas faltantes, fuera de zona, solicitudes y correcciones pendientes; filtros fecha, colaborador, área, estado, tipo (§11.2)
- [x] `GET /reports/overtime` — colaborador, fecha, horas, concepto, estado, evidencias (§11.3)
- [x] `GET /reports/vacations` — saldo, días utilizados y disponibles, historial de solicitudes (§11.4)
- [x] Control de alcance: colaborador solo lo propio; gerente su equipo; RH todo
- [x] Reportes devuelven la información indicada con filtros funcionales
- [x] Alcances por rol verificados en backend (colaborador, gerente y RH) en cada reporte
- [x] Escalamiento para 20–50 colaboradores y 1 tienda
- [ ] Pantallas de reportes: ver `plan-frontend.md`, Fase 8

---

## Fase 9 — Dashboards y calendario

> **Objetivo:** indicadores operativos y calendario (§12, §13).
> **Estado: ✅ Backend completa.** Sin migraciones. Contratos en `progreso-backend.md` §24.

- [x] `GET /dashboard` (RH) con indicadores §12.1: activos, presentes hoy, ausentes, fuera de zona, incidencias, solicitudes y horas extra pendientes, próximos cumpleaños y aniversarios
- [x] `GET /dashboard` (gerente): mismos indicadores restringidos a su equipo, sin cumpleaños ni aniversarios (§12.2)
- [x] `GET /calendar`: calendario personal: ausencias aprobadas, asistencia e incidencias, horas extra aprobadas (§13)
- [x] Consulta por RH de calendarios de colaboradores
- [x] Validación: indicadores consistentes con datos reales; el gerente no ve datos fuera de su equipo

> Consumo: `plan-frontend.md`, Fase 9 (dashboards y calendarios web), y `plan-movil.md`, Fase 9 (calendario personal).

---

## Fase 10 — Alertas y notificaciones

> **Objetivo:** comunicación y recordatorios (§10).
> **Estado: ✅ Backend completa, sin push (FCM).** Decisión del usuario: el **móvil se entera por correo** cuando se aprueba o rechaza algo (más la bandeja por API), y el **panel recibe eventos en tiempo real** por SSE. El push móvil queda descartado por ahora; el registro de tokens (`dispositivos_push`) se conserva por si se retoma. Sin migraciones. Contratos en `progreso-backend.md` §25 y §26.
> **Dependencia para el equipo web:** consumir `GET /api/v1/panel/events` (ver `integracion-clientes.md`); no se edita `app-frontend/` desde aquí.

- [x] Alertas automáticas: cumpleaños y aniversarios del **día**, generadas a diario (`Notificaciones:Alertas:HoraLocal`, 07:00) para el colaborador, RH y los gerentes de su equipo; sin correo (bandeja). Los «próximos» se consultan en `GET /dashboard` (Fase 9)
- [x] Notificaciones por cambio de estado: solicitud aprobada/rechazada; hora extra aprobada/rechazada (bandeja **y correo**)
- [x] Canal definido: bandeja por API + correo SMTP de resoluciones + stream SSE para el panel (el cliente aún puede confirmar otro canal, §31)
- [x] Entidad `notificaciones` (usuario, tipo, título, mensaje, leída, fecha) con `datos` para navegar
- [x] Almacenar, consultar, marcar leída/no leída (`/notifications`, `unread-count`, `read`, `unread`, `read-all`)
- [x] SMTP: `NotificacionDespachador` envía las resoluciones pendientes cada 60 s; sin correo se marca atendida; si el SMTP falla se reintenta dentro de 24 h
- [x] Eventos en tiempo real para el panel (extensión pedida; **no está en §10/§12**): marcaciones, solicitudes de ausencia y correcciones nuevas → RH y gerentes del equipo
- [x] Sin buzón interno separado (fuera de MVP): solo bandeja de avisos
- [x] Validación: el colaborador recibe aviso al aprobar/rechazar; RH y gerentes reciben las alertas y los eventos de su alcance
- [ ] (Descartado por ahora) Integrar FCM / push móvil

> Consumo: `plan-movil.md`, Fase 10, y `plan-frontend.md`, Fase 10.

---

## Fase 11 — Archivos, auditoría, seguridad y rendimiento

> **Objetivo:** archivos seguros, auditoría (§17), endurecimiento (§16) y rendimiento.
> **Estado: ✅ Backend completa.** Sin migraciones. Contratos en `progreso-backend.md` §27–§31. **Backups automáticos de PostgreSQL y revocación real de sesiones quedan para la versión 2** (ver el backlog de la versión 2).

### 11.1 Archivos

- [x] Almacenamiento compatible con S3 (`IFileStorage`, MinIO en dev), adelantado en FASE 4
- [x] Subir fotos de horas extra al almacenamiento (FASE 7; asistencia y documentos de incapacidad, FASES 4 y 6)
- [x] Guardar referencia y clave en BD, no el binario
- [x] Descarga privada: se sirve **por la API con `Bearer`**, validando sesión y alcance en cada petición (decisión del usuario: sin URLs firmadas ni prefirmadas; MinIO no se expone)
- [x] No eliminar archivos del historial laboral: ningún proceso borra archivos (solo huérfanos de un guardado fallido). **Excepción consciente:** al comprimir una foto se reemplaza el original por su WebP (§11.5)

### 11.2 Auditoría

- [x] Auditoría de login, contraseñas, expediente, configuración, marcaciones, activar/desactivar cuenta, saldos y horas extra (`auditoria`)
- [x] Auditar aprobaciones y rechazos de solicitudes de ausencia con motivo (FASE 6) y de horas extra (FASE 7)
- [x] Auditar correcciones de marcaciones y resolución de incidencias (FASE 5)
- [x] `GET /api/v1/audit` (solo RH, acotado a su sucursal; filtros fecha, usuario, acción, entidad, registro) — la pantalla está en `plan-frontend.md`, Fase 11
- [x] La bitácora no guarda contraseñas ni hashes (probado)

### 11.3 Seguridad

- [x] HTTPS y certificados: los termina el tunnel de Cloudflare (`api.yefrasoft.com`); la configuración definitiva de despliegue es de la FASE 12
- [x] Cabeceras de seguridad (`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, CSP, HSTS fuera de Development)
- [x] IP real detrás de Cloudflare (`ForwardedHeaders:TrustedNetworks`): el rate limit por IP y `auditoria.ip` usan la del cliente
- [x] Rate limiting en login, recuperación y auto-registro (10/min por IP)
- [x] Validación de tamaño y tipo de archivos (firma real, tamaño y cantidad); protección de rutas privadas
- [x] Control de permisos por rol y alcance en cada endpoint
- [ ] **(v2)** Backups automatizados de PostgreSQL
- [ ] **(v2)** Revocación real de sesiones (`jti`)

### 11.4 Validación

- [x] RH descarga fotos y documentos con sesión y alcance (con `Bearer`)
- [x] Auditoría registra responsable, acción, fecha y cambio
- [x] Endpoint protegido rechaza sesión inválida (401/403), también con caché y `ETag`

### 11.5 Rendimiento (nuevo)

- [x] Redis en `docker-compose.yml` (sin puerto publicado, contraseña `REDIS_PASSWORD`, persistencia AOF, `volatile-lru`) y check de salud `redis` (Degraded, nunca Unhealthy)
- [x] Caché de salida sobre Redis: catálogos (10 min, invalidados al escribir) y dashboard/reportes (15 s por usuario)
- [x] `ETag` + revalidación privada (`304`) en catálogos y en descargas de fotos
- [x] Cola duradera (Redis Streams) que comprime fotos a WebP con SkiaSharp (lado mayor 1600 px, calidad 80, sin EXIF/GPS, orientación aplicada) **sin hacer esperar la respuesta**
- [x] Barrida de recuperación (al arrancar y cada 6 h) que también comprime las fotos históricas
- [x] Degradación: sin Redis el API responde igual (sin caché y con fotos sin comprimir)

---

## Fase 12 — Despliegue en producción

> **Objetivo:** llevar el backend a producción (§29, §30).
> **Estado: ✅ Cerrada con alcance acotado (versión 1).** El usuario decidió desplegar **desde su propio equipo por Cloudflare Tunnel** y relajar el resto; cada ítem indica qué se hizo y qué pasa a v2.

### Resuelto en la versión 1

- [x] Dockerfile del backend reproducible: compila y la imagen se verificó (incluida la librería nativa de SkiaSharp en Debian con usuario sin privilegios)
- [x] Secretos y cadenas de conexión por variables de entorno para JWT, semillas, SMTP, almacenamiento y Redis (`.env` fuera de git; `.env.example` documentado)
- [x] Dominio y TLS por **Cloudflare Tunnel**: `api.yefrasoft.com` → `127.0.0.1:8088` (API, directo) y `by.yefrasoft.com` → `127.0.0.1:3000` (panel). Caddy ya no interviene (la referencia anterior a «Caddy → API» estaba obsoleta). Solo estos dos nombres se exponen públicamente
- [x] Despliegue desde este equipo: `api.yefrasoft.com` ya responde con las fases 1–9, verificada por `curl` con tokens de RH y gerente (lecturas y validaciones; sin escrituras sobre datos reales)
- [x] Panel web dentro del `docker-compose.yml` (servicio `frontend` en el `up` normal; el backend solo aporta la ruta de build y `FRONTEND_API_URL`; el `Dockerfile` es del equipo web)
- [x] Política de reinicio `restart: unless-stopped` en todos los servicios, para que el stack vuelva tras reiniciar el equipo
- [x] Procedimiento de despliegue y de rollback de imagen documentado (`progreso-backend.md` §32)

### Diferido a la versión 2 (no está hecho)

- [ ] Registry de imágenes y etiquetado por versión
- [ ] Hosting definitivo y TLS propio (hoy: este equipo + Cloudflare)
- [ ] CI/CD (build, test, deploy) por ambiente
- [ ] Backups de BD programados y restauración probada
- [ ] Smoke tests automatizados del despliegue y plan de rollback formal
- [ ] Carga de configuración inicial en producción documentada (tienda, zona, horarios, saldos, usuarios)
- [x] Contraseña de Postgres a variable y cierre de puertos de Postgres/MinIO (hecho antes de 1.0.0)
- [ ] CORS por lista y cierre del `8080` (deuda aceptada) y `HEALTHCHECK`

> Los criterios de aceptación finales (§29) se consolidan en `plan-desarrollo-mvp.md`.

---

## Dependencias con otras apps

- Fase 2: el móvil necesita `self-register` y `auth` (cerrados).
- Fase 4: el checador móvil ya consume `attendance/*`; el historial móvil y la consulta web usan `GET /attendance` (cerrados).
- Fase 4 (D6 y D7, 23-oct-2026): el móvil debe manejar el `409` «Fuera de horario», mostrar `ventanaMarcacion` de `GET /attendance/today`, y para marcas sin conexión enviar `registradaEnDispositivoAt` con zona horaria, reenviar con el mismo UUID y en orden de captura. El historial móvil con foto sigue pendiente en `plan-movil.md`.
- Fase 5 y 6 (12-nov-2026): web y móvil ya pueden consumir `/incidents`, `/attendance-corrections` y `/leave-requests` (ver [[rh-cotla-integracion-clientes-manual]]). La solicitud de ausencia es `multipart/form-data` y la consulta de notificaciones llega en la Fase 10.
- Fases 7 a 11 (13-nov-2026): web y móvil ya pueden consumir horas extra, reportes, dashboard y calendario, notificaciones (bandeja y correo; panel por SSE), auditoría, y las respuestas con `ETag`/`304` y fotos que pasan a WebP (ver `integracion-clientes.md`). **Están en el código; llegan a `api.yefrasoft.com` al ejecutar `docker compose up -d --build`.**
- Panel web: ya tiene `Dockerfile`; se levanta con el `docker compose up -d --build` normal.
- Push móvil (FCM): descartado por ahora; el móvil se entera por correo y consultando la bandeja.
- `PasswordReset:BaseUrl` se configura con `PASSWORD_RESET_BASE_URL` (en `.env` apunta al panel web, `by.yefrasoft.com/reset-password`). Falta decidir cómo llega el enlace a la app móvil (`appmovil://reset-password` o una URL por entorno).
