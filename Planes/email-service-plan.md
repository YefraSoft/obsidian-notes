---
tipo: plan
titulo: EmailSendService — Plan de Desarrollo
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-11-09
estado: en-progreso
proyecto: email-service
tags:
  - plan
  - email-service
  - dotnet
  - rabbitmq
  - grpc
  - redis
  - astro
  - vertical-slices
  - clean-architecture
---

# EmailSendService — Plan de Desarrollo

> API .NET 10 (minimal API) que envía correos 1 a 1 y a grupos, con PostgreSQL, RabbitMQ y secretos en OpenBao, más un panel de administración en Astro. Se desarrolla por **vertical slices** (BD + API + panel) sobre una **Clean Architecture adaptada**. Primero van las reglas de trabajo y los diagramas (decisiones de arquitectura); después, el plan desde la Fase 1.

## Contenido

- [[#Reglas de Trabajo]]
- [[#Diagramas de Funcionamiento]]
- [[#Pendientes]] · [[#Estado Actual (resumen ejecutivo)]] · [[#Decisiones de Diseño (vigentes)]]
- [[#Arquitectura y Notas Técnicas]] · [[#Archivos Clave del Proyecto]] · [[#Cómo se hace un slice (definición de hecho)]]
- FASE 1 a 4 · [[#Dependencias y Orden de Ejecución]]

---

# Reglas de Trabajo

## 1. Clean Architecture (adaptada)

- Un solo proyecto; los límites son carpetas: `Core/ · Application/ · Infrastructure/ · Api/ · Platform/`.
- **Las dependencias apuntan hacia adentro:** `Api → Application → Core` e `Infrastructure → Application → Core`. Nada de adentro conoce a nada de afuera.
- Qué vive en cada capa:

| Capa | Contenido |
|------|-----------|
| `Core/` | Entidades (`Core/Entities`, generadas por scaffold), `ConsumerRole`, `ApiKeyHasher`, jerarquía de excepciones con su nivel de log |
| `Application/` | Servicios (`IXService`) y lecturas (`IXQueries`), puertos (`IXRepository`, `IDataTransaction`, `INotificationPublisher`…), contratos (request/response), `IResponseBuilder` y sus builders, permisos (`Actor`, `Can/Require`) |
| `Infrastructure/` | `EmailDbContext`, repositorios, `DataTransaction`, RabbitMQ, SMTP |
| `Api/` | Controllers REST, tipos GraphQL, módulos de cada dominio (DI y rutas), pipeline |
| `Platform/` | Transversal técnico: handler de API key, opciones, atributos de validación, descriptores de error |

- La regla se documenta; el compilador no la verifica (un solo proyecto).

## 2. SOLID aplicado a los servicios

- **SRP:** un servicio por agregado (`ConsumerService`, `KeyService`), nunca un servicio "de administración" que lo haga todo.
- **ISP:** `IXService` para escrituras y reglas; `IXQueries` para lecturas y listas. GraphQL solo conoce `Queries`.
- **DIP:** las interfaces viven en `Application`; las implementaciones, en `Infrastructure`. Cada módulo de dominio (`IServiceInstaller`) registra las suyas.
- **OCP:** se extiende agregando implementaciones (otro `IEmailSender`, otro canal), no editando servicios.
- **LSP:** un repositorio devuelve `null` si no encuentra algo; no lanza.
- **Servicios sin HTTP:** reciben el `Actor` por parámetro (el controller hace `Actor.From(user)`) y usan `TimeProvider`; así se prueban sin framework.

## 3. Capas de un endpoint: qué hace cada pieza

| Pieza                  | Responsabilidad                                                                                                   |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------- |
| **Controller**         | HTTP: bind, política de rol, `Actor.From`, `Results.*`. No toca el `DbContext`                                    |
| **Service**            | Reglas de negocio, `Can/Require`, excepciones de dominio, un `CommitAsync` por caso de uso                        |
| **Queries**            | Lecturas y listas paginadas                                                                                       |
| **Repository**         | Consultas por intención (`NameExistsAsync`, `FindAsync`, `ListAsync`); sin genéricos ni `IQueryable` expuesto     |
| **`IDataTransaction`** | Confirma la transacción del caso de uso (`CommitAsync`) y traduce la unicidad de Postgres a `InConflictException` |
| **`IResponseBuilder`** | Proyección de EF a respuesta (`.Select(builder.Projection)`)                                                      |
| **Data**               | `EmailDbContext` y entidades                                                                                      |

Todo punto de entrada (controller REST, resolver GraphQL, consumidor de RabbitMQ) es un **adaptador**: solo traduce y delega a un servicio o a las queries.

## 4. API: REST y GraphQL

- **REST** para lo simple (GET por id, `whoami`, health) y para todo `POST / PUT / PATCH / DELETE`, con códigos 200, 201, 202 y 204.
- **GraphQL.NET** (paquetes `GraphQL` + `GraphQL.SystemTextJson`, endpoint propio, **solo consultas**) para las listas grandes, así no hay URLs largas.
- Listas: `page` y `pageSize` van en el cuerpo (con tope); respuesta `{ items, page, pageSize, total }`; filtros como *input* y orden como *enum*.
- GraphQL con la misma `X-Api-Key` y las mismas políticas; profundidad y complejidad máximas; introspección apagada en producción.
- Rutas en plural y en kebab-case; fechas en UTC con formato ISO; `Idempotency-Key` solo en los `POST` de envío.
- **gRPC para las apps consumidoras** (decisión del 2026-11-05): los envíos (`SendMessage` en el slice 7, `SendToGroup` en el 8) se exponen también por gRPC en un puerto propio solo HTTP/2 (`8081`), con la misma `x-api-key` en los metadata y las mismas políticas; `POST messages` REST se conserva para el panel y curl. Contrato `.proto` en `Src/Api/Messages/Protos`. Las consultas siguen por REST y GraphQL.

## 5. Seguridad y acceso

- Tres identidades con una sola cabecera `X-Api-Key`: **Admin** (igualdad con el secreto de OpenBao), **Consumer** (app, backend a backend) y **Manager** (persona con email que administra contactos; sin contraseña, usa su key). Son dominios separados: tablas `consumer` y `manager`, cada una con su tabla de keys (`consumer_keys`, `manager_keys`).
- La key es un token cifrado y autenticado (AES-256-GCM) que lleva el dominio, el id de la key y el del dueño: al descifrarla se sabe a qué tabla ir (una sola consulta). Una key alterada se rechaza sin tocar la base; si es válida, la base manda (SHA-256, revocada, vencida, dueño inactivo). **Una sola key vigente por dueño**: rotar reemplaza la anterior al instante; `expire_at` opcional para keys temporales.
- Roles `ConsumerRole` planos y combinables, en `capabilities`, **por dominio sin mezcla**: Consumer → `Sender`, `GroupSender`; Manager → `GroupManager`, `TemplateManager`. El administrador tiene `Admin` más los de `Auth:AdminRoles`.
- Una política de autorización por rol; un consumidor que llama a una ruta de gestión recibe el mismo 401 que con una key inválida.
- Protocolo `CanX` (bool) / `RequireX` (lanza) con matriz de dos capas: rol + alcance por recurso (`consumer_group_roles`, `consumer_templates`).
- `messages/*` es consulta interna: solo `GroupManager` y Admin. Los consumidores no conocen estados internos.

## 6. Excepciones y logs

- Jerarquía: `DomainException → AuthException / RequestException / InfrastructureException → In*`.
- **El nivel de log es una propiedad de la clase base:** `RequestException` → Information, `AuthException` → Warning, `InfrastructureException` e `InExternalServiceException` → Error.
- Por debajo de 500 se muestra el mensaje al cliente; desde 500 solo un detalle genérico. La excepción completa se registra solo desde Error.
- Respuesta `ProblemDetails` con `code` y `traceId`.

## 7. Validación y respuestas

- Atributos nativos (`[Required]`, `[StringLength]`, `[RegularExpression]`…) más `Platform/Validations` (`IsEnum`, `IsEnumList`, `FutureDate`).
- Un request inválido responde 400 con `errors` por campo y `code: Validation`.
- Las respuestas se arman con `IResponseBuilder`; los requests no transforman nada.

## 8. Mensajería

- La API **valida** (rol, plantilla, alcance) y **publica** en RabbitMQ; responde 202 con un `MessageId` derivado del consumidor y la `Idempotency-Key`.
- **Idempotencia en Redis** (patrón Stripe, 2026-11-05): con key, la API busca `idem:{consumidor}:{key}`; si existe con la misma huella del cuerpo devuelve el mismo `MessageId` sin reprocesar, si la huella difiere responde 409 `IdempotencyKeyReused`; si no existe, valida, publica y guarda (`SET NX`, 24 h). Redis caído con key → 503. El id derivado y el `ON CONFLICT` del worker son la red de seguridad.
- **El worker es dueño del registro:** lo crea en `QUEUED`, envía por SMTP y lo deja en `SENT`, `RETRYING` o `FAILED`. `SEEN` llega por el píxel de apertura.
- Entrega *at-least-once*: ack manual, reintentos con espera creciente y cola de fallidos.

## 9. Datos

- **Sin migraciones (por ahora):** EF solo consume la base; la base de producción vive aparte y la administra Efraín.
  - **Cambio previsto en el slice 9** (decisión del 2026-11-09): pasar a **migraciones automáticas con EF Core**.
  - Se hará con una migración base igual a `db.sql`, marcada como aplicada en las bases que ya existen.
  - Hasta entonces sigue el `ALTER` a mano. Ver [[email-service-seguimiento-bajas-sub-plan]].
- `db.sql` es referencia del esquema; cada slice deja listo el `ALTER` que Efraín aplica.
- Las entidades se generan por scaffold hacia `Core/Entities`.

## 10. Estilo, pruebas y proceso

- Comentarios solo a nivel de clase, breves y describiendo parámetros; un comentario dentro de un método solo si su uso es muy específico.
- Documentación en archivos `.md` (sin OpenAPI).
- **Pruebas:** unit tests los escribe Efraín; integración, otro miembro del equipo; rendimiento, con k6. Claude no escribe tests.
- Todo se trabaja en `main` hasta terminar la Fase 1; los commits los hace Efraín.
- Despliegue y respaldos se definen al terminar el desarrollo.

---

# Diagramas de Funcionamiento

## D1. Capas y dependencias

```mermaid
flowchart TB
    API["Api<br/>controllers REST · GraphQL · módulos"]
    INFRA["Infrastructure<br/>repositorios · DataTransaction · RabbitMQ · SMTP"]
    APP["Application<br/>servicios · queries · puertos · contratos · permisos"]
    CORE["Core<br/>entidades · roles · excepciones"]
    PLAT["Platform<br/>handler de API key · validaciones · errores"]
    API --> APP
    INFRA --> APP
    APP --> CORE
    PLAT -.->|transversal| API
```

## D2. Escritura de un endpoint: [CONTROLLER] → [SERVICE] → [REPOSITORY] → [DATA]

```mermaid
sequenceDiagram
    participant C as Cliente
    participant CT as CONTROLLER
    participant S as SERVICE
    participant R as REPOSITORY
    participant T as IDataTransaction
    participant D as DATA (PostgreSQL)
    C->>CT: POST /consumers + X-Api-Key
    CT->>CT: política de rol, Actor.From(user)
    CT->>S: CreateAsync(actor, request)
    S->>S: RequireManage(actor)
    S->>R: NameExistsAsync(name)
    R->>D: SELECT
    D-->>R: resultado
    S->>R: Add(consumer)
    S->>T: CommitAsync()
    T->>D: SaveChanges
    alt nombre repetido (unicidad)
        T-->>S: InConflictException
        S-->>CT: InConflictException (mensaje del servicio)
        CT-->>C: 409
    else correcto
        S-->>CT: CreatedKeyResponse
        CT-->>C: 201
    end
```

## D3. Lectura: REST por id y GraphQL para listas

```mermaid
flowchart LR
    GET["CONTROLLER<br/>GET /consumers/{id}"] --> Q["QUERIES"]
    GQL["GRAPHQL<br/>POST /graphql"] --> Q
    Q --> R["REPOSITORY"]
    R -->|"Select(IResponseBuilder.Projection)"| D[("DATA<br/>PostgreSQL")]
```

## D4. Flujo de la key (autenticación y autorización)

```mermaid
flowchart TD
    REQ["Petición con X-Api-Key"] --> RL["Rate limiting"]
    RL --> AUTH{"ApiKeyAuthenticationHandler"}
    AUTH -->|"igual a ADMIN_API_KEY"| ADM["Claims: Admin + Auth:AdminRoles"]
    AUTH -->|"descifra la key (dominio + ids)"| DEC{"¿íntegra?"}
    DEC -->|"alterada"| F401
    DEC -->|"consumer o manager: una consulta a su tabla de keys<br/>(SHA-256, vigente, dueño activo)"| CON["Claims: dominio + roles válidos del dominio"]
    AUTH -->|"no coincide"| F401["401"]
    ADM --> POL{"Política de rol"}
    CON --> POL
    POL -->|"cumple"| CTRL["CONTROLLER → SERVICE"]
    POL -->|"ruta de gestión sin rol"| F401
    POL -->|"otra política sin rol"| F403["403"]
```

## D5. Excepción → log

```mermaid
flowchart LR
    EX["EXCEPCIÓN"] --> H["GlobalExceptionHandler"]
    H --> DESC["ErrorDescriptors<br/>status · code · detalle · LogLevel"]
    DESC --> LOG["LOG con el nivel de la clase base"]
    DESC --> PD["ProblemDetails<br/>code · traceId<br/>≥500: detalle genérico"]
```

| Excepción | Estado | Nivel de log | Qué se registra |
|-----------|:------:|:------------:|-----------------|
| `InValidationException`, `InNotFoundException`, `InConflictException` (`RequestException`) | 400 / 404 / 409 | Information | Mensaje, sin excepción |
| `InLoginException`, `InRoleNotAllowedException`, `InResourceNotGrantedException` (`AuthException`) | 401 / 403 | Warning | Mensaje, sin excepción |
| `InConfigurationException`, `InExternalServiceException` (`InfrastructureException`) | 500 / 502 | Error | Mensaje y excepción completa |

## D6. Envío de un correo

```mermaid
sequenceDiagram
    participant C as Cliente
    participant API as API
    participant MQ as RabbitMQ
    participant W as Worker
    participant DB as PostgreSQL
    participant SMTP as SMTP
    C->>API: SendMessage (gRPC) o POST /messages (REST) + Idempotency-Key
    API->>API: con key, consulta Redis: mismo cuerpo → mismo MessageId; otro cuerpo → 409
    API->>API: valida rol, plantilla y alcance, y calcula MessageId
    API->>MQ: publica notificacion.email.individual
    API->>API: guarda la key en Redis (SET NX, 24 h)
    API-->>C: 202 + MessageId
    MQ->>W: entrega el mensaje
    W->>DB: crea el registro QUEUED (ignora duplicados)
    W->>SMTP: envía el correo
    alt enviado
        W->>DB: SENT
    else falla
        W->>DB: RETRYING, luego FAILED con last_error
    end
    Note over C,DB: Al abrir el correo, el píxel marca SEEN
```

---

# Pendientes

- [x] Migrar el panel a `/whoami`, a leer `errors`, a la lista por GraphQL y a los dos dominios; su arquitectura toma como ejemplo el frontend de `bolsa-trabajo-tsj` (ver `AGENTS.md`).
- [x] Terminar la Fase 1 (API y panel).
- [x] Testing en Dokploy: la API se publica en Harbor con `push-harbor.sh` (versión `X.Y.Z.TT`) y corre con `compose.testing.yml` contra una PostgreSQL externa; el panel se usa en local con `API_URL` apuntando a testing.
- [ ] Pruebas: Efraín escribe las unitarias, el equipo las de integración y k6 mide el rendimiento (la carpeta `EmailSendService.Tests/` se queda).
- [x] Slice 3 (plantillas) implementado en API y panel, con `@maily-to/core` para el editor; probado en local.
- [ ] Deploy del slice de plantillas a testing (aplicar el `ALTER` en la base) y fase de dudas.
- [x] Slice 4 (sincronización con el Core) implementado el 2026-10-30 como **revisión asistida por población** (ver [[email-service-sync-core-sub-plan]]), probado en local contra el Core real; la lista de contactos por GraphQL y la sección Contactos del panel quedan adelantadas para el slice 5.
- [x] Ajuste del slice 4 (2026-10-30): el `GroupManager` revisa y aplica la sincronización y entra a `/contactos`; se descartó un rol `ContactManager` con alcances (para un subconjunto se crea un grupo). Nota v2 en el sub-plan: profundidad de filtros en las consultas del Core.
- [x] Deploy a testing de los slices 3 y 4 (imagen 0.0.0.03, base regenerada, variables del Core en Dokploy) y E2E desde el panel (2026-10-30). Slice 4 cerrado.
- [x] Slice 5 (contactos) implementado y probado en local el 2026-10-30: alta, edición y baja de locales, edición de cualquier contacto, exportación a Excel desde el panel (importación a v2). Ver [[email-service-contactos-sub-plan]].
- [ ] Deploy a testing de los slices 3 a 5 al cerrar la Fase 2 (imagen nueva y sus `ALTER`) y E2E.
- [x] Slice 6 (grupos) implementado y probado en local (2026-10-30, docs 2026-11-03): grupos con **dueño** (`groups.id_manager`; el `GroupManager` gestiona solo los suyos, el Admin todos), nombre único global, miembros por ids, población (con exclusiones) o desde otro grupo, salida automática de los contactos dados de baja, y `consumer_group_roles` solo con `GroupSender` asignado por el Admin. Ver [[email-service-grupos-sub-plan]]. Con esto la Fase 2 queda implementada en local.
- [x] Panel: borradores de plantilla en `localStorage` (guardado al perder el foco, recuperación desde el editor y la lista) — 2026-10-30.
- [ ] CI/CD (en curso desde el 2026-11-03): Gitea Actions con runner en el servidor de Dokploy, imágenes de API y panel en Harbor, entrega a testing con push a `staging`, etiquetas `v*` para producción más adelante. Ver [[email-service-cicd-sub-plan]].
- [ ] Cerrar la Fase 2 con la primera entrega por pipeline: slices 3 a 6 a testing (`ALTER` de plantillas, sincronización, contactos y grupos a mano), E2E, fase de dudas y endpoints en Insomnia. Después, Fase 3 (envío).
- [ ] Al cerrar la Fase 2, guardar los endpoints en Insomnia.
- [x] Bitácora de errores del slice 7 (ER-014 a ER-032) corregida y verificada de punta a punta con Gmail (2026-11-06).
- [x] Slice 8 (envío a grupos y lotes) implementado y verificado en local (2026-11-09), con lotes de 2036 contactos.
  - Incluye REST, gRPC `SendToGroup`, el avance en vivo por SSE, el reconciliador y el panel.
  - De las pruebas salió **ER-033** (direcciones con "ñ" antes de la arroba): corregido con SMTPUTF8.
  - Falta el deploy a staging (`ALTER` de `batches`) y la fase de dudas. Ver [[email-service-envio-grupal-sub-plan]].
- [ ] **Siguiente versión:** consumo concurrente de la cola de envío y reparto entre lotes.
  - Hoy se procesa un mensaje a la vez y los lotes salen en orden FIFO.
  - El SMTP pasará a **Stalwart**. Antes de subir la concurrencia hay que medir la capacidad del servicio y la de Stalwart, para no saturarlos.
- [ ] **Slice 9: migraciones automáticas con EF Core** en lugar de los `ALTER` a mano (regla 9).
  - Migración base que reproduzca `db.sql` y se marque como aplicada en las bases que ya existen.
  - Hay que decidir quién las aplica. Ver [[email-service-seguimiento-bajas-sub-plan]].
- [ ] Fase 3 iniciada el 2026-11-04 con el slice 7 (envío 1 a 1): SMTP nativo tras el puerto `IEmailSender`, 3 reintentos con colas de espera y dlq, error propio por cuota de Google, `messages.id_consumer` nullable para los envíos del Admin. Bloque CREAR (C1–C6) hecho el 2026-11-05. Ver [[email-service-envio-individual-sub-plan]].
- [ ] Ajuste del slice 7 (2026-11-05): **las apps envían por gRPC** (puerto 8081 solo HTTP/2, `x-api-key` en metadata, `.proto` en `Api/Messages/Protos`; REST se conserva para el panel) y la **idempotencia vuelve a Redis** con el patrón de Stripe (mismo id sin reprocesar, 409 si se reutiliza la key con otro cuerpo, 503 sin Redis); el id derivado queda como red de seguridad. El servidor es **Valkey** (`valkey/valkey:9.0-alpine`, fork BSD compatible con Redis; el código conserva los nombres `Redis:*`): entra en los compose, en `health/ready` y en Dokploy (`EMAIL_SENDER_VALKEY_PASSWORD`, `EMAIL_SENDER_GRPC_PORT`).
- [ ] Compose de Dokploy en Swarm (2026-11-05): `compose.testing.yml` → `compose.staging.yml` (solo API y panel) y nuevo `compose.servicios.yml`, stack compartido con RabbitMQ en clúster de 2 nodos y Valkey primario + réplica sobre `dokploy-network`, reutilizable por otras apps; las colas pasan a tipo quorum. Detalle en la decisión 14 del sub-plan.
- [x] Documentación de entrega para QA de la Fase 2 (2026-11-03): [[email-service-datos-maestros-testing]] (actores, flujo, reglas, entornos de testing, los 24 endpoints con request/response, pantallas del panel y nota de pruebas); sincronizada a `3. Operación/Testing/` en Nextcloud.

# Estado Actual (resumen ejecutivo)

| Área | Estado |
|------|--------|
| Infra: compose, `make env/init/up`, OpenBao (AppRole) | ✅ |
| Errores: `DomainException`, ProblemDetails con `code` y `traceId`, nivel de log por clase | ✅ |
| Rate limiting: Relaxed / Moderate / Strict, global para lo no declarado | ✅ |
| Autenticación: `X-Api-Key` única cifrada con su dominio, `GET /whoami` (con `kind`), admin por OpenBao | ✅ |
| Dominios Consumer y Manager con keys separadas, una key vigente por dueño y roles por dominio | ✅ |
| Roles, políticas por rol, `Auth:AdminRoles` con `IOptions` | ✅ |
| Protocolo `Can` / `Require` y matriz `consumer_group_roles` | ✅ (sin endpoints que lo usen aún) |
| Gestión de consumidores y keys (API) | ✅ por capas (servicios, queries y repositorios); lista por GraphQL |
| GraphQL.NET (solo consultas, `POST /graphql`, solo Admin) | ✅ piloto: lista de consumidores |
| Validación nativa y `Platform/Validations` | ✅ |
| Módulos por responsabilidad y `IResponseBuilder` | ✅ |
| Mensajería RabbitMQ: colas quorum, reintentos con colas de espera, dlq y cola grupal | ✅ worker SMTP real (slice 7) y expansión y cierre de lotes (slice 8) |
| Endpoints de negocio (plantillas, contactos, grupos, envíos 1 a 1 y a grupos) | ✅ en local; seguimiento y bajas en el slice 9 |
| Panel Astro | ✅ login por identidad (Admin, Manager, Consumer), datos maestros, envíos y lotes con avance en vivo |
| Testing: imagen de la API en Harbor, `compose.testing.yml` en Dokploy, base externa y secretos como `Secrets__*` (OpenBao opcional) | ✅ |

| Fase / Slice | Dominio | Estado |
|--------------|---------|--------|
| 1 | Auth (key cifrada con su dominio: Admin, Consumer, Manager) — [[email-service-auth-sub-plan]] | ✅ hecha |
| 2 | Consumidores, managers y keys — [[email-service-consumidores-keys-sub-plan]] | ✅ hecha |
| 3 | Plantillas — [[email-service-plantillas-sub-plan]] | ✅ Implementada (falta deploy a testing y refinamiento) |
| 4 | Sincronización con el Core — [[email-service-sync-core-sub-plan]] | ✅ hecha (en testing, E2E 2026-10-30) |
| 5 | Contactos — [[email-service-contactos-sub-plan]] | ✅ Implementado (falta deploy a testing y E2E) |
| 6 | Grupos, miembros y asignaciones — [[email-service-grupos-sub-plan]] | ✅ Implementado (falta deploy a testing y E2E) |
| 7 | Envío 1 a 1 (gRPC y REST), idempotencia en Valkey y worker SMTP — [[email-service-envio-individual-sub-plan]] | ✅ Implementado, documentado y verificado en local (bitácora cerrada el 2026-11-06); falta deploy a staging y dudas |
| 8 | Envío a grupos y lotes — [[email-service-envio-grupal-sub-plan]] | ✅ Implementado y verificado en local (2026-11-09); falta deploy a staging |
| 9 | Seguimiento y bajas, más migraciones automáticas con EF — [[email-service-seguimiento-bajas-sub-plan]] | ⏳ Pendiente |

# Decisiones de Diseño (vigentes)

Resumen; el detalle de cada una está en las **Reglas de Trabajo**.

1. Clean Architecture adaptada y SOLID, con dependencias hacia adentro (regla 1 y 2).
2. Controller → service → repository → data; servicios y queries separados; `IDataTransaction` por caso de uso (regla 3).
3. REST para lo simple y las escrituras; GraphQL.NET para listas grandes (regla 4).
4. Una sola `X-Api-Key` cifrada con su dominio (Admin, Consumer, Manager); una key vigente por dueño; roles por dominio sin mezcla; matriz de dos capas; `messages/*` interno (regla 5).
5. El nivel de log vive en la clase base de cada excepción (regla 6).
6. Validación nativa y respuestas por `IResponseBuilder` (regla 7).
7. El worker es dueño del registro del mensaje (regla 8).
8. Sin migraciones; entidades por scaffold a `Core/Entities` (regla 9). En el slice 9 se adoptan las migraciones automáticas con EF Core (2026-11-09).
12. El consumidor ve en vivo el avance de **sus propios lotes**: correo y estado, sin ids ni errores. Lo recibe por SSE, con pub/sub de Valkey y el proxy del panel. El detalle y la lista de lotes son solo del Admin (2026-11-06).
9. Sin OpenAPI; documentación en `.md`; reparto de pruebas; trabajo en `main` (regla 10).
10. Rutas de gestión sin prefijo `admin`; dar de alta contactos locales (`source = LOCAL`) lo hacen `GroupManager` y Admin; los contactos del Core llegan por sincronización.
11. Las apps consumidoras envían por **gRPC** (puerto HTTP/2 propio, misma key en metadata); el panel usa REST y GraphQL. La idempotencia de los envíos vive en **Redis** (patrón Stripe) con el `MessageId` derivado como red de seguridad (reglas 4 y 8, 2026-11-05).

# Arquitectura y Notas Técnicas

- **Stack:** .NET 10, EF Core + Npgsql, RabbitMQ.Client, GraphQL.NET, Grpc.AspNetCore (envíos de las apps), StackExchange.Redis (idempotencia), OpenBao (AppRole + KV v2), Astro 7 híbrido con Actions y Bun.
- **Pipeline HTTP:** manejo de errores → rate limiting → autenticación y autorización → validación nativa → controller. Para gRPC, el mismo pipeline en el puerto 8081 y un interceptor que traduce las excepciones de dominio a `RpcException` (las excepciones de un método gRPC no pasan por el manejador de errores HTTP).
- **Mensajería:** exchange `notificacion` (direct, durable) y una cola por canal y alcance `notificacion.<canal>.<alcance>`; hoy solo existe `notificacion.email.individual`.
- **Módulos:** cada funcionalidad es un `IModule` que implementa solo lo que necesita: `IServiceInstaller` (DI), `IEndpointMapper` (rutas) e `IPipelineStep` (pipeline, con `Order`).
- **Ejecución local:** `bao login` → `make env` → `make init` → `make up APP=1`; el panel con `bun --bun run dev` en `EmailServiceManagment`.
- **Deuda conocida:** `Actor.cs` importa `Platform.Auth`, contra la regla de dependencias. (`GroupPermissions` y `TemplatePermissions` ya viven en `Application/Permissions` con puertos.) La baja de suscripción del slice 9 debe llamar a `IGroupMemberRepository.RemoveContactsAsync` para sacar al contacto de sus grupos.
- **Módulos:** cada dominio es un `IModule` (`ConsumerModule`, `ManagerModule`, `TemplateModule`…) con las capacidades `IServiceInstaller`, `IEndpointMapper`, `IQueryMapper` (campos GraphQL) e `IPipelineStep`; se descubren por reflexión en orden alfabético. Las regex viven en `Core/Text/RegexLibrary` (`IRegexLibrary`).

# Archivos Clave del Proyecto

| Archivo | Propósito |
|---------|-----------|
| `EmailSendService/Program.cs` | Arranque: OpenBao, módulos, pipeline y endpoints |
| `EmailSendService/Src/Infrastructure/Composition/` | `IModule`, `IServiceInstaller`, `IEndpointMapper`, `IPipelineStep`, `ModuleRegister` |
| `EmailSendService/Src/Platform/Auth/` | Handler de API key, opciones y claims |
| `EmailSendService/Src/Platform/Validations/` | Atributos de validación reutilizables |
| `EmailSendService/Src/Api/` | Controllers REST por dominio, `GraphQL/` y módulos (auth, rate limiting, validación) |
| `EmailSendService/Src/Application/` | Servicios, queries, puertos, contratos, builders y permisos |
| `EmailSendService/Src/Infrastructure/Persistence/` | `EmailDbContext`, repositorios y `DataTransaction` |
| `infra/db.sql` | Esquema de la base (referencia) |
| `Makefile`, `compose.yml`, `compose.app.yml` | Operación local y contenedor de la API |
| `compose.testing.yml`, `push-harbor.sh`, `docs/versions/version-history.md` | Testing en Dokploy: compose con la API de Harbor, script que la publica y su historial de versiones |
| `EmailServiceManagment/src/` | Panel Astro: acciones, cliente de la API y páginas |

**Estructura por dominio:** `Core/Entities`, `Application/<Dominio>` (servicios, queries, repositorios como interfaz, contratos, builders), `Infrastructure/Persistence/Repositories`, `Api/<Dominio>`.

# Cómo se hace un slice (definición de hecho)

- [ ] **Esquema:** cambios en `db.sql` y el `ALTER` listo para que Efraín lo aplique.
- [ ] **Entidad:** generada por scaffold en `Core/Entities`.
- [ ] **Datos:** puerto `IXRepository` en `Application` e implementación en `Infrastructure`.
- [ ] **Lógica:** `XService` (escrituras y reglas) y `XQueries` (lecturas).
- [ ] **Contratos:** request y response con validación nativa, y su builder.
- [ ] **Entrada:** controller REST (y tipo GraphQL si hay lista) con política de rol y `Can/Require`.
- [ ] **Panel:** pantalla en Astro con su Action, leyendo los errores por campo.
- [ ] **Documentación:** README y la nota del slice en esta bóveda al día.
- [ ] **Verificación:** `dotnet build` sin advertencias, curl contra el contenedor y `bun run build`. Los tests los agregan Efraín y el equipo.

# FASE 1 — Auth y consumidores (Slices 1 y 2)

> **Objetivo:** que la API acepte la API key (cifrada con su dominio: Admin, Consumer o Manager) con sus roles y que el administrador gestione consumidores, managers y keys desde el panel.
> **Estado:** ✅ Hecha (API, panel y despliegue de testing) — [[email-service-auth-sub-plan]] y [[email-service-consumidores-keys-sub-plan]].

# FASE 2 — Datos maestros (Slices 3 a 6)

> **Objetivo:** plantillas, base de contactos (sincronizada con el Core y local) y grupos con sus asignaciones.
> **Estado:** ✅ Implementada en local (2026-11-03); falta el deploy a testing de los slices 3, 5 y 6, el E2E y la fase de dudas — [[email-service-plantillas-sub-plan]], [[email-service-sync-core-sub-plan]], [[email-service-contactos-sub-plan]], [[email-service-grupos-sub-plan]].

# FASE 3 — Envío (Slices 7 y 8)

> **Objetivo:** enviar correos 1 a 1 y a grupos con el worker SMTP real, estados y lotes.
> **Estado:** ✅ Implementada en local.
> - **Slice 7:** verificado el 2026-11-05 (REST y gRPC, idempotencia en Valkey, reintentos con colas quorum, panel `/envios`), documentado y con su bitácora cerrada. Faltan el deploy a staging con el stack `servicios` y la fase de dudas ([[email-service-envio-individual-sub-plan]]).
> - **Slice 8:** envío a grupos y lotes, con avance en vivo por SSE. Implementado y verificado en local el 2026-11-09; falta su deploy a staging ([[email-service-envio-grupal-sub-plan]]).
> - **Siguiente versión:** consumo concurrente y reparto entre lotes, midiendo antes la capacidad del servicio y la de Stalwart (el futuro SMTP).

# FASE 4 — Seguimiento y producción (Slice 9 y endurecimiento)

> **Objetivo:** apertura, bajas y consulta interna de mensajes; **migraciones automáticas con EF Core** en lugar de los `ALTER` a mano; y dejar el servicio listo para producción (OpenBao por https y rotación del `secret_id`, AppRole restringido por IP, `ForwardedHeaders`, `ASTRO_KEY` y `allowedDomains` del panel, rotación de la admin key).
> **Estado:** ⏳ Pendiente — [[email-service-seguimiento-bajas-sub-plan]].

# Dependencias y Orden de Ejecución

```
Slice 1 — Auth
 │
 └── Slice 2 — Consumidores y keys
      │
      ├── Slice 3 — Plantillas ──────────────────────────┐
      │                                                   │
      └── Slice 4 — Sincronización con el Core            │
           │                                              │
           └── Slice 5 — Contactos                        │
                │                                         │
                └── Slice 6 — Grupos y asignaciones       │
                     │                                    │
                     ├── Slice 7 — Envío 1 a 1 ◄──────────┘
                     │        │
                     │        └── Slice 8 — Envío a grupos y lotes
                     │                 │
                     └─────────────────└── Slice 9 — Seguimiento y bajas
```

El Slice 7 necesita las plantillas (3) y el 8 necesita además los grupos (6). El Slice 4 puede empezar en paralelo con el 3.

# Relacionado

- [[AGENTS]]
- [[backend-manual]]
- [[email-service-datos-maestros-testing]] (documentación para QA de la Fase 2)
