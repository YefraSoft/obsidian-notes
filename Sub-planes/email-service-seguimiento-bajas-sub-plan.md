---
tipo: sub-plan
titulo: Fase 4 — Seguimiento, bajas, deuda y producción (slice 9 y v1.0.0)
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-11-10
estado: en-progreso
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - seguimiento
  - bajas
  - migraciones
  - deuda
  - produccion
  - slice-9
  - cicd
---

# Fase 4 — Seguimiento, bajas, deuda y producción (slice 9 y v1.0.0)

> **Plan padre:** [[email-service-plan]]
> Deja el servicio listo para producción con el tag **v1.0.0**: migraciones automáticas con EF Core, la deuda del envío elegida en la revisión del 2026-11-09, el slice 9 (aperturas, bajas y seguimiento para managers) y el endurecimiento para producción.

## Pendientes

- [x] **4a:** migraciones de EF y deuda del envío (pasos 1–6). Implementada y verificada en local el 2026-11-09.
- [ ] Deploy de 4a a testing y fase de dudas. La API adopta la base al arrancar; antes hay que aplicar los `ALTER` que le falten (slices 7 y 8) y dar permisos DDL al usuario de la base.
- [ ] Prueba manual de la pausa por cuota (con Mailpit no se puede simular).
- [x] **4b:** slice 9, seguimiento y bajas (pasos 7–11). Implementada y verificada en local el 2026-11-10.
- [ ] Deploy de 4b a testing y fase de dudas. Antes, en Dokploy: `EMAIL_SENDER_TRACKING_SECRET` (base64 de 32 bytes) y `EMAIL_SENDER_PUBLIC_URL` (obligatorias; sin ellas la API no arranca). Las migraciones `TrackingSeenAt` y `BatchesGroupIndex` las aplica la API.
- [ ] Posibles mejoras: bajas por categoría (v1.1 o futuro); re-suscripción por autoservicio (v1.3, decisión del 2026-11-10).
- [x] **4c:** producción, pasos 12–13 implementados el 2026-11-10 (llavero `v2.<kid>`, `AdminKeyMatcher`, proxy, límites, panel con CSP y cookie `Secure`, `compose.production.yml` —hoy `infra/perfiles/compose.prod.yml`— y CI de producción). Paso 14 (documentación) en curso.
- [x] **Flujo de ramas y versionado** (2026-11-10): ramas de trabajo desde `QA` → PR a `QA` → `staging` → `main` (PM); versión SemVer decidida con `make commit-*` (envuelve `infra/utils/push.sh`; `X.Y.Z` y pre-release de testing `X.Y.Z-t.N`); el bot del CI solo crea tags y Releases; producción promueve la API probada en testing. Ver [[flujo-ci-cd-manual]] y [[versionado-semver-manual]].
- [x] **Reorganización de la raíz** (2026-11-10): composes por perfil en `infra/perfiles/compose.{dev-services,dev,testing,prod,shared-services}.yml`, scripts en `infra/utils/` (`push.sh`, `push-harbor.sh`, `version-lib.sh`, `perfil.sh`, `check.sh`), `.env` por perfil en `infra/` y uno por app; el `Makefile` es la entrada única (`make commit-*`, `make publish-*`, `make up|down|logs|config-<perfil>`, `make check`). En `QA`, la candidata `X.Y.Z` deja `deploy-testing` en verde sin publicar. Dokploy: testing con `infra/perfiles/compose.testing.yml` (rama `QA`) y producción con `infra/perfiles/compose.prod.yml` (rama `main`).
- [ ] **v1.1:** quitar del llavero la rama que abre los tokens `v1` (cuando no quede ninguna key `v1` vigente).
- [ ] Siguientes pasos (los hace el responsable): staging completo y fase de dudas; después la **v1.0.0**: `make commit-major-test` (`1.0.0-t.1`) para testing y, aprobada, `make commit-major` (`1.0.0`) para la candidata y el PR a `main`.

## Estado Actual

| Área | Estado |
|------|--------|
| Estados de mensaje en la base (`QUEUED`, `SENT`, `RETRYING`, `FAILED`, `SEEN`) | ✅ |
| Campos locales de baja en `contacts` (la sincronización no toca `Subscribed`) | ✅ |
| Plan de la Fase 4 aprobado (2026-11-09) | ✅ |
| 4a — Migraciones y deuda del envío | ✅ en local (2026-11-09) |
| 4b — Píxel, baja, seguimiento para managers y panel `/seguimiento` | ✅ en local (2026-11-10) |
| 4c — Proxy, keyring, panel endurecido, compose y pipeline de producción | ✅ en local (2026-11-10; pasos 12–13) |
| Flujo de ramas `QA` → `staging` → `main` y versionado SemVer con `push.sh` | ✅ (2026-11-10) — [[flujo-ci-cd-manual]] |

## Alcance

Cubre apertura, baja, seguimiento acotado por dueño del grupo, migraciones, la deuda del envío que entra (ver "Revisión de fases 1–3" en [[email-service-plan]]) y la salida a producción.

No cubre:
- webhooks de entrega o rebote ni seguimiento de clics (roadmap);
- retención y purga de mensajes (v1.2.0);
- consultas del Core con `CONCAT_WS` y vigencia (v1.1.0);
- consumo concurrente y Stalwart (siguiente versión).

`SEEN` es un indicador aproximado: algunos clientes precargan imágenes y otros las bloquean.

## Decisiones del responsable (2026-11-09)

- **Migraciones:** las aplica la API al arrancar (`DatabaseMigrator` con `MigrateAsync`, opción `Database:MigrateOnStartup`). El CI deja el script idempotente como artefacto para revisarlo. Se pasa a code-first: sin scaffold y sin `db.sql`.
- **Bajas:** solo en lotes **no transaccionales**; el 1 a 1 las ignora (lo decide la app consumidora). La baja **conserva los grupos**: solo marca `subscribed=false`, así que las transaccionales siguen llegando. Se corrige la nota de `AGENTS.md` sobre `RemoveContactsAsync`.
- **Seguimiento:** el `GroupManager` ve los mensajes y los lotes **de los grupos de los que es dueño**; el Admin ve todo.
- **Cuota agotada:** se pausa el consumo y se reintenta cada cierto intervalo; un endpoint de Admin reenvía desde la dlq.
- **Retención y purga:** pasan a la v1.2.0.
- **Producción:** proxy y API, panel, keyring de cifrado, compose y pipeline de producción. OpenBao queda fuera: Dokploy genera el `.env`.

## Sub-fase 4a — Migraciones y deuda del envío

- **Migraciones:** parcial `EmailDbContext.Schema.cs` con los CHECK e índices de `db.sql` (los de expresión con `migrationBuilder.Sql`), `dotnet-tools.json` con `dotnet-ef` 10 y una fábrica de diseño que lee `ConnectionStrings__Postgres`. Migración base `Inicial`, comparada con `db.sql` antes de darla por buena. ~~`infra/migrations/baseline.sql`~~: sustituido por la **adopción automática** en `DatabaseMigrator` (valida la base antigua contra el modelo de `Inicial` y la marca como aplicada; si difiere, no arranca y lista lo que falta).
- **`DatabaseMigrator`:** `IHostedService` registrado primero; `MigrateAsync` con su candado (o `pg_advisory_lock` si no basta entre réplicas). Si falla, la API no arranca. `compose.yml` (hoy `infra/perfiles/compose.dev-services.yml`) y el `Makefile` dejan de cargar `db.sql`, que se elimina.
- **Idempotencia:** `TryReserveAsync` hace `SET NX` de la entrada final **antes** de publicar (el `messageId` es determinista) y `RemoveAsync` la libera si la publicación falla.
- **`ISentLedger` en Valkey:** asíncrono, `sentledger:<id>` con TTL de 24 h; el contador sigue en memoria y, si Valkey falla, respaldo en memoria con aviso.
- **`xmin` en `messages`:** `DbUpdateConcurrencyException` → `InConcurrencyConflictException`. Si el correo ya salió, se reescribe `SENT` (y `BatchCloser.RecountAsync` si el lote ya cerró); si no, se salta la fila terminal.
- **Pausa por cuota:** `QuotaExceeded` no pasa la fila a `FAILED` ni cuenta intento. El consumidor hace nack, cancela y vuelve tras `Smtp:QuotaPauseMinutes` (30); la marca `smtp:quota-paused` en Valkey frena al reconciliador. `POST messages/dlq/redrive` (Admin) reabre y republica los que llevan `x-failure: quota`.
- **Limpieza:** `WatchAsync` se suscribe dentro del iterador; `DeliveryResult` lleva el intento desde la fila; `AuthClaims` a `Core/Security` y `ToActor()` en `Api/Auth`; `SendTestModal` con `useListOnOpen` y `useSelectedTemplate`.

**Implementado (2026-11-09).** Migraciones `Inicial` y `MessageXmin`; `DatabaseMigrator` con `pg_advisory_lock` y adopción automática; `make migrations-script` y artefacto `migrations-sql` en el CI; `ISentPendingStore` en Valkey (`SentLedger:TtlHours` 24) e `ISentLedger` solo con los contadores; `DeliveryResult(Outcome, Attempt, Failure)` y cabecera `x-failure` (`quota`/`rejected`/`exhausted`/`corrupt`/`failed`); `POST messages/dlq/redrive?max=` (Admin, `Strict`, 1..1000, por defecto 100) que devuelve `{redriven, skipped}`. Verificado en local:
- migraciones al arrancar con adopción de la base existente;
- 1 a 1 en `SENT`;
- dos peticiones simultáneas con la misma key publican una sola vez;
- lote `FINISH` por SSE y 0 suscripciones tras cerrar;
- redrive 200/400/401.

Pendiente de prueba manual: la pausa por cuota (Mailpit no la simula). `EmailSendService.Tests` necesita ajustes por las firmas nuevas; los hace el responsable.

## Sub-fase 4b — Slice 9 (seguimiento y bajas)

- **Esquema** (primera migración nueva): `messages.seen_at` e índice `ix_mb_contact`.
- **Tokens:** secreto obligatorio `TRACKING_SECRET` y `App:PublicUrl`. `ITrackingTokens` con HMAC-SHA256 (`v1.<payload>.<hmac>`, propósitos `open` y `unsub`); `TryRead` no lanza.
- **Inyección** en `MessageDispatchService`, tras el `Sanitize`: píxel en todos los mensajes; enlace de baja y cabeceras `List-Unsubscribe` / `List-Unsubscribe-Post` solo en lotes no transaccionales.
- **Endpoints públicos** con el rate limit `Public` por IP:
  - `GET t/o/{token}.gif`: `SENT` → `SEEN` atómico; siempre devuelve el GIF con `no-store`.
  - `GET u/{token}`: página de confirmación (los precargadores no dan de baja).
  - `POST u/{token}`: baja idempotente (también one-click); `subscribed=false`, sin sacar al contacto de sus grupos.
- **Seguimiento:** `MessagePermissions` y `BatchPermissions` acotados al dueño del grupo (403 `ResourceNotGranted`). GraphQL `messages` y `batches` y `GET batches/{id}` se abren al `GroupManager`; filtros `batchId`, `groupId` y `templateId`; `seenAt` y `seenCount`.
- **Panel:** página `/seguimiento` (`GroupManager` y Admin) con pestañas Mensajes y Lotes reutilizadas, columnas de apertura y totales "abiertos / enviados". `/envios` del Admin no cambia.

**Implementado y verificado (2026-11-10).**
- **Migraciones:** `TrackingSeenAt` (`messages.seen_at`, `ix_mb_contact`) y `BatchesGroupIndex` (`ix_batches_group`).
- **Tokens:** HMAC-SHA256 con `TRACKING_SECRET`, formato `v1.<base64url(id+propósito)>.<firma>`, propósitos `Open` y `Unsubscribe`, firma comparada en tiempo constante. Rotar el secreto invalida los enlaces de los correos ya enviados.
- **`App:PublicUrl`:** obligatoria fuera de Development (URL absoluta http/https). Con `TRACKING_SECRET`, sin ellas la API no arranca.
- **Píxel:** en todos los correos, después del saneo. `GET {PublicUrl}/api/v1/t/o/{token}.gif` responde siempre 200 (GIF de 43 B, `no-store`); `SENT` → `SEEN` con `seen_at` por UPDATE condicional.
- **Baja:** solo en lotes no transaccionales; pie, línea en el texto plano, `List-Unsubscribe` y `List-Unsubscribe-Post: List-Unsubscribe=One-Click` (RFC 8058). `GET u/{token}` confirma con botón y `POST u/{token}` da de baja (también one-click); 200 "Te diste de baja", 200 "Ya estabas dado de baja", 400 enlace no válido. CSP estricta, `nosniff`, `no-referrer`, sin datos del contacto. Marca `subscribed=false`, `unsubscribed_at` y `unsubscribed_reason='enlace'` y conserva los grupos.
- **Rate limit `Public`:** por IP, 60 fichas con recarga de 30 cada 60 s.
- **Seguimiento para managers:** `GET messages/{id}`, `GET batches/{id}` y GraphQL `messages`/`batches` abiertos al `GroupManager`, acotados a sus grupos (ajeno o inexistente → 403 `ResourceNotGranted`); filtros `batchId`, `groupId`, `templateId`; `seenAt` y `seenCount`.
- **Panel `/seguimiento`:** pestañas Mensajes y Lotes, columnas "Abierto" y "Abiertos / enviados", filtros y "Ver mensajes" desde el detalle del lote; `SEEN` se muestra como "Abierto" en todo el panel.
- **Operación:** `make init` genera `TRACKING_SECRET` si no existe (no la rota); `compose.app.yml` (hoy `infra/perfiles/compose.dev.yml`) pasa `App__PublicUrl` (`http://localhost:5037`); staging pide `EMAIL_SENDER_TRACKING_SECRET` y `EMAIL_SENDER_PUBLIC_URL`.

Prueba de punta a punta en local:
- Lote no transaccional: el correo lleva píxel, pie de baja, línea en el texto plano y las cabeceras `List-Unsubscribe` y `List-Unsubscribe-Post`.
- GIF con token alterado → 200 sin cambio; con token válido → `SEEN` y `seenCount` 1.
- `GET u/{token}` 200 con CSP; token alterado 400; `POST` 200; el one-click repetido es idempotente.
- El contacto queda `subscribed=false` y sigue en el grupo.
- El siguiente lote no transaccional sale `FINISH 0/0`; el transaccional, `1/1` y sin baja.
- El 1 a 1 sale sin baja.
- Manager: su lote 200; ajeno o inexistente 403; GraphQL solo devuelve lo suyo.
- Rate limit: 56×200 y 19×429.

## Sub-fase 4c — Producción

- **API:** `UseForwardedHeaders` como primer paso (`Proxy:KnownNetworks` de Traefik; el rate limit usa la IP real), `AllowedHosts` por entorno, cuerpo máximo de 1 MB (también en gRPC) y `OpenBaoHealthCheck` en Healthy si no está configurado.
- **Keyring:** `AesGcmKeyProtector` emite `v2.<kid>.…` y abre `v2` (actual o `KEY_ENCRYPTION_PREVIOUS`) y `v1`. Admin key comparada en tiempo constante, con `ADMIN_API_KEY_PREVIOUS` opcional.
- **Panel:** `security.allowedDomains`, `ASTRO_KEY` estable desde Dokploy y cabeceras (CSP compatible con Maily y el `iframe sandbox`, `frame-ancestors 'none'`, `X-Content-Type-Options`, `Referrer-Policy`).
- **Despliegue:** `compose.production.yml` (hoy `infra/perfiles/compose.prod.yml`; Traefik con TLS, router h2c para gRPC, sin puertos publicados, healthcheck del panel), `push-harbor.sh --env prod` (otro proyecto y robot, versión `X.Y.Z` del tag) y job de CI para los tags `v*` con el artefacto de migraciones y el webhook de Dokploy.
- **Wiki:** rotación de la admin key, `KEY_ENCRYPTION`, `TRACKING_SECRET` y `ASTRO_KEY`, y las variables de producción.

**Implementado (2026-11-10, pasos 12 y 13).**
- **Llavero de cifrado:** `KeyringKeyProtector` emite tokens `v2.<kid>` con AAD por `kid` y abre los `v1` con la clave vigente o las anteriores; sustituye a `AesGcmKeyProtector`. Se valida al arrancar: `KEY_ENCRYPTION_ID` o `KEY_ENCRYPTION_PREVIOUS` mal escritos impiden arrancar en vez de fallar en la primera petición. Pendiente v1.1: quitar la rama `v1`.
- **Admin key:** `AdminKeyMatcher` compara en tiempo constante, con `ADMIN_API_KEY_PREVIOUS` opcional para el solape al rotar.
- **Proxy:** `UseForwardedHeaders` es el primer paso del pipeline (`ProxyOptions`, `KnownIPNetworks`); sin redes de confianza no se acepta `X-Forwarded-For`.
- **Límites y health:** cuerpo máximo de 1 MB (5 MB en plantillas) y límite en gRPC; `health` no queda en Degraded si no hay OpenBao.
- **Panel:** dominio y `ASTRO_KEY` fijados al compilar (build-arg y secreto de BuildKit por entorno), cookie `Secure`, CSP y cabeceras de seguridad.
- **`compose.production.yml`** (hoy `infra/perfiles/compose.prod.yml`): Traefik con TLS para la API, gRPC h2c y el panel, sin puertos publicados, healthcheck del panel, `AllowedHosts` y HSTS; `APP_VERSION=latest` y una versión fija solo para volver atrás.
- **Publicación y CI:** `push-harbor.sh --env prod` (otro proyecto y cuenta robot) y `--promote` (copia a producción la API probada en testing, `tree-<hash>`, sin recompilar). El job previsto "de tags `v*`" quedó como `deploy-production` en el push a `main`: el tag `vX.Y.Z` lo crea `release-staging` y producción lo marca como definitivo. Detalle en [[flujo-ci-cd-manual]].
- **Versionado:** `infra/utils/push.sh` + `infra/utils/version-lib.sh` (por `make commit-*`) deciden la versión SemVer y escriben la fila del historial; `push-harbor.sh` ya no lo escribe. Ver [[versionado-semver-manual]].

## Pasos

Un sub-agente por paso; build Release `-warnaserror` 0/0 y se espera "avanza" del responsable.

| # | Sub-fase | Bloque | Paso | Hecho |
|---|----------|--------|------|-------|
| 1 | 4a | CREAR | Parcial de esquema, `dotnet-tools.json`, fábrica de diseño, migración `Inicial` comparada con `db.sql` (`baseline.sql` sustituido por la adopción automática) | - [x] |
| 2 | 4a | CREAR | `DatabaseMigrator`, `RedisSentLedger`, `TryReserveAsync`/`RemoveAsync`, `DeliveryResult`, `InConcurrencyConflictException`, `ToActor()` | - [x] |
| 3 | 4a | MODIFICAR | `AuthClaims` a Core, `ISentLedger` asíncrono, `xmin` en `Message`, `x-failure` en la dlq | - [x] |
| 4 | 4a | ENLAZAR | DI, `MessageService` con reserva, dispatch con `DeliveryResult` y conflicto, pausa por cuota, reconciliador, `dlq/redrive`, `WatchAsync`; sin `db.sql` en compose y Makefile; paso de CI | - [x] |
| 5 | 4a | ENLAZAR | Panel: refactor de `SendTestModal` | - [x] |
| 6 | 4a | Docs | `AGENTS.md`, wiki y Obsidian (datos code-first, deuda cerrada) | - [x] |
| 7 | 4b | CREAR | Migración `seen_at` + índice, `ITrackingTokens`/HMAC, `TrackingService`, `UnsubscribeService`, `PublicLimitPolicy`, opciones | - [x] |
| 8 | 4b | MODIFICAR | Permisos acotados por dueño del grupo, filtros, `seenAt`/`seenCount` | - [x] |
| 9 | 4b | ENLAZAR | Píxel y baja en el dispatch, `List-Unsubscribe`, endpoints `t/o` y `u`, GraphQL y REST para el `GroupManager` | - [x] |
| 10 | 4b | ENLAZAR | Panel `/seguimiento` | - [x] |
| 11 | 4b | Docs | README (tracking, bajas y límites), `AGENTS.md`, wiki, este sub-plan | - [x] |
| 12 | 4c | CREAR | Keyring `v2.<kid>`, `ForwardedHeaders`, `compose.production.yml` (hoy `infra/perfiles/compose.prod.yml`) | - [x] |
| 13 | 4c | ENLAZAR | Pipeline, límites, health, admin key en tiempo constante, panel (dominios, `ASTRO_KEY`, cabeceras), `push-harbor.sh --env prod`, job de producción (`deploy-production` en `main`); después, `push.sh` y el flujo `QA` → `staging` → `main` | - [x] |
| 14 | 4c | Docs | Wiki (rotación y producción), README, nota de QA, plan maestro | - [ ] |

Tras cada sub-fase: deploy a testing por el responsable (en 4a, la API adopta la base al arrancar; antes, los `ALTER` que falten) y fase de dudas.

## Criterios de Aceptación

- [x] Abrir el píxel de un mensaje `SENT` lo pasa a `SEEN`; un token alterado no cambia nada y siempre se devuelve el GIF. Verificado el 2026-11-10.
- [x] Un lote no transaccional lleva el píxel, el enlace de baja (también en el texto plano) y `List-Unsubscribe`; el 1 a 1 no lleva enlace.
- [x] `POST u/{token}` da de baja: el siguiente lote no transaccional excluye al contacto y uno transaccional lo incluye.
- [x] La baja sobrevive a la sincronización con el Core y el contacto conserva sus grupos.
- [x] Un `GroupManager` ve solo los mensajes y lotes de sus grupos (403 con otro o inexistente); el Admin ve todo.
- [x] El rate limit `Public` responde 429 (56×200 y 19×429).
- [ ] Una base vacía queda con el esquema completo solo con migraciones (`\d+` igual a lo que dejaba `db.sql`). Falta dejar constancia de la comparación.
- [x] Una base existente (antiguo `db.sql`) se adopta al arrancar sin perder datos y aplica las migraciones pendientes (`MessageXmin`). Verificado en local el 2026-11-09.
- Deuda cerrada:
  - [x] Dos `POST messages` simultáneos con la misma key publican una vez.
  - [x] 1 a 1 en `SENT` y lote `FINISH` por SSE; tras cerrar el SSE quedan 0 suscripciones.
  - [x] `dlq/redrive` responde 200, 400 (`max` fuera de rango) y 401.
  - [ ] Reiniciar con `sentledger` pendiente no reenvía.
  - [ ] La cuota pausa sin `FAILED` ni vencer el lote y se reanuda; `dlq/redrive` reabre y reenvía un mensaje de cuota. Pendiente de prueba manual (Mailpit no simula la cuota).
- [ ] Producción: el rate limit usa la IP real detrás de Traefik; una key emitida con la clave vieja entra tras rotar; la cookie sale con `Secure` y las cabeceras están presentes; `docker stack config` valida el compose; el job de tags pasa en `--dry-run`.
- [ ] `dotnet build` Release 0/0 y `bun run build`, `check` y `lint` correctos.

## Preguntas abiertas

- (Resuelta) La consulta de mensajes es interna: `GroupManager` (acotado a sus grupos) y Admin.
- Re-suscripción por autoservicio y bajas por categoría: fuera de la v1.0.0 (posibles mejoras de v1.1 o futuro).
- (Resuelta) Las migraciones las aplica la API al arrancar (`DatabaseMigrator`, `Database:MigrateOnStartup`); el CI deja el script idempotente como artefacto.
- (Resuelta) La retención de los mensajes consultables pasa a la v1.2.0.
- (Resuelta, 2026-11-10) `security.allowedDomains` y `ASTRO_KEY` se fijan al compilar el panel: entran por entorno en `push-harbor.sh` (build-arg `PANEL_PUBLIC_HOST` y secreto de BuildKit `astro_key`); por eso el panel se compila siempre, también al promover a producción.

## Riesgos

- Si la migración base no coincide con `db.sql`, una base existente diverge: la comparación del script es obligatoria en el paso 1. Mitigado: la adopción valida la base contra `Inicial` y, si difiere, la API no arranca.
- El usuario de la base de la API necesita permisos DDL en staging y producción.
- `SEEN` es aproximado (precarga o bloqueo de imágenes).
- Rotar `TRACKING_SECRET` invalida los píxeles y enlaces de baja de los correos ya enviados.
- La pausa por cuota es por instancia de SMTP; con varias réplicas, todas leen la misma marca en Valkey.

## Relacionado

- [[email-service-plan]]
- [[email-service-envio-individual-sub-plan]]
- [[email-service-envio-grupal-sub-plan]]
- [[email-service-sync-core-sub-plan]]
- [[email-service-grupos-sub-plan]]
- [[email-service-cicd-sub-plan]]
- [[flujo-ci-cd-manual]]
- [[versionado-semver-manual]]
