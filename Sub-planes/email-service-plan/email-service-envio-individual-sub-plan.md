---
tipo: sub-plan
titulo: Slice 7 — Envío 1 a 1 y worker SMTP
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-11-10
estado: en-progreso
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - envio
  - smtp
  - rabbitmq
  - grpc
  - redis
  - slice-7
---

# Slice 7 — Envío 1 a 1 y worker SMTP

> **Plan padre:** [[email-service-plan]]
> Enviar un correo individual con una plantilla, de punta a punta: petición (gRPC o REST), cola, SMTP y estado.

## Pendientes

- [x] Definir alcance con Efraín (fase de detallado, 2026-11-03): solo este slice; SMTP nativo; 3 reintentos y error propio por cuota de Google.
- [x] Aclaración del 2026-11-05: **las apps consumidoras envían por gRPC** (REST se conserva para el panel y curl) y la idempotencia vuelve a **Redis** con el patrón de Stripe (decisiones 10 a 13).
- [x] Esquema: `messages.attempts` y `messages.id_consumer` nullable (envíos del administrador). (C1)
- [x] Bloque CREAR C1 a C6: core, contratos, servicios, repositorio, SMTP, controller REST y tipos GraphQL (sin enlazar).
- [x] C7 Idempotencia con Redis y C8 servicio gRPC (sin enlazar).
- [x] Bloque MODIFICAR (M1 a M4, más M3b: compose de Swarm) y ENLAZAR E1 (backend) y E2 (panel). **Verificado de punta a punta en local el 2026-11-05** (REST, gRPC con grpcurl, idempotencia con Valkey, reintento por cola de espera, GraphQL).
- [x] Panel: `/envios` (solo Admin) con historial, filtros y "Enviar prueba" (por REST). Pendiente la prueba manual con sesión de Admin en el navegador (la hace Efraín).
- [x] Documentación D (2026-11-05): wiki (API con la sección gRPC y el `.proto`, Mensajería, Arquitectura, Modelo de datos, Reglas, Seguridad, Despliegue con staging y el stack `servicios`, Capas, Panel), README y AGENTS.md; esta nota y el plan padre. La wiki la sube Efraín.
- [x] Slice 7b, panel para consumidores (decisión 15): B1 y B2 (backend, verificado con curl), P1 y P2 (panel, `check`/`lint`/`format`/`build` en verde) hechos el 2026-11-05; D (docs e Insomnia) en curso. Falta la prueba manual en el navegador con keys de consumidor (la hace Efraín).
- [ ] Deploy a staging: `ALTER` en la base externa, borrar la cola clásica, stack `servicios` (`SERVICIOS_*`) y variables nuevas de staging (`EMAIL_SENDER_SMTP_*`, `EMAIL_SENDER_VALKEY_PASSWORD`, `EMAIL_SENDER_GRPC_PORT`), compose `compose.staging.yml` en Dokploy (hoy `infra/perfiles/compose.testing.yml`, rama `QA`; ver [[flujo-ci-cd-manual]]), PR a `staging`, nota QA.
- [ ] Deploy a testing por pipeline (PR a `staging`), nota QA al día y fase de dudas.

## Estado Actual

| Área | Estado |
|------|--------|
| Publicador RabbitMQ con confirmaciones y `mandatory` | ✅ |
| Consumidor de `notificacion.email.individual` con ack manual | ✅ |
| `LoggingEmailIndividualHandler` (solo escribe en el log) | ⚠ provisional, se reemplaza en este slice |
| Tabla `messages`, rol `Sender` (política y `MessagePermissions.RequireSend`), `TemplatePermissions.RequireUseAsync`, `ITemplateRenderer`, `ITemplateHtmlSanitizer` | ✅ |
| CREAR C1–C6: esquema, `Message` con `attempts`, excepciones, `MessageStatus`, `MessageIdentity`, `EmailRetryPolicy`, contratos, `MessageService`/`MessageQueries`/`MessageDispatchService`, `MessageRepository`, `SmtpEmailSender`, `MessageController`, tipos GraphQL | ✅ (sin enlazar) |
| Idempotencia con Valkey, servicio gRPC (puerto 8081), topología de reintentos con colas quorum, enlace y panel `/envios` | ✅ implementado y verificado en local (2026-11-05) |
| Documentación, deploy a staging (stack `servicios` + `compose.staging.yml`; hoy `infra/perfiles/compose.shared-services.yml` + `compose.testing.yml`) y fase de dudas | ⏳ |

## Decisiones (fase de detallado, 2026-11-03)

1. **Alcance:** solo este slice. Fuera: grupos y lotes (slice 8), píxel, baja y `List-Unsubscribe` (slice 9), adjuntos (v2). Redis entra solo para idempotencia (decisión 11); gRPC entra por las decisiones 10, 11 y 13.
2. **SMTP nativo** con `System.Net.Mail.SmtpClient`, sin paquetes nuevos. Hoy Google (`smtp.gmail.com:587`, STARTTLS, app password); después Stalwart u otro servidor. Modularidad: puerto `IEmailSender` (`Application/Messages`) con `EmailEnvelope`; implementación `SmtpEmailSender` + `SmtpOptions` en `Infrastructure/Email`; registro en un solo lugar (`Platform/Email/SmtpServiceCollectionExtensions.AddSmtpEmail`). Cambiar de servidor = cambiar secretos; cambiar de método = una implementación nueva y una línea de registro. Configuración: sección `Smtp` y secretos `SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM`, `SMTP_FROM_NAME`, `SMTP_USER`, `SMTP_PASSWORD` (mismo patrón que RabbitMQ). En local, **Mailpit** en `compose.yml` (SMTP 1025 sin TLS, UI en `http://127.0.0.1:8025`).
3. **Id del mensaje determinista** (regla 8 del plan padre). El `MessageId` son los primeros 16 bytes del SHA-256 de `"{consumidor | admin}:{Idempotency-Key}"` (se descartó el UUID v5: nadie fuera de la API recalcula el id); el worker inserta con `ON CONFLICT DO NOTHING` y no reenvía lo que ya está en estado terminal. Sin key → id aleatorio. La API **nunca escribe** `messages`. ~~Sin Redis~~: el 2026-11-05 se decidió que **Redis vuelve como caché de idempotencia** (decisión 12) y el id derivado queda como **red de seguridad**: si Redis pierde la clave, el correo sigue sin duplicarse.
4. **Reintentos: 3**, con esperas de 1, 5 y 15 minutos en colas de espera con TTL (`notificacion.email.individual.retry.1m/5m/15m`) que devuelven el mensaje a la cola principal; header AMQP `x-attempt`. Al agotarse, `FAILED` con `last_error` y copia en `notificacion.email.individual.dlq`. Errores permanentes (destinatario inválido, plantilla inexistente o inactiva, variables obligatorias sin valor, rechazo 5xx del servidor) → `FAILED` directo. **Cuota de Google agotada** → excepción propia `InSmtpQuotaExceededException` (código `SmtpQuotaExceeded`, log Error), mensaje `FAILED` con ese `last_error` y copia en la dlq; pausar el consumidor y reenviar desde la dlq quedan pendientes.
5. **Envíos del administrador:** `messages.id_consumer` pasa a nullable (`NULL` = envío del Admin), como `groups.id_manager`; así el panel envía pruebas con la key de Admin.
6. **Contrato REST:** `POST messages` `{ to, template (nombre), variables }`, header `Idempotency-Key` opcional (≤ 64 caracteres sin control), política `Sender`, rate limit `Relaxed`. Validaciones: rol (403 `RoleNotAllowed`), plantilla por nombre (404 `TemplateNotFound`), activa (409 `TemplateInactive`), asignación `MESSAGE` (403 `ResourceNotGranted`), variable no declarada u obligatoria sin valor (400 con la lista), valor de más de 1000 caracteres o con caracteres de control (400). Respuesta 202 `{ messageId }`. La cola lleva el `TemplateId` resuelto por la API. Destinatario normalizado con `MessagePreparer` (`IEntityPreparer<Message>`), como el resto de entidades.
7. **Entidades a mano** (como el slice 6): `Message.cs` y el bloque `Message` de `EmailDbContext.cs` con el estilo del scaffold.
8. **Worker:** vuelve a sanear el HTML tras sustituir variables (pendiente del slice 3) y genera un texto plano alternativo para el `AlternateView`. Un `SmtpClient` por envío; consumidor serial (una instancia).
9. **Consulta interna:** `GET messages/{id}` para `GroupManager` y Admin; lista GraphQL `messages` (filtros por consumidor, estado, correo y fechas) solo Admin, como todo GraphQL hoy. `MessageResponse` con `consumerId`, `templateId`, `email`, `status`, `attempts`, fechas y `lastError`; nunca el cuerpo ni las variables.

## Decisiones añadidas el 2026-11-05 (gRPC y Redis)

Surgieron con C1–C6 ya hechos. Efraín aclaró que las apps consumidoras hablan con el servicio por gRPC (`APP → gRPC → API → RabbitMQ → worker`); el panel sigue por REST/GraphQL. No estaba en la wiki ni aquí (el csproj traía `Grpc.AspNetCore` sin usar desde el primer commit).

10. **Transporte del envío.** gRPC es la puerta de las apps: hoy `SendMessage` (1 a 1); `SendToGroup` (lotes) en el slice 8. **Se conserva `POST messages` REST** para el botón "Enviar prueba" del panel y para curl. Las dos puertas delegan en el mismo `IMessageService.SendAsync`; `GET messages/{id}` y GraphQL `messages` siguen como consulta interna.
11. **gRPC sin TLS = puerto propio HTTP/2.** Kestrel no negocia HTTP/2 sin TLS en un endpoint `Http1AndHttp2` (documentación oficial: sin TLS todas las conexiones caen a HTTP/1.1 y gRPC falla). La API escucha en dos puertos: `8080` HTTP/1.1 (REST, GraphQL, salud; sin cambios) y **`8081` solo HTTP/2 (h2c) para gRPC** (el Dockerfile ya lo exponía). Local `127.0.0.1:5039`; testing `EMAIL_SENDER_GRPC_PORT` (5039; 5038 es el panel). En producción Traefik termina TLS y habla `h2c` con 8081. Autenticación: la app manda **`x-api-key` en los metadata** (son cabeceras HTTP/2: el handler de API key no cambia y aplican las mismas políticas; 401/403 llegan como `Unauthenticated`/`PermissionDenied`). Rate limit `Relaxed` por atributo. Errores: un `Interceptor` traduce las excepciones de dominio con `ErrorDescriptors` a `RpcException` (400 `InvalidArgument`, 401 `Unauthenticated`, 403 `PermissionDenied`, 404 `NotFound`, 409 `FailedPrecondition`, 429 `ResourceExhausted`, 502/503 `Unavailable`, resto `Internal`) con trailers `code` y `trace-id`. Validación: el servicio gRPC mapea el proto al `SendMessageRequest` existente y lo valida con `Validator.TryValidateObject` (la validación automática solo cubre minimal APIs).
12. **Idempotencia con Redis (patrón Stripe, [stripe.com/blog/idempotency](https://stripe.com/blog/idempotency))**, como describía la wiki original. Clave `idem:{consumerId|admin}:{key}`, valor `{messageId}:{huella}`, `SET NX EX` con TTL `Idempotency:TtlHours` (24). Flujo en `MessageService.SendAsync` (REST y gRPC comparten): si hay key, `GET` en Redis; existe y la huella coincide → 202 con el mismo `messageId` **sin reprocesar**; existe y difiere → **409 `IdempotencyKeyReused`**; no existe → validar y publicar → `SET NX` (si otro concurrente ganó, se compara la huella igual). Huella = SHA-256 del cuerpo normalizado (`to` en minúsculas, `template`, variables ordenadas). **Redis caído y hay key → 503 `IdempotencyUnavailable`** (falla cerrada); sin key no se toca Redis. No se cachean errores. Red de seguridad: el id derivado + `ON CONFLICT` del worker. **Servidor: Valkey** (fork BSD de Redis, mismo protocolo y comandos; decidido el 2026-11-05): imagen `valkey/valkey:9.0-alpine`, servicio `valkey` en `compose.yml` (loopback, sin contraseña) y en staging (hoy en el stack compartido `compose.servicios.yml`, decisión 14; red interna, `--requirepass` con `EMAIL_SENDER_VALKEY_PASSWORD`, AOF). En el código y la configuración se conserva el nombre del protocolo: opciones `Redis:Host`, `Redis:Port`, secreto `REDIS_PASSWORD` (opcional); paquete `StackExchange.Redis`; `RedisHealthCheck` en `ready`. Puerto `IIdempotencyStore` en `Application/Messages`.
13. **Contrato `.proto`** en `Src/Api/Messages/Protos/notificaciones.proto` (es transporte, como el controller): `package notificaciones.v1`, servicio `Notificaciones { rpc SendMessage(SendMessageRequest) returns (SendMessageResponse) }`, `SendMessageRequest { to, template, map<string,string> variables, idempotency_key }` (la key viaja **como campo**, visible en el contrato), `SendMessageResponse { message_id }`. Las apps copian el archivo (se publica en la wiki). Sin reflexión gRPC (grpcurl usa `-proto`).

14. **Compose de Dokploy en Swarm** (2026-11-05, tarde). Dokploy gestiona varios servidores en Docker Swarm. `compose.testing.yml` pasa a **`compose.staging.yml`** (solo API y panel; el entorno se llama staging; desde el 2026-11-10 es `infra/perfiles/compose.testing.yml`) y se conecta por la red overlay externa `dokploy-network` a un stack compartido nuevo, **`compose.servicios.yml`** (hoy `infra/perfiles/compose.shared-services.yml`), que Efraín despliega a mano y reutilizará para otras apps: RabbitMQ en **clúster de 2 nodos** (`rabbitmq-1` y `rabbitmq-2`, un servicio por nodo con nombre fijo, `classic_config` en `infra/rabbitmq/rabbitmq.conf`, cookie Erlang y datos fijados a dos nodos Swarm) y **Valkey primario + réplica** (`valkey-primary`, `valkey-replica`; la API escribe en el primario; failover manual, sin Sentinel). En el código, **todas las colas pasan a `x-queue-type = quorum`** para replicarse en el clúster (en un solo nodo funcionan igual; una cola clásica ya creada hay que borrarla antes). Riesgo: con dos nodos una cola quorum necesita ambos vivos para escribir; con tres toleraría una caída. Variables nuevas en Dokploy: `SERVICIOS_RABBITMQ_USER`, `SERVICIOS_RABBITMQ_PASSWORD`, `SERVICIOS_RABBITMQ_COOKIE`, `SERVICIOS_VALKEY_PASSWORD`, `SERVICIOS_NODE_1`, `SERVICIOS_NODE_2` (stack `servicios`); en staging, `EMAIL_SENDER_RABBITMQ_HOST` y `EMAIL_SENDER_VALKEY_HOST` opcionales. Los compose locales no cambian.

15. **Panel para consumidores (slice 7b, 2026-11-05, antes del PR).** Los consumidores (`Sender`/`GroupSender`) entran al panel con su key y ven dos páginas: `/envios` solo con el formulario de envío (**sin lista ni estados**: no consultan `messages`; el `messageId` del toast es su comprobante) y `/mis-plantillas` (solo lectura: nombre, asunto, alcances, variables y vista previa de sus plantillas asignadas). Backend nuevo: `GET templates/assigned` y `POST templates/{id}/render` (autenticado; el permiso lo decide `TemplatePermissions`: `RequireListAssigned` solo consumidor; `RequireRenderAsync` Admin, `TemplateManager` o consumidor con la plantilla asignada en cualquier alcance). GraphQL y `GET messages/{id}` siguen cerrados a consumidores. Panel: `PanelAccess` con `consumerRoles` por página, `/envios` por identidad (`MessagesScreen` para Admin, `ConsumerSendScreen` para consumidores), `SendTestModal` con `loadTemplates` inyectado, `AssignedTemplatesScreen` + `TemplateRenderModal`. `GroupSender` deja el hueco: botón "Enviar a grupo" deshabilitado hasta el slice 8. Pasos: B1 (CREAR backend), B2 (ENLAZAR backend), P1 (panel API y acceso), P2 (pantallas), D (docs, Insomnia, Obsidian).

## Esquema (ALTER para testing)

```sql
ALTER TABLE messages ADD COLUMN attempts INT NOT NULL DEFAULT 0;
ALTER TABLE messages ALTER COLUMN id_consumer DROP NOT NULL;
```

## Alcance

Cubre el envío 1 a 1 por gRPC (apps) y REST (panel), y el seguimiento de su estado hasta `SENT` o `FAILED`, con reintentos y cola de fallidos. No cubre lotes ni grupos (slice 8), la apertura y las bajas (slice 9), adjuntos, el reenvío desde la dlq ni el bloqueo "en curso" por key. Solo se guardan metadatos: no se guarda el cuerpo del correo ni las variables.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]] (regla 8, diagrama D6): la API **valida y publica** → 202 con el `MessageId`; **el worker es dueño del registro**. El consumidor de RabbitMQ es un adaptador: llama a `IEmailIndividualHandler`, que abre un scope y delega en `IMessageDispatchService` (Application); según el resultado (`Sent | Skipped | Retry | Failed`) publica la copia a la cola de espera o a la dlq y confirma. `IEmailSender` e `IIdempotencyStore` son puertos con implementación SMTP y Redis. Ciclo CREAR → MODIFICAR → ENLAZAR con un sub-agente por paso; los commits los hace Efraín.

## Implementación

### CREAR (aditivo, sin registrar en DI)
- [x] C1 Esquema y entidad: `infra/db.sql`, `Core/Entities/Message.cs`, bloque `Message` de `EmailDbContext`.
- [x] C2 Core: excepciones (`InTemplateNotFoundException`, `InTemplateInactiveException`, `InSmtpQuotaExceededException`, `InSmtpTransientException`, `InSmtpRejectedException`), `MessageStatus`, `MessageIdentity`, `EmailRetryPolicy`, `EmailDeliveryOutcome`.
- [x] C3 `Application/Messages`: contratos (`SendMessageRequest`, `MessageAcceptedResponse`, `MessageResponse`, opciones de lista), builder, `MessagePreparer`, puertos (`IMessageRepository`, `IEmailSender`, `IPlainTextConverter`), `IMessageService`, `IMessageQueries`, `IMessageDispatchService`; `MessagePermissions` (`CanRead/RequireRead`, `CanList/RequireList`); `ITemplateRepository.FindByNameAsync`.
- [x] C4 Servicios: `MessageService` (valida y publica), `MessageQueries`, `MessageDispatchService` (flujo del worker).
- [x] C5 Infraestructura y Platform: `MessageRepository` (`INSERT … ON CONFLICT DO NOTHING`), `SmtpOptions` con validador, `SmtpEmailSender`, `AddSmtpEmail`.
- [x] C6 Api: `MessageController` (`POST messages` 202, `GET messages/{id}`) y tipos GraphQL (sin módulo todavía); `JsonRegistry` con las dos respuestas.
- [x] C7 Idempotencia con Redis: paquete `StackExchange.Redis`, `InIdempotencyKeyReusedException` (409), `InIdempotencyUnavailableException` (503), `IIdempotencyStore` + `IdempotencyEntry`, `IdempotencyFingerprint`, `RedisOptions`, `IdempotencyOptions`, `RedisIdempotencyStore`, `RedisHealthCheck`, `AddRedis` (sin llamar).
- [x] C8 gRPC sin módulo: `Src/Api/Messages/Protos/notificaciones.proto` y su `<Protobuf>` en el csproj, `GrpcStatusMap`, `DomainExceptionInterceptor`, `NotificacionesGrpcService` (`[Authorize(Sender)]`, `[EnableRateLimiting(Relaxed)]`).

### MODIFICAR (sin cambiar comportamiento)
- [x] M1 Contratos de `Core/Messaging`: rutas de espera y dlq, `x-attempt`, `HandleAsync` que devuelve el resultado; `DispatchEmailIndividualHandler`; handler de log adaptado.
- [x] M2 Topología (principal + 3 esperas + dlq, declarada igual por publicador y consumidor) y consumidor con publicación a espera/dlq.
- [x] M3 Tipos GraphQL en `GraphQLModule`, csproj (carpetas vacías), `compose.yml` (Mailpit y Valkey), `compose.app.yml` (SMTP, `Redis__Host=valkey`, puerto 5039), `.env.example`.
- [x] M3b Compose de Swarm (decisión 14): `compose.testing.yml` → `compose.staging.yml`, nuevo `compose.servicios.yml` e `infra/rabbitmq/`, colas quorum en `RabbitMqTopologyDeclarer`, referencias en `push-harbor.sh`, `ci.yml`, AGENTS y README.
- [x] M4 `MessageService` con `IIdempotencyStore` (flujo de la decisión 12); `Kestrel:Endpoints` en `appsettings` (8080/5037 HTTP/1.1 y 8081/5039 HTTP/2), secciones `Redis` e `Idempotency`; `launchSettings` sin `applicationUrl`.

### ENLAZAR
- [x] E1 Backend (verificado con curl y grpcurl el 2026-11-05; el `Dockerfile` pasa la etapa de build a `sdk:10.0` Debian porque el `protoc` de `Grpc.Tools` no arranca en Alpine): `AddSmtpEmail` y `AddRedis` en `Program.cs`, handler real en lugar del de log, `IMessageRepository`, `RedisHealthCheck` en `ready`, `GrpcModule` (`AddGrpc` con el interceptor), `MessageModule` (servicios, rutas REST, `MapGrpcService`, campo GraphQL); `ALTER` local, purga de la cola y verificación con curl y grpcurl.
- [x] E2 Panel: `api/messages`, `actions/messages`, `MessagesScreen`, `SendTestModal`, `pages/envios.astro`, `PanelAccess` (`/envios` solo Admin); `bun run check/lint/format/build` en verde. El panel usa REST; no necesita cliente gRPC. Falta la prueba manual con sesión de Admin.
- [x] D Documentación (2026-11-05): wiki (API con la sección gRPC y el `.proto`, Mensajería, Arquitectura con gRPC y Valkey, Modelo de datos con la idempotencia real, Reglas, Seguridad, Despliegue con staging, el stack `servicios`, el puerto 8081, Traefik `h2c`, el `ALTER` y los secretos, Capas, Panel), README, AGENTS (regla "API": gRPC para los envíos de las apps); esta nota y el plan padre. Pendiente: nota QA al desplegar.

### Infraestructura (Efraín)
- [ ] Secretos `SMTP_*` en OpenBao de testing (`secret/email-sender`) o variables `EMAIL_SENDER_SMTP_*` en Dokploy; sin ellos la API no arranca.
- [ ] Dokploy, stack `servicios` (`compose.servicios.yml`, hoy `infra/perfiles/compose.shared-services.yml` con las variables de `infra/.env.shared-services`; a mano): `SERVICIOS_RABBITMQ_USER/PASSWORD/COOKIE`, `SERVICIOS_VALKEY_PASSWORD`, `SERVICIOS_NODE_1/2`; desplegarlo antes que staging. Borrar las colas clásicas existentes (`notificacion.email.individual`) antes de arrancar la versión con colas quorum.
- [ ] Dokploy, staging: `EMAIL_SENDER_VALKEY_PASSWORD` (obligatoria, igual que la del stack), `EMAIL_SENDER_RABBITMQ_DEFAULT_USER/PASS` iguales a las del stack, y `EMAIL_SENDER_GRPC_PORT` si 5039 está ocupado; abrir el puerto gRPC a la red local. Producción: router de Traefik a 8081 con esquema `h2c`.
- [ ] Aplicar el `ALTER` en la base de testing antes del merge a `staging`; purgar la cola si tiene mensajes con la forma vieja.
- [ ] Local: `brew install grpcurl`.

## Criterios de Aceptación

- [x] Repetir la petición con el mismo `Idempotency-Key` (REST) o `idempotency_key` (gRPC) devuelve el mismo `messageId` sin reprocesar y no envía otro correo; con otro cuerpo responde 409 `IdempotencyKeyReused` (`FailedPrecondition` en gRPC); con Redis caído y key, 503 (`Unavailable`); sin key, se envía con normalidad.
- [x] Un consumidor sin el rol `Sender` recibe 403 `RoleNotAllowed` (`PermissionDenied`); plantilla inexistente 404, inactiva 409, no asignada 403 `ResourceNotGranted`; variables inválidas 400 con el detalle. Los mismos casos por gRPC devuelven el `StatusCode` de la tabla y el trailer `code`.
- [x] gRPC solo responde en el puerto 8081/5039 con `x-api-key` en los metadata; sin metadata, `Unauthenticated`.
- [x] Un correo enviado queda `SENT` con `attempts: 1` y `sentAt`; con el SMTP caído pasa por `RETRYING` (colas de 1, 5 y 15 min) y termina `FAILED` con el motivo y copia en la dlq; un error permanente va a `FAILED` sin reintentos.
- [x] El Admin envía pruebas desde el panel (`consumerId: null`); `GET messages/{id}` responde a `GroupManager` y Admin, 403 a un consumidor; la lista GraphQL solo al Admin.
- [x] Con RabbitMQ caído la API responde 502 `ExternalService` al enviar y se recupera sola.
- [x] `dotnet build` (0/0), `bun run check` y `bun run build` correctos; `health/ready` en 200 con `redis`.

> **Desviaciones conocidas (revisión 2026-11-09).** Criterios verificados en local el 2026-11-05 con estas diferencias, que no bloquean el cierre: el 403 por falta de rol lo da la política antes del servicio y llega sin `code` `RoleNotAllowed` (deuda anotada, no entra a la Fase 4); con Valkey caído el 503 sale con `code` `Internal` en vez de `IdempotencyUnavailable` (desde 500 no se expone el código de dominio); el health check de `ready` se llama `valkey`, no `redis`.

## Preguntas abiertas (resueltas)

- Proveedor SMTP: Google por ahora (app password), Stalwart u otro después; el puerto `IEmailSender` aísla el cambio.
- Adjuntos: no en este slice (v2).
- Reintentos: 3, con esperas de 1, 5 y 15 minutos; error propio al agotarse la cuota de Google.
- Transporte (2026-11-05): gRPC para las apps, REST para el panel; `.proto` en `Api`; key como campo del mensaje.
- Idempotencia (2026-11-05): Redis con el patrón de Stripe; 409 si se reutiliza la key con otro cuerpo.

## Riesgos y notas

- Los argumentos de una cola son inmutables: cambiar un TTL exige borrar la cola de espera.
- Mensajes viejos en la cola (forma JSON anterior) terminarían en `FAILED`: purgar antes de desplegar.
- Dos instancias podrían enviar dos veces (*at-least-once*); hoy hay una sola y el consumidor es serial. Mitigación futura: token de concurrencia `xmin`. **Entra a la Fase 4** (revisión 2026-11-09).
- Cuota de Google (500/día personal, 2000/día Workspace): al agotarse, todo lo encolado falla a la dlq; no hay reenvío automático. **Entra a la Fase 4:** pausa del consumidor por cuota agotada y reenvío desde la dlq.
- Dos peticiones **simultáneas** con la misma key pueden publicar las dos (no hay bloqueo "en curso" como en Stripe): el worker las funde por el id derivado. **Entra a la Fase 4:** bloqueo "en curso" antes de publicar.
- El rate limit particiona por IP también en gRPC (la autenticación corre después del limitador): apps detrás del mismo NAT comparten cubo. Deuda anotada (no entra a la Fase 4).
- `ISentLedger` vive en memoria (no sobrevive a un reinicio ni se comparte entre réplicas) y, si falta `x-attempt`, el número de intento del log sale de la cabecera y no de la fila. **Entra a la Fase 4:** `ISentLedger` en Valkey e intento desde la fila.
- gRPC sin TLS en local y testing (red interna); en producción la TLS la termina Traefik.
- Matices vistos en la verificación (2026-11-05): el 403 por falta de rol lo da la política antes del servicio, así que llega como `Forbidden` genérico (no `RoleNotAllowed`), igual que el resto de rutas; el 503 sin Valkey sale con código `Internal` porque desde 500 el proyecto no expone el código de dominio (la causa va al log). En macOS, `/bin/sh` (bash 3.2) rompe las comillas escapadas dentro de `"$(…)"`: los JSON de los scripts de prueba se construyen en una variable.

## Relacionado

- [[email-service-plan]]
- [[email-service-plantillas-sub-plan]]
- [[email-service-envio-grupal-sub-plan]]
- [[email-service-datos-maestros-testing]]
