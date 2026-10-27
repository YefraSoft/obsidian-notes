---
tipo: wiki
titulo: "Diagramas del SSO MiTSJ"
autor: "Ana Espinoza"
creado: 2026-10-27
actualizado: 2026-10-27
estado: vigente
proyecto: sso
tags:
  - sso
  - diagramas
  - arquitectura
---

# Diagramas del SSO MiTSJ

## Contenido

### 1. Arquitectura general (Mermaid)


Equivale al diagrama de `arquitectura.md` §1 (imagen y vista en texto), reconstruido en Mermaid para que se pueda editar y versionar como texto.

```mermaid
flowchart TB
    subgraph SSO["SSO MiTSJ · C# / .NET · mínimo 2 instancias"]
        subgraph IFZ["Interfaces del SSO"]
            ACC["Acceso del SSO<br/>login · 2FA · recuperación · crear contraseña"]
            CON["Consola del SSO<br/>mis datos · administración · dashboard"]
        end
        subgraph NUC["Núcleo"]
            OIDC["Servidor OAuth / OIDC<br/>OpenIddict"]
            IDR["Identidad y registro<br/>CURP/RFC · RENAPO"]
            AUT["Autenticación<br/>Argon2id · Google · 2FA"]
            PERM["Motor de permisos<br/>puede · cuáles · campos"]
            ADM["API de administración"]
            KEYS["Motor de API keys"]
            BIT["Bitácora de seguridad"]
            MON["Monitoreo (SignalR)"]
            BG["Procesos en segundo plano"]
        end
    end

    APPS["Aplicaciones del ERP<br/>SDK MiTSJ (NuGet)"]
    DATA[("PostgreSQL replicado")]
    CACHE[("Redis<br/>caché · sesiones")]
    BUS["Bus de mensajes<br/>RabbitMQ"]
    CDN["CDN institucional<br/>archivos"]
    CONN["Conectores<br/>Google Workspace · Hikvision"]
    EXT["Sistemas externos<br/>RENAPO · Google · HikCentral (red aislada)"]

    ACC --> OIDC
    CON --> ADM
    APPS -- "OIDC: login y tokens" --> OIDC
    APPS -- "¿puede? · ¿cuáles? · ¿campos?" --> PERM
    APPS -- "registro de cuentas" --> IDR
    IDR --> AUT
    PERM --> CACHE
    AUT --> BIT
    PERM --> BIT
    BIT --> MON
    NUC --> DATA
    NUC --> BUS
    BUS -- "invalida caché" --> APPS
    BUS --> CONN
    IDR -- "validación de CURP" --> EXT
    CONN --> EXT
    ACC -- "archivos y fotos" --> CDN
    KEYS -- "valida API keys" --> CDN
    BG --> DATA
```

### 2. Diagramas de flujo

> Copia literal de `Obsidian/documents/SCRUM/diagramas-sso.md` (27-oct-2026).


> Origen: [[sso]] · Backlog: [[backlog-sso]]
> Los diagramas están en Mermaid: se ven en Obsidian en modo lectura o Live Preview.

---

## 1. Visión general (arquitectura)

```mermaid
flowchart LR
    subgraph Usuarios
        U1[Interesado / Aspirante]
        U2[Alumno / Egresado]
        U3[Docente]
        U4[Administrativo]
        U5[Organismo / Proveedor]
    end

    subgraph SSO["SSO MiTSJ (microservicio, PostgreSQL)"]
        AUTH[Autenticación<br/>OAuth2 / OIDC / JWT<br/>2FA · bloqueo · sesiones]
        PERM[Motor de permisos<br/>roles · grupos · alcance<br/>registros asignados · campos]
        ADM[Consola de administración]
        CUENTA[Portal Mi cuenta]
        BIT[(Bitácora de seguridad)]
        DASH[Dashboard en tiempo real]
    end

    subgraph ERP["Aplicaciones del ERP (usan el SDK MiTSJ)"]
        A1[Servicios Escolares]
        A2[Finanzas]
        A3[Vinculación]
        A4[Futuras apps]
    end

    GW[Google Workspace]
    HIK[HikCentral / Hikvision<br/>torniquetes · plumas · cámaras de placas]

    Usuarios -->|login / registro| AUTH
    Usuarios --> CUENTA
    ADM --> PERM
    AUTH --> BIT
    PERM --> BIT
    BIT --> DASH

    AUTH -->|token JWT| ERP
    ERP -->|¿puede? ¿sobre cuáles? ¿qué campos?| PERM
    PERM -->|evento: cambió un permiso| ERP
    ERP -->|reporta cambio de perfil| SSO

    AUTH <-->|inicio de sesión / alta y suspensión| GW
    SSO -->|personas, fotos, placas, vigencias| HIK
    HIK -->|eventos de paso| BIT
```

---

## 2. Mapa de usuarios y cómo se relacionan

```mermaid
flowchart TB
    subgraph Jerarquia["Jerarquía institucional (define el alcance)"]
        DG[Dirección General<br/>alcance Global] --> UA[13 Unidades Académicas<br/>alcance UA]
        UA --> AR[Áreas administrativas<br/>y Carreras<br/>alcance Área / Carrera]
    end

    subgraph Personas["Tipos de usuario (una cuenta = una credencial, CURP o RFC)"]
        INT[Interesado] -->|primer pago| ASP[Aspirante]
        ASP -->|inscripción| ALU[Alumno<br/>Vigente / Baja]
        ALU -->|egresa| EGR[Egresado]
        DOC[Docente<br/>grupo · investigación<br/>tutor · asesor residencia]
        ADMV[Administrativo<br/>roles dinámicos]
        ORG[Organismo<br/>Empresa / Institución]
        PROV[Proveedor]
    end

    ORG --> REP[Representante]
    ORG --> USR[Varios usuarios del organismo]

    subgraph Admins["Quién administra"]
        ROOT[Root<br/>arranque inicial]
        AG[Administrador global]
        AUA[Administrador de UA]
    end

    ROOT -.crea.-> AG
    AG -.alcance.-> DG
    AUA -.alcance.-> UA
    Personas -.pertenecen a.-> UA
```

> Los cambios de perfil (Aspirante → Alumno → Egresado) **no los decide el SSO**: los reporta la aplicación responsable (Admisión / Servicios Escolares) y el SSO ajusta roles y permisos.

---

## 3. Registro y activación de cuentas

```mermaid
flowchart TD
    S([Persona nueva]) --> F[Captura CURP/RFC, correo, celular,<br/>tipo de cuenta, UA, foto si aplica<br/>+ acepta aviso de privacidad]
    F --> C{CAPTCHA y límite por IP}
    C -->|falla| X1([Rechazado])
    C -->|ok| D{¿CURP/RFC ya registrada?}
    D -->|sí| X2([Rechazado: cuenta existente])
    D -->|no| G{¿Correo existe en<br/>Google Workspace?}
    G -->|sí| G1[Asocia la cuenta del SSO a Google]
    G -->|no| G2[Alta normal]
    G1 --> V[Valida correo y celular]
    G2 --> V
    V --> T{Tipo de cuenta}
    T -->|Aspirante| ACT([ACTIVA automáticamente])
    T -->|Docente / Organismo /<br/>Administrativo / Proveedor| PEND[PENDIENTE de activación]
    PEND --> AD{Administrador revisa<br/>global: todas · UA: las de su UA, no aspirantes}
    AD -->|aprueba| ACT
    AD -->|rechaza con motivo| REJ([Rechazada, correo al usuario])
    ACT --> GW[Crea cuenta en Google Workspace<br/>si requiere correo institucional]
```

### Ciclo de vida de una cuenta (nunca se elimina)

```mermaid
stateDiagram-v2
    [*] --> Pendiente: registro
    Pendiente --> Activa: validación / aprobación
    Pendiente --> Rechazada: rechazo con motivo
    Activa --> Bloqueada: N intentos fallidos
    Bloqueada --> Activa: pasan X minutos
    Activa --> Inactiva: administrador (con motivo)
    Inactiva --> Activa: solicita reactivación y admin aprueba
    note right of Inactiva
        No puede iniciar sesión.
        Google Workspace: suspendida.
        Torniquetes: acceso retirado.
        Vehículos: inactivos.
    end note
```

---

## 4. Flujo de inicio de sesión y autorización

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant APP as Aplicación (SDK MiTSJ)
    participant SSO as SSO (Autenticación)
    participant G as Google Workspace
    participant P as Motor de permisos
    participant B as Bitácora

    U->>APP: Abre la aplicación
    APP->>SSO: Redirige a login (OIDC)
    alt Contraseña
        U->>SSO: CURP/RFC o correo + contraseña
        SSO->>SSO: Verifica Argon2id, bloqueo por intentos
    else Google
        SSO->>G: Autenticación OAuth 2.0
        G-->>SSO: Identidad confirmada
    end
    opt 2FA requerido (cuenta o rol)
        SSO->>U: Pide código TOTP
        U->>SSO: Código
    end
    opt Acciones obligatorias
        SSO->>U: Cambiar contraseña, aviso de privacidad, etc.
    end
    SSO->>B: Registra sesión (IP, navegador, dispositivo)
    SSO-->>APP: Token JWT ligero (identidad, perfiles, UA, roles resumidos)
    APP-->>U: Sesión iniciada

    U->>APP: Intenta una acción (p. ej. editar horario)
    APP->>P: ¿Puede? ¿Sobre cuáles? ¿Qué campos?
    P-->>APP: Permitido / denegado + filtro de registros + campos visibles
    APP->>APP: Aplica el filtro y vuelve a verificar en backend
    APP-->>U: Resultado (o denegado por omisión)

    Note over P,APP: Si cambia un permiso, el SSO publica un evento<br/>y las cachés se invalidan en menos de 1 minuto
```

### Cómo se decide un permiso

```mermaid
flowchart TD
    Q([Solicitud: cuenta + acción + formulario/registro]) --> A{¿Cuenta activa<br/>y sesión vigente?}
    A -->|no| DEN([DENEGADO])
    A -->|sí| B{¿Denegación individual<br/>explícita?}
    B -->|sí| DEN
    B -->|no| C{¿Permiso individual,<br/>de grupo o rol vigente?}
    C -->|no| DEN
    C -->|sí| D{¿Dentro del alcance?<br/>Global · UA · Área/Carrera}
    D -->|no| DEN
    D -->|sí| E{¿Es un registro asignado<br/>a la persona?<br/>docente/tutor/alumno}
    E -->|no| DEN
    E -->|sí| F[PERMITIDO<br/>+ filtro de registros<br/>+ campos visibles / solo lectura / ocultos]
    DEN -.si es excepción.-> K[Llave de autorización excepcional<br/>un solo uso, con caducidad]
    K -.registrada.-> F
```

---

## 5. Control de acceso físico (fase 3)

```mermaid
flowchart LR
    SSO[(SSO<br/>fuente de verdad)] -->|alta / cambio / baja<br/>foto validada, vigencia,<br/>nivel de acceso, placas| CON[Conector Hikvision]
    CON -->|OpenAPI| HC[HikCentral]
    HC --> T[Terminales y torniquetes<br/>deciden localmente]
    HC --> CAM[Cámaras de placas<br/>abren o cierran la pluma]
    T -->|evento de paso| HC
    CAM -->|lectura + foto| HC
    HC -->|eventos| CON
    CON -->|bitácora y dashboard| SSO

    Inactiva([Cuenta inactivada<br/>o vigencia vencida]) -->|retiro en menos de 5 min| CON
    CON -.conciliación completa diaria.-> HC
```

> A Hikvision solo se envía lo necesario (ID interno, nombre, foto, vigencia). **Nunca la CURP.**

---

## 6. Fases del proyecto

```mermaid
flowchart LR
    F0[Fase 0<br/>Levantamiento<br/>inventario, decisiones,<br/>impacto de datos] --> F1[Fase 1<br/>Núcleo MVP<br/>cuentas, login, 2FA,<br/>permisos, SDK,<br/>Mi cuenta, migración]
    F1 --> F2[Fase 2<br/>Gobierno<br/>campos, vigencias, llaves,<br/>suplencias, dashboard,<br/>Google Workspace]
    F2 --> F3[Fase 3<br/>Acceso físico<br/>fotos, Hikvision,<br/>vehículos y placas]
    F3 --> F4[Fase 4<br/>Biometría y mejoras]

    F1 -.pruebas de seguridad.-> P1((OK))
    F2 -.pruebas de seguridad.-> P2((OK))
    F3 -.pruebas + pentest.-> P3((OK))
```

## Relacionado

- [[sso-implementacion-plan]]
- Requerimientos: `sso` (v0.16) y `arquitectura` (v0.9), en las Fuentes del plan.
