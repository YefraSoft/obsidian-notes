---
tipo: wiki
titulo: "Asistencias — Arquitectura y flujo de información"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-17
estado: vigente
proyecto: asistencias
tags:
  - wiki
  - asistencias
  - arquitectura
  - diagramas
---

# Asistencias — Arquitectura y flujo de información

> Diagramas de cómo funciona la app: contexto, componentes, flujos de checada, evidencia, incidencias, datos y despliegue. Plan maestro: [[asistencias-plan]]. Roles y autorización: [[asistencias-roles-permisos-wiki]].

## Contenido

- [[#1. Principios y decisiones]]
- [[#2. Contexto del sistema]]
- [[#3. Componentes]]
- [[#4. Ingreso: primer ingreso y posteriores]]
- [[#5. Checada válida]]
- [[#6. Rechazo y reintento]]
- [[#7. Consulta de evidencia]]
- [[#8. Incidencias y avisos]]
- [[#9. Modelo de datos de new-core]]
- [[#10. Despliegue y secretos]]
- [[#11. Pendientes técnicos]]

## 1. Principios y decisiones

- **Ligero por Astro Actions, pesado directo al CDN.** El navegador habla con el servidor de Astro (Actions) para todo lo ligero; el video/foto de evidencia va **directo al CDN**, sin pasar por la API ni por Astro Actions.
- **Cálculo en el cliente, comparación en el servidor.** Detección, reto de gestos (MediaPipe) y embedding se calculan en el navegador; la API verifica la comparación contra el embedding de referencia, la geocerca y las reglas.
- **`new-core` es la fuente tras el registro.** El Core viejo (MariaDB) solo se consulta en el primer ingreso de cada persona; los roles viven en `new-core` y se ligan por usuario.
- **Sin RabbitMQ en Asistencias.** El post-proceso y los avisos usan una tabla **outbox** y un servicio en segundo plano; las incidencias, una tarea programada con bloqueo entre réplicas.
- **Evidencia privada.** Los archivos van a la ruta `private/` del CDN; Astro (servidor) hace de proxy de lectura para RH/Admin.

## 2. Contexto del sistema

```mermaid
flowchart LR
    P["Persona<br/>navegador del celular"]
    subgraph ASIS["Asistencias"]
        W["App web Astro + React<br/>servidor: Actions y proxy de lectura"]
        API["API .NET"]
        PG[("new-core<br/>PostgreSQL")]
    end
    CORE[("Core viejo<br/>MariaDB<br/>solo registro del usuario")]
    CDN["CDN institucional<br/>docs-service + Caddy"]
    MAIL["email-service<br/>notificaciones"]
    KEYS["App que provee la key rotatoria del CDN<br/>mecanismo por definir"]
    BAO["OpenBao<br/>apps/asistencias y apps/asistencias-prod"]

    P -- "Actions: login, reto, checada, consultas" --> W
    P -- "video/foto de evidencia directo" --> CDN
    W --> API
    API --> PG
    API -. "solo primer ingreso, lectura" .-> CORE
    API -- "avisos desde la outbox" --> MAIL
    W -- "lectura de evidencia, proxy" --> CDN
    KEYS -. "X-Api-Key vigente" .-> W
    BAO -. "secretos" .-> API
    BAO -. "secretos" .-> W
```

## 3. Componentes

```mermaid
flowchart TB
    subgraph NAV["Navegador"]
        UI["Pantallas React<br/>checada, historial, RH, admin"]
        ML["MediaPipe + modelo ONNX Runtime Web<br/>reto de gestos y embedding"]
        REC["Grabación del clip<br/>cámara del dispositivo"]
    end
    subgraph ASTRO["Servidor Astro"]
        ACT["Astro Actions<br/>valida sesión y llama a la API"]
        PXY["Proxy de lectura de evidencia<br/>agrega keys del CDN"]
        KG["Entrega de keys temporales de subida<br/>solo con sesión y reto activo"]
    end
    subgraph APIB["API .NET - capas por carpeta"]
        AP["Api<br/>controllers REST por dominio"]
        APP["Application<br/>servicios, queries, puertos, permisos"]
        INF["Infrastructure<br/>AsistenciasDbContext, CoreDbContext lectura, clientes CDN y email"]
        PLT["Platform<br/>autenticación, opciones, validaciones, errores"]
        BG["Segundo plano<br/>outbox y tarea de incidencias"]
    end
    UI --> ACT
    UI --> ML
    ML --> REC
    ACT --> AP
    AP --> APP
    APP --> INF
    PLT -.-> AP
    BG --> APP
    PXY -.-> KG
```

Las dependencias apuntan hacia adentro: `Api → Application → Core` e `Infrastructure → Application → Core`.

## 4. Ingreso: primer ingreso y posteriores

El Core viejo solo se consulta para el **registro del usuario**; todo inicio de sesión posterior es contra `new-core`.

```mermaid
sequenceDiagram
    autonumber
    actor P as Persona
    participant A as Astro (Action login)
    participant API as API .NET
    participant NC as new-core
    participant CV as Core viejo (solo lectura)

    P->>A: usuario o CURP + contraseña
    A->>API: POST /sesiones
    API->>NC: ¿existe el usuario?
    alt existe en new-core
        NC-->>API: usuario, credencial, rol y alcance
        API->>API: verifica contraseña (hash propio)
        API-->>A: sesión con rol y alcance
    else no existe
        API->>CV: valida credenciales y lee datos
        alt credenciales válidas
            CV-->>API: datos del docente, UA, URL de foto, horarios
            API->>NC: migra perfil, crea credencial propia y asigna rol nivel 3
            API-->>A: sesión
        else inválidas
            API-->>A: 401 uniforme
        end
    end
    A-->>P: cookie de sesión
```

Pendiente: qué se migra exactamente y cómo se reflejan cambios posteriores del Core (ver [[#11. Pendientes técnicos]]).

## 5. Checada válida

```mermaid
sequenceDiagram
    autonumber
    actor P as Persona (navegador)
    participant A as Astro (Actions)
    participant API as API .NET
    participant DB as new-core
    participant CDN as CDN
    participant M as email-service

    P->>A: iniciarChecada
    A->>API: POST /checadas/retos
    API->>DB: crea reto de un solo uso con vigencia
    API-->>A: retoId, gestos, expiración
    A-->>P: reto y keys temporales de subida
    Note over P: MediaPipe: reto de gestos<br/>Detección y embedding en el cliente<br/>Graba clip corto con la cámara
    P->>CDN: sube el clip directo (ruta private, archivo UUID)
    CDN-->>P: fileId
    P->>A: registrarChecada (GPS, embedding, retoId, fileId)
    A->>API: POST /checadas
    API->>DB: valida sesión, reto, geocerca, bloque de horario y embedding
    alt válida
        API->>DB: guarda checada y fila en outbox
        API-->>A: 201 válida
    else rechazada
        API->>DB: guarda intento con motivo
        API-->>A: 422 con motivo
    end
    A-->>P: resultado
    API-)M: aviso desde la outbox, en segundo plano
```

Verificaciones del backend (el cliente no decide): sesión y rol, reto vigente y no reutilizado, coordenada dentro de la geocerca con la precisión permitida, bloque de horario correspondiente, forma y dimensión del embedding, similitud contra el embedding de referencia (umbral configurable) y hora del servidor.

## 6. Rechazo y reintento

```mermaid
flowchart TD
    S["Registrar checada"] --> V{"¿Reto vigente<br/>y no usado?"}
    V -- no --> R1["Rechazo: reto inválido"]
    V -- sí --> G{"¿Dentro de la<br/>geocerca?"}
    G -- no --> R2["Rechazo: fuera de la zona"]
    G -- sí --> E{"¿Embedding<br/>coincide?"}
    E -- no --> R3["Rechazo: rostro no coincide"]
    E -- sí --> B{"¿Hay bloque de<br/>horario aplicable?"}
    B -- no --> R4["Rechazo: sin horario"]
    B -- sí --> OK["Checada válida"]
    R1 --> I["Se guarda el intento con motivo y evidencia"]
    R2 --> I
    R3 --> I
    R4 --> I
    I --> RT["La persona reintenta de inmediato<br/>con un reto nuevo"]
    OK --> N["Outbox: confirmación"]
    I --> N2["Outbox: aviso de rechazo"]
```

## 7. Consulta de evidencia

```mermaid
sequenceDiagram
    autonumber
    actor R as RH o Admin
    participant A as Astro (servidor)
    participant API as API .NET
    participant C as CDN (Caddy + docs-service)

    R->>A: solicita ver la evidencia de una checada
    A->>API: GET /checadas/{id}/evidencia (permiso y alcance)
    alt permitido por rol y alcance
        API-->>A: fileId y ruta
        A->>C: GET /cdn/private/... con X-Api-Key, X-User-Key y X-Private-Route-Key
        C-->>A: archivo
        A-->>R: transmite el archivo
    else no permitido
        API-->>A: 403
        A-->>R: sin permiso
    end
```

La `X-Private-Route-Key` y las demás keys nunca salen del servidor de Astro. Las imágenes no se incluyen en reportes.

## 8. Incidencias y avisos

```mermaid
flowchart TD
    T["Tarea programada<br/>con pg_advisory_lock entre réplicas"] --> H["Lee horarios por bloque y checadas válidas"]
    H --> C{"¿Falta la entrada o salida<br/>del bloque?"}
    C -- "hasta 15 min" --> W["Sin incidencia aún"]
    C -- "15 min o más, menos de 1 h" --> RE["Retardo"]
    C -- "1 h o más" --> FA["Falta del bloque"]
    C -- "sin checada en el bloque" --> OL["Falta de entrada o salida (olvido)"]
    C -- "salida antes del fin" --> SA["Salida anticipada"]
    C -- "ninguna checada en el día" --> AU["Ausencia del día"]
    RE --> DB[("Incidencia en new-core<br/>sin duplicar")]
    FA --> DB
    OL --> DB
    SA --> DB
    AU --> DB
    DB --> OB["Fila en outbox"]
    OB --> SV["Servicio en segundo plano<br/>envía a email-service con reintentos"]
```

La evaluación es **por bloque**: un docente puede tener varias incidencias el mismo día.

## 9. Modelo de datos de new-core

```mermaid
erDiagram
    USUARIO ||--|| CREDENCIAL : tiene
    USUARIO ||--o{ ASIGNACION_ROL : tiene
    ROL ||--o{ ASIGNACION_ROL : "se asigna"
    UA ||--o{ USUARIO : adscribe
    UA ||--|| GEOCERCA : define
    USUARIO ||--o{ HORARIO_BLOQUE : tiene
    USUARIO ||--o{ EMBEDDING_REFERENCIA : tiene
    USUARIO ||--o{ CHECADA : registra
    USUARIO ||--o{ INTENTO : registra
    RETO ||--o| CHECADA : valida
    CHECADA }o--|| HORARIO_BLOQUE : "corresponde a"
    HORARIO_BLOQUE ||--o{ INCIDENCIA : "evaluado en"
    USUARIO ||--o{ INCIDENCIA : genera
    CHECADA ||--o{ OUTBOX : dispara
    INCIDENCIA ||--o{ OUTBOX : dispara

    USUARIO {
        uuid id PK
        string curp
        string nombre
        uuid ua_id FK
        string url_foto_referencia
        string origen_registro
    }
    ROL {
        int nivel PK
        string nombre
    }
    ASIGNACION_ROL {
        uuid usuario_id FK
        int rol_nivel FK
        string alcance
        uuid ua_id FK
        date vigente_desde
        date vigente_hasta
    }
    CHECADA {
        uuid id PK
        uuid usuario_id FK
        string tipo
        timestamptz fecha_utc
        float lat
        float lon
        float precision_m
        string evidencia_file_id
    }
    INTENTO {
        uuid id PK
        uuid usuario_id FK
        string motivo
        timestamptz fecha_utc
        string evidencia_file_id
    }
    OUTBOX {
        uuid id PK
        string tipo
        string estado
        int intentos
    }
```

El Core viejo no aparece: solo se lee en el primer ingreso y no se migra su esquema.

## 10. Despliegue y secretos

```mermaid
flowchart LR
    DEV["Desarrollo<br/>rama feat/*"] --> GIT["Gitea<br/>QA, staging, main"]
    GIT --> CI["Gitea Actions"]
    CI --> HB["Harbor<br/>imágenes API y app web"]
    CI -- "webhook" --> DK["Dokploy<br/>testing sigue QA, producción sigue main"]
    HB --> DK
    BAO["OpenBao<br/>apps/asistencias<br/>apps/asistencias-prod"] -.-> DK
    MOD["Volumen de modelos ONNX<br/>fuera de git y de la imagen"] -.-> DK
    DK --> APPS["API .NET y app web Astro"]
    APPS --> PG[("new-core PostgreSQL")]
    APPS --> CDN["CDN"]
    APPS --> MAIL["email-service"]
```

## 11. Pendientes técnicos

- Cómo llega la `X-Api-Key` rotatoria a Asistencias (hoy "se obtiene desde otra app"; el CDN espera un endpoint propio que reciba la nueva key) y dónde se guardan la `private_route_key` y la `X-User-Key` de rol `UPLOADER` (OpenBao).
- Supuesto vigente: Astro entrega al navegador una `X-Api-Key` vigente y la `X-User-Key` UPLOADER solo en sesión autenticada con reto activo; riesgo aceptado (visibles en el navegador). El límite de tasa del CDN lo sube el responsable (tarea en Plane).
- Nombres de colección e identificador en el CDN (propuesta: `UA-<id>-<AAAA-MM>` y archivo con UUID, sin datos personales).
- Qué se migra del Core en el primer ingreso (perfil, UA, URL de foto, horarios) y cómo se reflejan cambios posteriores (baja, UA, horario, contraseña).
- Algoritmo del hash propio de la credencial y política de bloqueo del login.
- Si los horarios pasan a administrarse en Asistencias en la v1 (`RH_DIR`) o solo se migran desde el Core.

## Índice / Tabla de contenidos

- [[#1. Principios y decisiones]]
- [[#2. Contexto del sistema]]
- [[#3. Componentes]]
- [[#4. Ingreso: primer ingreso y posteriores]]
- [[#5. Checada válida]]
- [[#6. Rechazo y reintento]]
- [[#7. Consulta de evidencia]]
- [[#8. Incidencias y avisos]]
- [[#9. Modelo de datos de new-core]]
- [[#10. Despliegue y secretos]]
- [[#11. Pendientes técnicos]]

## Relacionado

- [[asistencias-plan]]
- [[asistencias-roles-permisos-wiki]]
- [[asistencias-reglas-de-negocio-wiki]]
- [[cdn-service-manual]]
- [[email-service-plan]]
