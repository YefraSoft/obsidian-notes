---
tipo: sub-plan
titulo: "Fase 1 — Fundaciones, contactos, cola de Odoo y JWT"
autor: "Efraín García"
creado: 2026-10-27
actualizado: 2026-10-28
estado: en-progreso
proyecto: nummex
padre: "[[nummex-backend-v2-plan]]"
tags:
  - nummex
  - sub-plan
  - backend-v2
  - fase-1
  - jwt
  - odoo
  - rate-limiting
---

# Fase 1 — Fundaciones, contactos, cola de Odoo y JWT

> **Plan padre:** [[nummex-backend-v2-plan]]
> Base técnica de `nummex-backend-v2` (errores, datos, rate limiting), alta de contactos y prospectos con sesión, sincronización asíncrona con Odoo mediante `Channel<T>` + `BackgroundService`, y autenticación con usuarios y JWT. Cubre los Slices 1 a 4 del plan.

## Pendientes

- [x] Valores de rate limiting fijados: Relaxed 300 por 60 s, Moderate 60 por 60 s, Strict 5 con reposición de 1 cada 12 s y StrictWindow 5 por 900 s por IP (en `appsettings.json`; se pueden ajustar sin tocar código).
- [ ] Vigencia del access token y del refresh token.
- [ ] Política mínima de contraseña.
- [ ] Definir si `POST /auth/register` solo sirve para prospectos con sesión o también crea usuarios de backoffice.
- [ ] Transiciones y significado de cada valor de `prospects.funnel`.
- [ ] Límite por email (además de por IP) en la recuperación de sesión: exige leer el cuerpo antes de limitar; queda fuera de esta fase.
- [ ] Aceptado: los refresh tokens viven en `IMemoryCache`; un reinicio de la API obliga a iniciar sesión otra vez y solo sirve con una instancia. Si v2 escala, pasan a caché distribuida o a tabla.
- [ ] Convivencia con v1: v2 expone `/api/...` igual que v1 (`App:Path = /api`); mientras v1 siga en producción, Caddy debe enrutar v2 con otro criterio (puerto, subdominio o prefijo).
- [ ] `apply-migration.sh` sigue buscando la migración EF de v1 (`__EFMigrationsHistory`); no aplica el `initial.sql` nuevo. Definir el flujo de esquema de v2.
- [ ] Bases creadas con el `initial.sql` anterior: recrearlas o aplicar a mano `contacts.birth_date SET NOT NULL` y `contacts.id_odoo_contact DROP NOT NULL`. `dates.id_odoo_date` sigue `NOT NULL` hasta el Slice 8.
- [ ] Teléfono: hoy 10 a 15 dígitos (`[0-9]`). Edad mínima: hoy solo "anterior a hoy y no mayor de 120 años".
- [ ] Los 409 del alta revelan si un correo o teléfono existe (como v1); el límite `Strict` lo mitiga.
- [ ] La cookie vence a los 7 días, pero el `session_token` no expira en la base. La recuperación conserva el token existente.
- [ ] Las claves de `errors` del 400 salen en PascalCase (`Name`, `Phone`...); ajustar en `ValidationModule` si el frontend las espera en camelCase.
- [ ] Los títulos de `ProblemDetails` salen en inglés ("Not Found"); el `detail` sí va en español. Los mensajes de las excepciones ya no llevan envoltorio en inglés.
- [ ] `ProspectService` compila la proyección del builder en cada petición; optimizar solo si hace falta.
- [ ] Búsquedas por `lower(email)`: si crece la tabla, índice sobre `lower(email)`.

## Estado Actual

| Área | Estado |
|------|--------|
| Sistema de módulos (`IModule`, `IServiceInstaller`, `IEndpointMapper`, `IPipelineStep`, `ModuleRegister`) | ✅ |
| `GET /health` | ✅ |
| **Slice 1** — errores, `ProblemDetails`, validación, CORS | ✅ |
| **Slice 1** — rate limiting (Relaxed, Moderate, Strict, StrictWindow) | ✅ (límite global Moderate activo; Strict y StrictWindow se declaran en los slices 2 y 4) |
| **Slice 1** — scaffold de las 20 entidades, `NummexDbContext`, `IDataTransaction`, `PersistenceModule` | ✅ (la conexión real a la base se prueba con el primer endpoint, Slice 2) |
| **Slice 2** — contactos y prospectos, `session_token`, recuperación, `AuthModule` (cookie de sesión) | ✅ (verificado con curl contra Postgres: alta, 409, 400, 401/403, recuperación, 429 y cookie en Development y Production) |
| **Slice 1** — `ModuleRegister` corregido (un módulo con varias interfaces se registra en todas) | ✅ |
| **Slice 1** — CORS (`CorsModule`, `PipelineOrder.Cors = 150`) | ✅ |
| **Slice 2** — `initial.sql` editado: `contacts.birth_date` obligatoria e `id_odoo_contact` nulo | ✅ |
| **Slice 2** — rutas `POST /api/contacts`, `POST /api/prospects/session/recover`, `GET /api/prospects/{contactId}` | ✅ |
| **Slice 3** — `contacts.odoo_state`/`odoo_created_at`, `prospects.id_odoo_lead`, `dates.id_odoo_date` nulo en `initial.sql` | ✅ |
| **Slice 3** — abstracción HTTP compartida (`IExternalApiClient`, clase base, estrategia de autenticación) | ✅ |
| **Slice 3** — cliente de Odoo (JSON-2), cola en memoria (`Channel<Guid>`), procesador idempotente y `OdooSyncWorker` | ✅ (verificado con Odoo simulado; parámetros JSON-2 por confirmar con el Odoo real) |
| **Slice 4** — usuarios, JWT, refresh, políticas por permiso | ⏳ |

## Alcance

Cubre las fundaciones técnicas, el alta y la sesión del prospecto, la cola de Odoo para contactos y el acceso con usuarios y JWT. **No cubre** cotización con Libro Azul, datos de solicitud, documentos ni citas (Fase 2), ni clientes, préstamos, pagos y cobranza (Fase 3), ni el chat (Fase 4).

## Forma de trabajo

Sigue las **Reglas de Trabajo** y las **Reglas de Negocio** de [[nummex-backend-v2-plan]]. Cada slice se parte en **CREAR → MODIFICAR → ENLAZAR**, con visto bueno del responsable entre bloques; los commits los hace él. Las piezas técnicas se portan de `EmailSendService` (solo lectura, `/home/yefrasoft/proyects/proyecto-ejemplo/EmailSendService/Src/`) con los namespaces `nummex_backend.*` y sin su código de dominio de email. Los agentes no escriben tests.

**Cómo se ejecuta cada paso.** Cada paso lo ejecuta un sub-agente Haiku con un prompt autocontenido (rutas, firmas, reglas y qué no tocar). Quien coordina revisa el diff, corre `dotnet build` (0 errores y 0 advertencias), verifica con curl cuando hay endpoints y propone el mensaje de commit (sin `Co-Authored-By`); no se lanza el siguiente paso sin el visto bueno del responsable. Las clases que implementan `IModule` se activan al crearse, por eso los módulos solo se crean en ENLAZAR. De `EmailSendService` se porta únicamente lo que v2 usa.

Decisiones propias de esta fase:

- `birth_date` es **obligatoria** al crear el contacto, para que todo prospecto pueda recuperar su sesión.
- Los **refresh tokens** se guardan en `IMemoryCache` (solo el hash del token opaco), con rotación en cada uso y revocación al cerrar sesión o desactivar al usuario. No hay tabla ni `ALTER`.
- La contraseña se hashea con `PasswordHasher<T>` de ASP.NET (PBKDF2), sin paquetes extra.
- El rate limiting es el de EmailSendService (Relaxed, Moderate y Strict por token bucket, con Moderate como límite global) más una variante **sin bucket**, `StrictWindow`, de ventana fija por IP. Sustituye el "30 solicitudes por minuto por IP" de v1.
- Los trabajos de Odoo corren con `Channel<Guid>` + `BackgroundService` **sin tabla de trabajos**: el estado durable está en `contacts.odoo_state` (`PENDING | SYNCED | FAILED`) y `odoo_created_at`, y el worker rehidrata los `PENDING` al arrancar y con un barrido periódico.
- **Odoo 20 se consume por JSON-2** (`/json/2/<modelo>/<método>`, bearer + `X-Odoo-Database`), a través de una abstracción HTTP compartida con Libro Azul (`IExternalApiClient`).
- **Idempotencia en Odoo:** antes de crear se busca (partner por correo; lead por partner, tipo y `opportunity`) y se reutiliza lo existente; cada id se guarda al obtenerlo.
- Un contacto `FAILED` no se reintenta solo: se vuelve a poner `PENDING`.
- **Sesión del prospecto (Slice 2):** esquema de autenticación propio `ProspectSession` (cookie `nummex.session` = `session_token`, 7 días). Vive en el mismo `AuthModule` que recibirá JWT en el Slice 4.
- **Rutas bajo `/api`** (igual que v1), con la ruta base en `App:Path`.
- **Esquema por edición directa de `initial.sql`** mientras v2 no tenga base desplegada: `contacts.birth_date` obligatoria e `id_odoo_contact` nulo hasta que Odoo confirme; el modelo EF se edita a mano (sin re-scaffold).
- **El `funnel` no tiene endpoint:** queda en 1 en el Slice 2; lo suben los slices de la Fase 2 y el Slice 4 pone `completed_at`.
- **Scaffold de las 20 tablas** (no solo las de esta fase), con un Postgres temporal en Docker que se borra al terminar.

## Implementación

### Slice 1 — Fundaciones ✅

Ejecutado en los pasos 1.1 a 1.5 (CREAR), 2.1 (MODIFICAR) y 3.1 a 3.4 (ENLAZAR).

**Bloque 1 — CREAR (aditivo, sin registrar en DI)**
- [x] `Core/Exceptions/`: `DomainException` e `InfrastructureException`; `Auth/` (`AuthException`, `InLoginException`, `InForbiddenException`), `Requests/` (`RequestException`, `InValidationException`, `InNotFoundException`, `InConflictException`), `Configuration/InConfigurationException`, `ExternalServices/InExternalServiceException`. El nivel de log vive en la clase base. No se portaron `InRoleNotAllowed` ni `InResourceNotGranted` (son de email).
- [x] `Platform/Errors/` (`ErrorDescriptor`, `ErrorDescriptors`: por debajo de 500 se muestra el mensaje; desde 500, detalle genérico) y `Platform/Config/ConfigValidation`.
- [x] `Platform/Validations/`: `EnumNames`, `IsEnumAttribute`, `IsEnumListAttribute`. No se portó `FutureDateAttribute`.
- [x] `Application/Common/IDataTransaction` (`CommitAsync` y `RunAsync`). No se portaron `PagedResult`, `PageLimits` ni `SortDirection` (son de GraphQL).
- [x] `Api/RateLimiting/`: `RateLimits`, `RateLimitingOptions`, `PartitionKeys` (`ByUserOrIp`, `ByIp`), `LimitPolicy` (base con el 429 común), `Bucket/` (`BucketLimitPolicy`, `Relaxed`, `Moderate`, `Strict`) y la variante nueva `Window/` (`WindowLimitPolicy`, `StrictWindow`: ventana fija por IP, `QueueLimit = 0`, sin reposición gradual).
- [x] Paquetes: EF Core, `Relational`, `Design` y `Npgsql.EntityFrameworkCore.PostgreSQL` 10.0.3. El paquete de JWT se agrega en el Slice 4.
- [x] Scaffold de las **20 entidades** en `Core/Entities` y `Infrastructure/Persistence/NummexDbContext` (Postgres temporal en Docker, borrado al terminar). Columnas generadas por Postgres como solo lectura.
- [x] `Infrastructure/Persistence/DataTransaction` (la unicidad de Postgres se traduce a `InConflictException`, 409).

**Bloque 2 — MODIFICAR**
- [x] `appsettings.json`: `RateLimiting`, `Cors:AllowedOrigins` (vacío) y `ConnectionStrings:DefaultConnection` (vacía; la arma el compose desde `POSTGRES_*`).
- [x] `PipelineOrder.Cors = 150` (entre errores y rate limiting, para que los 429 lleven cabeceras CORS).
- [x] Corrección de `ModuleRegister`: tres `if` en lugar de un `switch`, para que un módulo con varias interfaces se registre en todas.

**Bloque 3 — ENLAZAR**
- [x] `ErrorHandlingModule` + `GlobalExceptionHandler` (`PipelineOrder.ExceptionHandling`): `ProblemDetails` con `code` y `traceId`; 4xx con mensaje y nivel de la excepción, 5xx genérico y `Error` con excepción.
- [x] `RateLimitingModule` (`PipelineOrder.RateLimiting`): 429 con `Retry-After`; Moderate como límite global (incluidos los 404).
- [x] `ValidationModule` (`AddValidation()`): 400 con `errors` por campo y `code: Validation`.
- [x] `CorsModule`: orígenes de `Cors:AllowedOrigins` unidos a `CORS_ALLOWED_ORIGINS` (CSV), con credenciales; sin orígenes no se permite ninguno.
- [x] `PersistenceModule`: `TimeProvider.System`, `NummexDbContext` (`DefaultConnection`) e `IDataTransaction`.
- [x] Verificado: `dotnet build` 0/0; `GET /health` 200; ruta inexistente 404 con `ProblemDetails`; 70 peticiones seguidas dan 60 respuestas 200 y 10 con 429 con `Retry-After`; preflight de CORS con origen permitido y no permitido, y con la variable CSV; arranque sin errores de DI.

### Slice 2 — Contactos y prospectos ✅

Ejecutado en los pasos 1.1 a 1.4 (CREAR), 2.1 (MODIFICAR) y 3.1 a 3.3 (ENLAZAR).

**Bloque 1 — CREAR**
- [x] Contratos (`Application/Prospects/Contracts`): `CreateContactRequest` (`name` ≤ 150, `phone` 10 a 15 dígitos `[0-9]`, `email` válido ≤ 150 y guardado en minúsculas, `birthDate` obligatoria), `RecoverSessionRequest`, `ProspectResponse` y `ProspectSession` (el token solo viaja en la cookie, nunca en el cuerpo). `Platform/Validations/BirthDateAttribute`: anterior a hoy y no mayor de 120 años.
- [x] Puertos: `IContactRepository` (`EmailExistsAsync`, `PhoneExistsAsync`, `Add`), `IProspectRepository` (`SessionTokenExistsAsync`, `FindBySessionTokenAsync`, `FindByRecoveryAsync`, `FindAsync` con proyección), `IProspectService`, `IProspectQueries`, `SessionActor` y `ProspectPermissions.RequireOwner` (403). `IResponseBuilder` y `ProspectResponseBuilder`. No hay `IContactService`: el alta es un caso de uso del prospecto.
- [x] `Core/Security/SessionToken`: 16 caracteres alfanuméricos con `RandomNumberGenerator`.
- [x] `ProspectService` (alta con un solo `CommitAsync`; la recuperación conserva el token y solo genera uno si faltara; 404 genérico) y `ProspectQueries`.
- [x] Repositorios EF: `ContactRepository` y `ProspectRepository`.
- [x] `Platform/Auth`: `AuthSchemes`, `AuthClaims`, `SessionCookie` y `ProspectSessionHandler`; `Api/Prospects/ProspectEndpoints`.

**Bloque 2 — MODIFICAR**
- [x] `db-migrations/initial.sql`: `contacts.birth_date DATE NOT NULL` e `id_odoo_contact BIGINT UNIQUE` (sin `NOT NULL`). Verificado en un Postgres temporal: varios contactos con `id_odoo_contact` nulo conviven y un alta sin `birth_date` falla.
- [x] `Contact`: `BirthDate` pasa a `DateOnly` e `IdOdooContact` a `long?`; ajustes en el builder y el servicio. `appsettings.json`: `App:Path = /api`.
- [x] Corrección de la regex del teléfono (`[0-9]`) y de los mensajes de las excepciones, que ya no llevan envoltorio en inglés.

**Bloque 3 — ENLAZAR**
- [x] `PersistenceModule` registra los repositorios; `AuthModule` (`IServiceInstaller` + `IPipelineStep`, `PipelineOrder.Auth`): esquema `ProspectSession`, política `Prospect`, `UseAuthentication` y `UseAuthorization`.
- [x] `ProspectsModule`: servicios, queries y builder en DI, y rutas bajo `App:Path`.
- [x] `POST /api/contacts` (público, `Strict`): contacto y prospecto con `funnel = 1` en un solo guardado; 409 si el correo o el teléfono ya existen; cookie `nummex.session` (`HttpOnly`, `Path=/api`, 7 días; `SameSite=None; Secure` fuera de Development y `Lax` en Development).
- [x] `POST /api/prospects/session/recover` (`StrictWindow`): restaura la cookie con el mismo token; si no coincide, 404 genérico (no revela qué dato falló); no distingue mayúsculas en el correo.
- [x] `GET /api/prospects/{contactId}` (política `Prospect`, `Moderate`): un prospecto solo ve su propio registro (otro id responde 403; sin cookie o con una desconocida, 401).
- [x] Verificado con curl contra Postgres (16 puntos): alta 201 y cookie; 409 por correo y por teléfono; 400 con `errors`; 429 al sexto alta seguido y al sexto intento de recuperación; GET propio 200, ajeno 403, sin cookie 401; recuperación correcta e incorrecta; en base, 2 contactos y 2 prospectos sin filas huérfanas, `id_odoo_contact` nulo, correos en minúsculas y token de 16; cookie en Production con `secure` y `samesite=none`.
- [ ] No hecho (por decisión): avance de `funnel` y `completed_at`; lo suben los slices de la Fase 2 y el Slice 4.

### Slice 3 — Cola de Odoo ✅

Ejecutado en los pasos 1.1 a 1.3 (CREAR), 2.1 (MODIFICAR) y 3.1 a 3.2 (ENLAZAR), con un ajuste posterior (idempotencia, logs). **Cambio frente al plan original:** no hay tabla `odoo_sync_jobs`. La cola vive en memoria de proceso y el estado durable está en `contacts.odoo_state` y `contacts.odoo_created_at`, más los ids de Odoo (`id_odoo_contact`, `prospects.id_odoo_lead`).

**Bloque 1 — CREAR (aditivo, sin registrar en DI)**
- [x] `initial.sql`: `contacts.odoo_state` (`PENDING | SYNCED | FAILED`, por defecto `PENDING`, con `CHECK`), `contacts.odoo_created_at` (último intento de crear el contacto en Odoo, haya salido bien o no, para ubicarlo en los logs), índice parcial `idx_contacts_odoo_state_pending`, `prospects.id_odoo_lead` (único, nulo) y `dates.id_odoo_date` nulo. Modelo EF editado a mano (`Contact`, `Prospect`, `Date`, `OdooSyncState`).
- [x] `Application/External/`: abstracción HTTP compartida (`IExternalApiClient`, `ExternalApiRequest`, `ExternalApiResponse`, `ExternalApiMethod`, `IExternalAuthStrategy`); oculta `HttpClient` y la comparten Odoo y, después, Libro Azul.
- [x] `Application/Odoo/`: `IOdooClient` (`CreateAsync`, `WriteAsync`, `FindIdAsync`), `IOdooSyncQueue` (`TryEnqueue`, `ReadAllAsync`), `IOdooSyncRepository`, `IOdooSyncProcessor`, `OdooSyncContact`, `OdooLeadStage` (9 etapas reales de `x_studio_ultimo_estado`, verificadas con el MCP de Odoo).
- [x] `Infrastructure/External/`: `ExternalApiClientBase` (ciclo de envío: un reintento ante red, timeout o 5xx; un reintento ante 401 tras invalidar la autenticación; errores como `InExternalServiceException` sin cuerpos ni cabeceras; logs a nivel de error), `ExternalRequestFactory`, `ExternalResponseReader` y `HttpExternalApiClient` (resuelve el cliente con nombre y la estrategia por servicio).
- [x] `Infrastructure/Odoo/`: `OdooOptions`, `OdooAuthStrategy` (bearer + `X-Odoo-Database`), `OdooClient` (**API JSON-2 de Odoo 20**: `POST /json/2/<modelo>/<método>`), `OdooSyncQueue` (`Channel<Guid>` acotado), `OdooSyncProcessor`, `OdooSyncWorker` y `Persistence/Repositories/OdooSyncRepository`.

**Bloque 2 — MODIFICAR**
- [x] `ProspectService.CreateAsync`: el contacto nace `PENDING` y, tras el único `CommitAsync`, hace `TryEnqueue(contactId)`; nunca llama a Odoo ni falla por él.

**Bloque 3 — ENLAZAR**
- [x] `Api/External/ExternalApiModule` (cliente compartido) y `Api/Odoo/OdooModule` (opciones, cliente HTTP `odoo` con `BaseAddress` terminada en `/`, estrategia, cliente, cola, repositorio, procesador y `AddHostedService<OdooSyncWorker>`). Sección `Odoo` en `appsettings.json` sin secretos.
- [x] Comportamiento del worker: consumidor sobre el canal; barrido al arrancar y cada 60 s que rehidrata los `PENDING`; reintentos con espera exponencial en memoria (2 s, 4 s… con tope); al agotar `MaxAttempts` el contacto queda `FAILED` y **no se reintenta solo** (se vuelve a poner `PENDING`); sin configuración de Odoo no consume intentos. Apagado ordenado.
- [x] Procesador idempotente: busca antes de crear (`search_read`: partner por correo; lead por partner + tipo `Préstamo` + `opportunity`) y reutiliza lo que exista; guarda cada id en cuanto lo tiene, con hasta 3 intentos de guardado; si el guardado falla, relanza la excepción original.
- [x] Verificado contra Postgres temporal y un Odoo simulado en puerto local: alta 201 en 0.015–0.8 s sin esperar a Odoo; `SYNCED` en ≤ 1 s; Odoo caído → reintentos a 2 s y 4 s y `FAILED` a los ~6.5 s; un `FAILED` no se rehidrata; un `PENDING` sí tras reiniciar; reutilización de partner y lead existentes sin crear duplicados; 401 con `FAILED` y sin la key en logs; sin Odoo configurado el alta sigue en 201 y queda `PENDING`; logs sin keys, contraseñas ni datos personales ni stack traces del `HttpClient`; contactos y prospectos 1 a 1.
- [ ] Pendiente: los nombres de parámetros de JSON-2 (`vals_list`, `vals`, `ids`, `domain`, `fields`, `limit`) siguen sin confirmar contra el Odoo real; la key del MCP solo sirve para el MCP (401 en `/json/2`), el worker necesita una key de JSON-2 y el nombre de la base.
- [ ] Pendiente: consulta interna de contactos `PENDING`/`FAILED` (`WHERE odoo_state <> 'SYNCED'`); se mapea en el Slice 4 con su permiso.

### Slice 4 — Usuarios y JWT

**Paso 1 — CREAR**
- [ ] Puertos: `IUserRepository`, `IAuthService`, `ITokenService`, `IRefreshTokenStore` (implementado sobre `IMemoryCache`).
- [ ] `JwtOptions` (emisor, audiencia, clave, vigencia del access y del refresh; solo nombres de variable en `.env.example`) con validación al arrancar.
- [ ] Contratos: `RegisterRequest` (contraseña), `LoginRequest`, `TokenResponse`, `MeResponse`.
- [ ] `AuthService`: hash con `PasswordHasher<T>`, verificación con 401 uniforme (credenciales inválidas, usuario inexistente o `is_active = false` responden igual), actualización de `last_session`.
- [ ] Refresh opaco aleatorio: se guarda solo su hash en `IMemoryCache`, con rotación en cada uso (el anterior deja de valer) y revocación al cerrar sesión o desactivar al usuario.

**Paso 2 — MODIFICAR**
- [ ] `ProspectService`: marcar `completed_at` al registrar la contraseña y subir el `funnel` a su fase final.
- [ ] La consulta interna de la cola de Odoo (Slice 3) pasa a exigir el permiso correspondiente.

**Paso 3 — ENLAZAR**
- [ ] `AuthModule` (`IServiceInstaller` + `IPipelineStep`, `PipelineOrder.Auth`): JWT bearer, claims con `permissions` (`READ`, `EDIT`) y políticas por permiso.
- [ ] `POST /auth/register` (prospecto con sesión válida): crea el `user` con su hash y `completed_at`.
- [ ] `POST /auth/login` (`StrictWindow`), `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`.
- [ ] La contraseña nunca se registra ni se devuelve.
- [ ] Verificar con curl: registro, login correcto, contraseña incorrecta, usuario inactivo (mismo 401), refresh con rotación (el token anterior ya no sirve), logout, ruta protegida sin token (401) y con permiso insuficiente (403), y el 429 en login.

## Estructura actual de `Src/` (al cerrar el Slice 3)

```
Src/
├── Api/
│   ├── Auth/AuthModule.cs
│   ├── Cors/CorsModule.cs
│   ├── External/ExternalApiModule.cs
│   ├── Odoo/OdooModule.cs
│   ├── Errors/{ErrorHandlingModule, GlobalExceptionHandler}.cs
│   ├── Health/HealthModule.cs
│   ├── Prospects/{ProspectsModule, ProspectEndpoints}.cs
│   ├── RateLimiting/{RateLimitingModule, LimitPolicy, RateLimits, RateLimitingOptions, PartitionKeys}.cs
│   │   ├── Bucket/ (BucketLimitPolicy y Policies: Relaxed, Moderate, Strict)
│   │   └── Window/ (WindowLimitPolicy y Policies: StrictWindow)
│   └── Validation/ValidationModule.cs
├── Application/
│   ├── Builders/IResponseBuilder.cs
│   ├── External/ (IExternalApiClient, IExternalAuthStrategy, ExternalApiRequest, ExternalApiResponse, ExternalApiMethod)
│   ├── Odoo/ (IOdooClient, IOdooSyncQueue, IOdooSyncRepository, IOdooSyncProcessor, OdooSyncContact, OdooLeadStage)
│   ├── Common/IDataTransaction.cs
│   ├── Contacts/IContactRepository.cs
│   └── Prospects/ (servicio, queries, puertos, permisos, Contracts/, Builders/)
├── Core/
│   ├── Entities/ (20 entidades por scaffold)
│   ├── Exceptions/ (Auth, Requests, Configuration, ExternalServices)
│   └── Security/SessionToken.cs
├── Infrastructure/
│   ├── External/ (ExternalApiClientBase, HttpExternalApiClient, ExternalRequestFactory, ExternalResponseReader)
│   ├── Odoo/ (OdooOptions, OdooAuthStrategy, OdooClient, OdooSyncQueue, OdooSyncProcessor, OdooSyncWorker)
│   └── Persistence/ (NummexDbContext, DataTransaction, PersistenceModule, Repositories/ incl. OdooSyncRepository)
└── Platform/
    ├── Auth/ (AuthSchemes, AuthClaims, SessionCookie, ProspectSessionHandler)
    ├── Config/ConfigValidation.cs
    ├── Errors/ (ErrorDescriptor, ErrorDescriptors)
    ├── Module/ (IModule, IServiceInstaller, IEndpointMapper, IPipelineStep, PipelineOrder, ModuleRegister)
    └── Validations/ (EnumNames, IsEnum, IsEnumList, BirthDate)
```

## Criterios de Aceptación

- [x] `dotnet build` en `nummex-backend-v2` con 0 errores y 0 advertencias (al cerrar los slices 1 a 3).
- [x] Todo error responde `ProblemDetails` con `code` y `traceId`; 4xx con mensaje en español, 5xx con detalle genérico.
- [ ] Superar un límite responde 429 con `Retry-After` (verificado en el límite global, en `Strict` y en `StrictWindow` de la recuperación); falta que el login use `StrictWindow` (Slice 4).
- [x] `POST /api/contacts` crea contacto y prospecto en un solo guardado; email o teléfono repetido → 409.
- [x] La recuperación de sesión responde igual (404) si falla el email o la fecha.
- [x] Crear un contacto no espera a Odoo; el trabajo sobrevive a un reinicio y no se duplica en Odoo (verificado con Odoo simulado).
- [x] Con Odoo caído el trabajo reintenta; al agotar reintentos queda `FAILED` y es visible (`contacts.odoo_state`; la consulta interna llega en el Slice 4).
- [ ] Login: credenciales inválidas o usuario inactivo → el mismo 401; el refresh rota y el anterior deja de valer.
- [x] Un prospecto no puede leer a otro (403); aún no hay endpoints de modificación.
- [ ] Esquema al día en `db-migrations/initial.sql` (hecho: `contacts`, `prospects.id_odoo_lead` y `dates.id_odoo_date`) y variables nuevas (solo nombres) en `.env.example` (las de Odoo ya existen; faltan las de JWT). Los tests los agrega el responsable.

## Relacionado

- [[nummex-backend-v2-plan]]
- [[nummex-app-plan]]
- [[AGENTS]]
