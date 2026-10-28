---
tipo: wiki
titulo: "Diseño base del SSO MiTSJ"
autor: "Ana Espinoza"
creado: 2026-10-27
actualizado: 2026-10-28
estado: draft
proyecto: sso
tags:
  - sso
  - diseno-base
  - seguridad
---

# Diseño base del SSO MiTSJ

> Esquema base de diseño (antes `Planes/sso-implementacion-plan.md`). **Ya no es un plan vigente:** las reglas de negocio se depuran en [[sso-reglas-negocio-wiki]] y, cuando estén cerradas, de ahí saldrán los planes. Ante conflicto, mandan las decisiones de [[sso-decisiones-wiki]].

## Fuentes copiadas

> Copia literal de los documentos de `Obsidian/documents/SCRUM/requerimientos/` (27-oct-2026). Si cambian allá, esta copia no se actualiza sola.

### Fuente 1: sso.md (v0.16)

# SSO – Sistema Integral de Seguridad (MiTSJ)

> Documento de requerimientos de alto nivel
> Proyecto: ERP Universitario MiTSJ
> Autor: Gabriel Cisneros · Versión 0.16 · 15-oct-2026
> Estado: Borrador para revisión

---

## 1. Propósito

Construir un **servicio de inicio de sesión único (SSO)** que sea la única puerta de entrada a todas las aplicaciones del ERP universitario y al control de acceso físico (torniquetes). El SSO decide **quién es** cada persona (autenticación), **qué puede hacer, dónde, sobre qué registros y hasta cuándo** (autorización), y deja registro de los eventos de seguridad.

La primera aplicación que lo usará es **Servicios Escolares** (gestión de alumnos, grupos, horarios, bajas, etc.), pero el diseño debe servir para cualquier aplicación futura sin cambiar el código del SSO.

---

## 2. Contexto institucional

- **Dirección General** (nivel superior).
- **13 Unidades Académicas**, cada una con:
  - **Áreas administrativas** (p. ej. Servicios Escolares, Finanzas, Vinculación).
  - **Áreas académicas / carreras** (p. ej. Ingeniería en Sistemas Computacionales).

Esta jerarquía (Dirección General → Unidad Académica → Área/Carrera) es la base para definir el **alcance** de los permisos (ver §6.4).

---

## 3. Arquitectura e integración

> Diagrama y flujos completos en [`arquitectura.md`](arquitectura.md).

| Componente | Requerimiento |
|---|---|
| Backend | **Microservicio** independiente, expuesto vía API REST, sin depender de las demás aplicaciones. |
| Lenguaje backend | **C# sobre .NET (versión LTS vigente)** con **ASP.NET Core**. |
| Base de datos | **PostgreSQL** (acceso con Entity Framework Core + Npgsql). |
| Servicios externos | **RENAPO** (validación de CURP, §4.4), **Google Workspace** (§8.1, §11.1), **Hikvision** (§12), **CDN institucional** (aplicación gestionada por el SSO y almacén de sus archivos, §3.5). |
| Protocolo | **OAuth 2.0 / OpenID Connect** con tokens **JWT** (de acceso de corta duración y de renovación). |
| Interfaz SSO | **Dos interfaces:** (1) **Acceso del SSO** (pública): login, 2FA, recuperación y crear contraseña; (2) **Consola del SSO**: mis datos, contraseña, foto, vehículos, sesiones, administración y dashboard en tiempo real (las opciones visibles dependen del rol). El registro de cuentas está en las aplicaciones autorizadas (§9). |
| Dashboard | Monitoreo en tiempo real de sesiones y actividad (ver §13). |
| Documentación | API (OpenAPI/Swagger), manual de administrador y guía de integración para nuevas aplicaciones. |

### 3.1 Registro de aplicaciones cliente
- Cada aplicación del ERP se registra en el SSO con su **identificador y secreto**, sus **URL de retorno** autorizadas y su catálogo de módulos, formularios y acciones (ver §6.1).
- Cada aplicación tiene configurados los **tipos de cuenta que puede registrar** (ver §9.1). Ejemplo: Admisión → aspirantes; Vinculación → organismos; Recursos Humanos → docentes y administrativos; Compras → proveedores. El SSO **rechaza** cualquier registro fuera de lo permitido.
- El administrador puede **suspender** una aplicación completa.
- Los tokens se pueden **revocar** (por sesión, por cuenta o por aplicación).

### 3.2 Comunicación entre microservicios
- Cuando un microservicio llama a otro (p. ej. Finanzas consulta a Servicios Escolares), se autentica con una **cuenta de servicio** (no de persona) con permisos propios, limitados y auditados.
- Mecanismo preferido: tokens OAuth 2.0 (*client credentials*). Para sistemas que no pueden usar tokens, el SSO emite **API keys** con su motor de API keys (§3.7).

### 3.3 Diseño propio inspirado en Keycloak
El SSO es un **desarrollo propio e independiente** (no se usará Keycloak ni otro producto), pero toma como **referencia de diseño** los conceptos probados de Keycloak:

| Concepto Keycloak | Equivalente en el SSO MiTSJ |
|---|---|
| Realm | Un solo dominio institucional (TecMM), con la jerarquía Dirección General → UA → Área |
| Client | Aplicación del ERP registrada (§3.1) |
| Client scopes / mappers | Datos que cada aplicación recibe en el token (perfil, UA, roles resumidos) |
| Realm roles / client roles / roles compuestos | Roles globales, roles por aplicación y roles que agrupan otros roles |
| Groups (con herencia) | Grupos con herencia de permisos (§6.3) |
| Identity brokering | Inicio de sesión con Google Workspace (§8.1) |
| Authentication flows | **Flujos de autenticación configurables** (p. ej. contraseña → 2FA según rol) |
| Required actions | **Acciones obligatorias** al iniciar sesión: verificar correo/celular, cambiar contraseña, configurar 2FA, aceptar aviso de privacidad, subir foto |
| SSO session idle / max | Tiempo de inactividad y duración máxima de sesión (§8.6, §8.7) |
| Brute force detection | Bloqueo por intentos (§8.5) |
| Login events / admin events | Bitácora de seguridad (§13.2) |
| Service accounts / client credentials | Cuentas de servicio (§3.2) y **motor de API keys** para sistemas que no usan tokens (§3.7) |
| Login pages | **Acceso del SSO**: login, 2FA, recuperación y crear contraseña |
| Account console + Admin console | **Consola del SSO**: mis datos, contraseña, 2FA, foto, vehículos, sesiones, administración y dashboard (según el rol) |
| Authorization Services (recursos, alcances, políticas) | Motor de permisos propio (§6 y §6.11) |

Al ser desarrollo propio, se deben seguir **estrictamente los estándares** (OAuth 2.0, OpenID Connect, JWT) usando **librerías criptográficas reconocidas** (no implementar criptografía propia), y validar con las **pruebas de conformidad de OpenID** y **pruebas de penetración** antes de producción.

### 3.4 Tecnología de referencia (C# / .NET)
Librerías sugeridas (componentes de programación, no productos de SSO; se confirman en diseño técnico):

| Necesidad | Opción en .NET |
|---|---|
| Servidor OAuth 2.0 / OpenID Connect | **OpenIddict** (librería de código abierto para construir el servidor propio) |
| Hash de contraseñas Argon2id | Librería Argon2id reconocida para .NET (p. ej. Konscious / Isopoh) |
| Tokens JWT | `Microsoft.IdentityModel.*` |
| 2FA (TOTP) | Librería TOTP estándar (RFC 6238) |
| Acceso a datos | Entity Framework Core + Npgsql (PostgreSQL) |
| Monitoreo en tiempo real | **SignalR** (dashboard §13.4) |
| Eventos de invalidación de caché | Bus de mensajes (p. ej. RabbitMQ) o Redis Pub/Sub |
| Caché distribuida | Redis |

### 3.5 CDN institucional: aplicación gestionada por el SSO y almacén de archivos del SSO
El **CDN institucional** es una **API propia, externa al SSO**, que ya opera en producción y maneja **archivos públicos y privados**. Tiene una **doble relación** con el SSO:

**a) El CDN es una aplicación más gestionada por el SSO**
- Se **registra como aplicación cliente** en el SSO (§3.1) con su catálogo de permisos (§6.1). Ejemplo:
  - *Módulos:* Fotos de cuentas · Fotos de vehículos · Documentos de identidad · Documentos de alumnos · Archivos públicos…
  - *Acciones:* subir · leer · reemplazar · eliminar · hacer público.
- El CDN **acepta los tokens del SSO** (de personas o de cuentas de servicio de las aplicaciones) y consulta al motor de permisos si la acción está permitida; si el CDN está en C#, usa el **SDK MiTSJ**.
- Aplican las mismas reglas que en cualquier aplicación: **alcance** (un jefe de carrera solo ve archivos de su carrera), **registros asignados** (un docente solo ve las fotos de sus alumnos) y **denegado por omisión**.
- **Archivos públicos:** se siguen sirviendo con enlace directo, sin token (rápido y cacheable). Solo **subir, reemplazar, eliminar o publicar** requiere permiso.
- **Archivos privados:** el CDN entrega una **URL firmada de corta duración** solo cuando el SSO confirma el permiso.
- **Los archivos van directo de la aplicación al CDN**; el SSO solo autoriza (no pasa archivos por el SSO, para no volverlo cuello de botella).
- Las acciones sobre archivos privados quedan registradas (quién, qué archivo, cuándo) y se reflejan en el dashboard del SSO.

**b) El SSO depende del CDN para guardar sus fotos y documentos**
- Todos los archivos que maneja el SSO (fotografías de cuentas y vehículos, tarjetas de circulación, constancias, documentos migratorios, etc.) se guardan en el CDN.
- El SSO **no guarda archivos**: solo el **identificador del archivo en el CDN** y sus metadatos (tipo, fecha, quién lo subió, estado de validación).
- El SSO usa una **cuenta de servicio propia** ante el CDN, emitida por el mismo SSO, con permisos solo sobre los módulos de archivos del SSO.
- El **conector Hikvision** obtiene las fotografías desde el CDN con su propia cuenta de servicio.
- Los nombres de archivo **no contienen datos personales** (nunca la CURP); **cifrado en reposo** y respaldo de archivos sensibles a cargo del CDN.
- Al inactivar una cuenta los archivos **no se eliminan** (§11); se aplican los plazos de conservación (§15).

**c) Cómo se evita que la dependencia mutua los tumbe a ambos**
- **El login no depende del CDN:** si el CDN no está disponible, el SSO sigue autenticando y autorizando; solo se afecta ver o subir fotos y documentos (se muestra un aviso y se reintenta).
- **El CDN valida los tokens localmente** (firma del JWT con las llaves públicas del SSO, que guarda en caché), así que no necesita consultar al SSO en cada petición.
- **Archivos públicos** siguen funcionando aunque el SSO esté caído; para **archivos privados**, el CDN puede usar una **caché corta de decisiones de permiso** y, si no tiene respuesta, **niega** (denegado por omisión).
- La **cuenta de servicio del SSO** en el CDN se configura al arranque (como parte de la instalación del root, §7), para no depender de un ciclo SSO→CDN→SSO al iniciar.
- Las subidas del SSO al CDN se hacen con **reintentos**; si una falla, el registro queda pendiente y se completa después.

**d) API keys del CDN: pasan a ser gestionadas por el SSO**
1. Las **API keys del CDN** dejan de administrarse en el CDN y pasan al **motor de API keys del SSO** (§3.7): el SSO las emite, rota, revoca y registra su uso; el CDN las valida contra el SSO.
2. Las API keys actuales se **inventarían e importan** (o se reemplazan por nuevas emitidas por el SSO) con un periodo de convivencia.
3. Las aplicaciones que puedan usar **tokens del SSO** (empezando por el SSO y Servicios Escolares) usan tokens; las que no, usan **API keys emitidas por el SSO**.

### 3.6 Propagación de cambios de permisos
- Cuando se quita un permiso, se inactiva una cuenta o se cierra una sesión, el cambio debe surtir efecto en todas las aplicaciones en un **tiempo máximo configurable** (objetivo: menos de 1 minuto).

### 3.7 Motor de API keys
Módulo del núcleo del SSO para que **aplicaciones y sistemas externos** que no pueden usar tokens OAuth 2.0 (p. ej. el **CDN institucional**, integraciones de terceros, scripts o equipos) se autentiquen con **API keys gestionadas centralmente**.

- **Emisión:** cada API key pertenece a una **aplicación registrada o cuenta de servicio** (§3.1, §3.2), nunca a una persona; la emite un administrador desde la Consola del SSO.
- **Permisos (scopes):** cada llave tiene permisos tomados del mismo catálogo del SSO (aplicación → módulo → acción) y, si aplica, **alcance** (UA / área). Denegado por omisión.
- **Seguridad de la llave:**
  - Se muestra **una sola vez** al crearla; el SSO guarda solo su **hash** (como una contraseña).
  - Prefijo visible para identificarla (p. ej. `mitsj_cdn_…`) sin revelar el secreto.
  - **Vigencia** obligatoria con fecha de expiración y aviso previo.
  - Restricción opcional por **IP / red de origen**.
  - **Límite de uso** (peticiones por minuto) por llave.
- **Rotación sin cortes:** se puede emitir una llave nueva mientras la anterior sigue válida un periodo corto, y después se revoca.
- **Revocación inmediata** desde la Consola; se propaga como cualquier cambio de permisos (§3.6).
- **Validación:** el sistema que recibe la llave (p. ej. el CDN) la valida con la **API de validación de API keys** del SSO (respuesta: válida/no válida, dueño, permisos), con caché corta; el SDK MiTSJ incluye esta validación.
- **Bitácora y dashboard:** cada uso queda registrado (llave, sistema, IP, acción, resultado); alertas por uso anómalo, llaves por vencer o llaves sin uso.
- **Inventario:** listado de todas las llaves activas, su dueño, permisos, último uso y fecha de expiración (reportes §14).

---

## 4. Identidad de las cuentas

### 4.1 Identificador único
| Tipo de persona | Identificador principal | Alterno |
|---|---|---|
| Persona con CURP | **CURP** | — |
| Persona sin CURP (p. ej. extranjeros) | **RFC** | — |
| Organismo / Proveedor persona moral | **RFC** del organismo | CURP (o RFC) de su representante |

- **Una cuenta = una sola credencial de acceso.** Con ella el usuario entra a todas las aplicaciones a las que tenga permiso.
- Una persona puede tener **varios perfiles a la vez** con la misma cuenta (p. ej. docente y alumno de posgrado, o administrativo y tutor).

### 4.2 Fotografía de la cuenta
- Cada cuenta tiene una **fotografía ligada**, **obligatoria** para los perfiles que ingresan físicamente al instituto.
- **Etapa 1:** la foto se usa para el **control de acceso físico** con torniquetes Hikvision (ver §12).
- **Etapa 2 (futura):** la foto será la base para **reconocimiento facial**. *Nota:* si las terminales Hikvision instaladas son de reconocimiento facial, este ya ocurre en la etapa 1 dentro del propio equipo.
- La foto debe cumplir los requisitos técnicos de Hikvision (formato JPG, tamaño y resolución según el modelo de terminal).
- Requisitos de la foto (fondo, tamaño, rostro visible) y un flujo de **validación por un administrador** antes de aceptarla.
- Historial de fotografías (no se sobreescriben sin registro).
- Las fotografías se almacenan en el **CDN institucional** (§3.5); el SSO solo guarda su identificador.
- La foto es un **dato personal sensible** (ver §15): requiere consentimiento expreso del titular.

### 4.3 Corrección de identidad
- Un administrador puede **corregir una CURP/RFC** capturada con error, dejando registro.
- Un administrador puede **fusionar cuentas duplicadas** de la misma persona, conservando el historial de ambas.
- Toda CURP corregida se **vuelve a validar con RENAPO** (§4.4).

### 4.4 Validación de CURP con RENAPO
El SSO consume el servicio de consulta de CURP de **RENAPO** para confirmar que la CURP existe, está vigente y corresponde a la persona. Como cada consulta tiene costo/cupo, **RENAPO es el último paso** de la validación: solo se consulta cuando todas las validaciones internas pasaron.

**Orden de validación (de la más barata a la más costosa)**
1. **Formato** de la CURP (18 caracteres, estructura oficial).
2. **Dígito verificador** (posición 18), calculado localmente con el algoritmo oficial.
3. **Coherencia** con los datos capturados (fecha de nacimiento y sexo contenidos en la CURP).
4. **¿Ya existe en el SSO?** (cuentas activas, pendientes o inactivas). Si existe, **no se consulta RENAPO**: se indica que la cuenta ya está registrada y se ofrece iniciar sesión o recuperar la contraseña.
5. **¿Ya se validó antes?** Si la CURP tiene una validación RENAPO reciente guardada (p. ej. de un registro que no se concluyó), se reutiliza sin volver a consultar (vigencia configurable).
6. **Consulta a RENAPO** — solo si todo lo anterior pasó.

**Uso de la respuesta de RENAPO**
- Si la CURP **no existe o no está vigente** (dada de baja, duplicada, etc.) → se rechaza el registro con un mensaje claro.
- Si es válida → se comparan **nombre(s), apellidos y fecha de nacimiento** capturados contra los de RENAPO; los datos oficiales quedan como **datos de identidad de la cuenta** (el usuario no los puede editar después; solo un administrador vía §4.3).
- **Protección de datos:** el sistema **no muestra** al usuario los datos que devuelve RENAPO antes de completar su registro (solo responde "coincide / no coincide"), para evitar que alguien use el registro para consultar datos de terceros con una CURP ajena.
- Se guarda: fecha de validación, resultado y estatus de la CURP.

**Control del consumo**
- La consulta a RENAPO se hace **después del CAPTCHA** y con **límite de consultas por IP y por sesión** de registro, para evitar abuso automatizado.
- **Contador de consultas** (por día/mes, por tipo de resultado) visible en el dashboard, con **alerta** al acercarse al cupo contratado.
- En **cargas masivas** (§9.4) las validaciones se hacen en lotes programados fuera de horario y respetando el cupo.

**Si RENAPO no está disponible**
- El registro **no se pierde**: la cuenta queda en estado *"Pendiente de validación RENAPO"* y el sistema **reintenta automáticamente**; se notifica al usuario cuando se complete.
- Protección con *circuit breaker* (si RENAPO falla repetidamente, se deja de llamar unos minutos para no saturarlo) y tiempos de espera definidos.

**Alcance**
- Aplica a personas **con CURP**. Las personas identificadas solo por **RFC** (extranjeros, personas morales) se validan por otros medios (documento migratorio, constancia de situación fiscal) revisados por el administrador.
- Las credenciales de acceso a RENAPO se guardan como secretos fuera del código; cada consulta queda en la bitácora de seguridad (sin exponer los datos devueltos).

---

## 5. Tipos de usuario y perfiles

| Tipo | Perfiles / subtipos | Notas |
|---|---|---|
| **Interesado** | — | Toda persona nueva que aspira a una Licenciatura o Posgrado. |
| **Aspirante** | — | Tras su primer pago de aportación para el examen de admisión. |
| **Alumno** | Vigente · Baja | Usa la misma cuenta que tuvo como aspirante. |
| **Egresado** | — | **Perfil propio** con sus permisos (p. ej. solicitar documentos, seguimiento de egresados). |
| **Docente** | De grupo · De investigación · Tutor · Asesor de residencia | Puede tener varios subtipos. |
| **Organismo** | Empresa · Institución | **Varios usuarios** por organismo (ver 5.1). |
| **Proveedor** | — | |
| **Administrativo** | Roles definidos dinámicamente | Los configura el administrador del SSO. |

- El SSO **no decide** los cambios de perfil o estado: los **reporta la aplicación responsable** (p. ej. Servicios Escolares reporta que un alumno pasó a *Egresado*). El SSO recibe el cambio y **ajusta roles y permisos** automáticamente.
- Cada cambio de perfil queda registrado (qué aplicación lo reportó y cuándo).

### 5.1 Organismos con varios usuarios
- El organismo es una entidad (RFC, razón social) con un **representante** y **varios usuarios** asociados (p. ej. contactos para residencias, recursos humanos).
- Cada usuario del organismo tiene **su propia cuenta** (con su CURP o RFC) ligada al organismo.
- El **representante** puede solicitar alta o baja de usuarios de su organismo; un administrador las aprueba.
- Se puede **cambiar de representante** sin perder las cuentas ni el historial del organismo.

---

## 6. Modelo de autorización (permisos)

### 6.1 Catálogo de aplicaciones
El administrador debe poder **registrar dinámicamente**:

```
Aplicación → Módulo → Formulario / Recurso → Campo → Acción
(Servicios Escolares) → (Grupos) → (Formulario "Horario") → (campo "Aula") → (ver, crear, editar, eliminar, aprobar, exportar…)
```

- Como la mayoría de los módulos son **formularios tipo Notion**, la granularidad llega a **formulario + campo + acción**.
- Las acciones no están fijas en código: cada aplicación declara las suyas.

### 6.2 Roles
- Conjunto de permisos con nombre (p. ej. "Jefe de carrera", "Coordinador de Servicios Escolares").
- Se crean y modifican dinámicamente desde la consola del SSO.

### 6.3 Grupos y permisos individuales
- Se pueden crear **grupos** (p. ej. "Administradores de Servicios Escolares – UA Tala").
- Las cuentas de un grupo **heredan todos los permisos del grupo**.
- El administrador puede **agregar o quitar permisos a una cuenta en particular**, sin afectar al resto del grupo.
- Prioridad: **denegación individual explícita** > permiso individual > permiso de grupo/rol.
- El administrador puede ver el **permiso efectivo** de una cuenta y de dónde proviene.

### 6.4 Alcance organizacional
Cada permiso se otorga **dentro de un alcance**: Global (Dirección General) → Unidad Académica → Área / Carrera.

Ejemplo: un **Jefe de Carrera de Sistemas** crea grupos y horarios de **su carrera**, pero **no puede editar** los de otra carrera ni de otra Unidad Académica.

### 6.5 Permisos sobre registros asignados
Además del alcance por área, un permiso puede limitarse a **los registros asignados a la persona**:
- Un docente solo ve **sus grupos** y a **sus alumnos**.
- Un tutor solo ve a **sus tutorados**; un asesor de residencia solo a **sus residentes**.
- Un alumno solo ve **su propia información**.
- Las aplicaciones informan al SSO (o el SSO consulta) la relación persona ↔ registro.

### 6.6 Permisos por campo
- Un permiso puede permitir ver un formulario pero **ocultar o hacer de solo lectura ciertos campos** (p. ej. ver el expediente de un alumno sin ver su CURP, domicilio o datos de salud).

### 6.7 Vigencia de permisos
- Cualquier permiso, rol, grupo o perfil puede tener **fecha de inicio y fin**.
- Al vencer, el permiso **se desactiva automáticamente** y queda registro.

### 6.8 Llaves de autorización excepcional (un solo uso)
Caso: un usuario no puede ejecutar una acción (p. ej. dar de baja a un alumno porque **ya terminó el periodo de bajas**). La regla que bloquea la acción pertenece a la aplicación; el SSO provee el mecanismo de la llave.

- Un usuario de **nivel superior** genera una **llave** para esa persona.
- La llave es para **una acción específica** (y, si aplica, un registro específico), de **un solo uso** y con **caducidad** configurable.
- Se registra quién la generó, para quién, para qué, el motivo y cuándo se usó.

### 6.9 Suplencias (delegación temporal)
- Un usuario (o su superior) puede **delegar temporalmente** sus permisos, o parte de ellos, a otra persona (p. ej. vacaciones, incapacidad).
- La delegación tiene **fecha de inicio y fin**, se desactiva sola y las acciones del suplente quedan registradas **como suplente de** la persona titular.

### 6.10 Separación de funciones
- El administrador puede definir **combinaciones de permisos incompatibles** (p. ej. "registrar pago" y "aprobar pago").
- El sistema **impide o alerta** cuando se intenta asignar a una misma cuenta permisos en conflicto.

### 6.11 Cómo se aplican los permisos en las aplicaciones
El SSO **decide** los permisos; cada aplicación los **aplica** sobre sus propios datos (el SSO no accede a las bases de datos de las aplicaciones).

```
Usuario → Aplicación (usa el SDK MiTSJ) ──¿puede?──► Motor de permisos del SSO
                    ◄── permitido / denegado, filtros y campos visibles ──
```

1. **Token de sesión (ligero):** contiene la identidad, perfiles, Unidad Académica y roles resumidos. **No** contiene el detalle de permisos (sería demasiado grande y quedaría desactualizado).
2. **API de permisos del SSO** con tres consultas:
   - **¿Puede?** — ¿esta cuenta puede hacer *esta acción* sobre *este formulario / registro*?
   - **¿Sobre cuáles?** — devuelve el **filtro** de registros permitidos (p. ej. "grupos donde es docente", "carrera = Sistemas, UA = Tala") para que la aplicación lo aplique en su consulta.
   - **¿Qué campos?** — por formulario, qué campos son **visibles, de solo lectura u ocultos**.
3. **Relaciones persona ↔ registro (permisos por asignación):** las aplicaciones informan al SSO las relaciones que dan acceso (p. ej. *docente X imparte grupo Y*, *tutor X tiene al alumno Z*); el motor las usa para resolver §6.5. Modelo de referencia: permisos basados en relaciones (estilo Google Zanzibar / OpenFGA).
4. **SDK MiTSJ** (librería común para backend y frontend) que todas las aplicaciones usan para: validar el token, consultar permisos, aplicar filtros, ocultar campos en los formularios tipo Notion y manejar la captura de **llaves excepcionales**.
5. **Caché con invalidación:** el SDK guarda en caché las respuestas por poco tiempo; cuando cambia un permiso, el SSO **publica un evento** y las cachés se invalidan (cumple §3.6).
6. **Doble verificación:** el frontend oculta lo no permitido, pero **el backend de cada aplicación siempre vuelve a verificar** (nunca se confía solo en la interfaz).
7. **Denegado por omisión:** si no hay un permiso explícito, la acción se niega.

### 6.12 SDK MiTSJ
Librería común que **todas las aplicaciones del ERP** usan para conectarse al SSO, para que ningún equipo programe la seguridad por su cuenta.

**Funciones**
| Función | Ejemplo (Servicios Escolares) |
|---|---|
| Inicio/cierre de sesión | Redirige al login del SSO y regresa al usuario autenticado |
| Registro de cuentas | Componente que envía los datos de identidad a la API de registro del SSO (solo en apps autorizadas, §9) |
| Validar el token en cada petición | Rechaza tokens falsos, vencidos o de sesiones cerradas |
| Usuario actual | Nombre, perfiles, UA y roles sin volver a consultar al SSO |
| ¿Puede? | ¿El jefe de carrera puede editar el horario del grupo 123? |
| ¿Sobre cuáles? | Filtro "carrera = Sistemas, UA = Tala" aplicado a la consulta |
| ¿Qué campos? | CURP oculta, calificación de solo lectura en el formulario |
| Llaves excepcionales | Ventana para capturar la llave y validarla con el SSO |
| Reportar relaciones | "Docente X imparte el grupo Y" |
| Reportar cambios de perfil | "Alumno Z egresó" |
| Caché con invalidación | Respuestas en caché por segundos; se borran al cambiar un permiso |
| Actividad de sesión | Avisa al SSO que el usuario sigue activo (inactividad y dashboard) |

**Presentaciones**
- **SDK de backend (C# / .NET):** paquete **NuGet** para ASP.NET Core. Es el que **realmente protege**. Incluye:
  - *Middleware* de autenticación (validación del token).
  - Atributo/política de autorización, p. ej. `[RequierePermiso("Grupos", "crear")]`.
  - Extensión para **Entity Framework Core** que aplica el filtro de "¿sobre cuáles?" a las consultas (`IQueryable`).
  - Cliente de la API de permisos, de llaves, de **registro de cuentas** y de reporte de relaciones/cambios.
  - Cliente para **cuentas de servicio** (comunicación entre microservicios, §3.2).
- **SDK de frontend:** para la interfaz web (framework por definir). Oculta botones y campos no permitidos, maneja login, la ventana de llaves y los **componentes base del formulario de registro** (captura de CURP, correo, celular, CAPTCHA, aviso de privacidad y código de celular). Es solo comodidad visual; la seguridad la aplica el backend.

**Reglas**
- Versionado (compatibilidad hacia atrás) y documentación con ejemplos.
- Si cambian las reglas de seguridad, se actualiza el SDK y todas las aplicaciones lo aprovechan sin reescribirse.

---

## 7. Arranque inicial del sistema

- Al iniciar por primera vez se crea un **super administrador "root"**.
- El root configura el SSO: Unidades Académicas, aplicaciones, módulos, roles, grupos y los primeros administradores.
- En su primer inicio el root debe **cambiar su contraseña y activar doble factor**; para la operación diaria se usan cuentas de administrador nominales.

---

## 8. Autenticación (inicio de sesión)

### 8.1 Métodos
- **Usuario (CURP/RFC o correo) + contraseña.**
- **Google Workspace** mediante **OAuth 2.0 / OpenID Connect**.

### 8.2 Política de contraseñas (estándar NIST SP 800-63B)
- Longitud mínima **8 caracteres** (12 para administradores); se permiten al menos 64.
- Se aceptan todos los caracteres, incluidos espacios (frases de contraseña).
- Se **rechazan contraseñas comunes, filtradas** o que contengan datos del usuario (nombre, CURP, matrícula).
- **Sin caducidad periódica forzada**; el cambio es obligatorio solo si hay sospecha de compromiso.
- No se permite reutilizar las **últimas 5** contraseñas.

### 8.3 Almacenamiento de contraseñas
- Las contraseñas se guardan con **hash**; nunca en texto plano ni de forma reversible.
- Algoritmo: **Argon2id** con sal única por usuario (estándar recomendado por OWASP). *Se descartó SHA-1* porque es demasiado rápido de calcular: si la base de datos se filtrara, las contraseñas se podrían descifrar masivamente.
- Los parámetros del algoritmo se guardan junto a cada hash, para poder **endurecerlos en el futuro** sin que los usuarios cambien su contraseña (se re-calcula al siguiente inicio de sesión).
- **Migración de sistemas actuales:** si el sistema actual guarda contraseñas en SHA-1 u otro formato, se importan tal cual y se **convierten automáticamente a Argon2id** la siguiente vez que cada usuario inicie sesión; las cuentas que no inicien sesión en un plazo definido deberán restablecer su contraseña.

### 8.4 Doble factor de autenticación (2FA)
- Se configura **por cuenta**: el administrador puede activarlo o exigirlo a una cuenta en particular.
- También se puede **exigir por rol** (obligatorio para root y administradores).
- Métodos: aplicación autenticadora (TOTP, p. ej. Google Authenticator) y, como respaldo, código por correo.
- Códigos de recuperación de un solo uso por si el usuario pierde su dispositivo.

### 8.5 Bloqueo por intentos fallidos
- Después de **N intentos** incorrectos la cuenta se bloquea **temporalmente X minutos** (configurables).
- El conteo considera la cuenta **y la IP**, para evitar que un tercero bloquee cuentas ajenas a propósito.

### 8.6 Sesiones concurrentes
- Por defecto, como máximo **una sesión en computadora y una en dispositivo móvil** al mismo tiempo.
- El **administrador puede configurar esta regla por rol**.
- Ante una segunda sesión del mismo tipo: rechazar el nuevo inicio o cerrar la anterior *(decisión pendiente)*.

### 8.7 Cierre por inactividad
- Si la sesión no tiene actividad durante un tiempo configurable (puede variar por rol), se cierra automáticamente.

---

## 9. Registro de cuentas (autoservicio)

> **Dónde se registra:** el registro **no está en las interfaces del SSO**. Lo implementa **cada aplicación autorizada** para crear cierto tipo de cuenta, con su propio formulario y el **componente de registro del SDK MiTSJ**, que llama a la **API de registro del SSO**. Las validaciones de identidad, la creación de la cuenta y la contraseña **siempre** las resuelve el SSO.

### 9.1 ¿Quién puede crear su propia cuenta?

| Tipo de cuenta | ¿Auto-registro? | Aplicación donde se registra (ejemplo, configurable) | ¿Requiere activación de un administrador? |
|---|---|---|---|
| **Aspirante** (inicia como *Interesado*) | ✅ Sí | Admisión | ❌ No – se activa sola al validar correo y celular |
| **Organismo** (Empresa / Institución) | ✅ Sí | Vinculación | ✅ Sí |
| **Docente** | ✅ Sí | Recursos Humanos | ✅ Sí |
| **Administrativo** | ✅ Sí | Recursos Humanos | ✅ Sí |
| **Proveedor** | ✅ Sí | Compras / Proveedores | ✅ Sí |
| **Alumno** | ❌ No | — | — (usa la cuenta que tuvo como aspirante) |

### 9.1.1 Reparto de responsabilidades

| Lo hace la aplicación (con el SDK) | Lo hace el SSO (API de registro) |
|---|---|
| Formulario de registro con su propio diseño | Validar el CAPTCHA y límites por IP |
| Capturar CURP/RFC, nombre, correo, celular, UA | Validar la CURP internamente y luego con RENAPO (§4.4) |
| Capturar los datos propios de la aplicación (carrera, empresa, documentos…) | Revisar duplicados y consultar Google Workspace |
| Guardar esos datos propios en **su** base de datos, ligados al identificador de la cuenta | Registrar la aceptación del aviso de privacidad |
| Mostrar el resultado al usuario | Verificar correo y celular |
| | Crear la cuenta y aplicar las reglas de activación (§9.3) |
| | **Creación de la contraseña** en el Acceso del SSO |

- **La aplicación nunca captura ni ve la contraseña.**
- La aplicación se identifica ante el SSO con sus credenciales de cliente; solo puede registrar los **tipos de cuenta que tiene permitidos** (§3.1).

### 9.2 Flujo de registro
1. En la **aplicación correspondiente** (p. ej. Admisión), el usuario captura sus datos (CURP o RFC, correo, celular, fotografía cuando aplique) y su Unidad Académica, y **acepta el aviso de privacidad**. El SDK envía los datos de identidad a la **API de registro del SSO**; el tipo de cuenta lo determina la aplicación.
2. Protección contra registros automatizados: **CAPTCHA** y límite de registros por IP.
3. El sistema **verifica en Google Workspace** (vía OAuth 2.0 / API de directorio) si el correo ya existe:
   - Si **no existe** → se da de alta normalmente.
   - Si **ya existe** → la cuenta del SSO se **asocia** a esa cuenta de Google Workspace.
4. **Validaciones internas** de la CURP/RFC: formato, dígito verificador y que **no esté registrada** previamente (§4.4 pasos 1–5).
5. Solo si pasan las validaciones internas: **validación con RENAPO** (§4.4 paso 6).
6. Validaciones obligatorias para todas las cuentas:
   - **Correo:** el SSO envía un enlace **"Confirma tu correo y crea tu contraseña"** que abre el **Acceso del SSO**; ahí el usuario crea su contraseña (cumpliendo §8.2). Así se verifica el correo y se crea la contraseña en un solo paso, sin que la aplicación la vea. *(Si la cuenta se asoció a Google Workspace, puede iniciar sesión con Google y la contraseña local es opcional.)*
   - **Celular:** código por SMS/WhatsApp, que puede capturarse en la aplicación (vía SDK) o en el Acceso del SSO.
7. **Aspirantes:** quedan **activos** automáticamente.
8. **Demás tipos:** quedan **pendientes de activación** (ver 9.3).

### 9.3 ¿Quién activa las cuentas pendientes?

| Administrador | Qué cuentas ve | Qué puede activar |
|---|---|---|
| **Administrador global** (Dirección General) | **Todas**, incluidas las de aspirantes | Cualquier cuenta pendiente |
| **Administrador de Unidad Académica** | Las de **su** Unidad Académica | Las de su UA **que no sean de aspirante** |

- El administrador puede **aprobar o rechazar** (con motivo); el usuario recibe correo en ambos casos.
- Queda registro de quién activó o rechazó cada cuenta y cuándo.

### 9.4 Carga masiva y migración
- Importación masiva de cuentas (alumnos, docentes y administrativos actuales) desde archivo o desde el sistema que se usa hoy.
- Las cuentas importadas reciben un correo para **establecer su contraseña**.

---

## 10. Recuperación de contraseña

- El usuario se identifica con su **CURP/RFC** o su **correo** (en Organismos, la del representante o del usuario del organismo).
- Se envía un **correo con un enlace** para cambiar la contraseña, de **un solo uso** y con **caducidad** configurable.
- El usuario captura su **nueva contraseña** (cumpliendo §8.2).
- Al cambiarla se **cierran las sesiones abiertas** y se notifica al usuario.
- Las cuentas que inician sesión con Google Workspace recuperan su contraseña desde Google.

---

## 11. Ciclo de vida de las cuentas

- **Las cuentas nunca se eliminan.** Solo cambian de estado: *Pendiente → Activa → Inactiva* (y *Bloqueada* temporalmente por intentos fallidos).
- La **inactivación la controla el administrador** (global o de Unidad Académica, según su alcance), indicando el motivo.
- Una cuenta inactiva **no puede iniciar sesión**. Si el usuario quiere volver a usarla, solicita la reactivación y **un administrador la activa**.
- Los cambios de perfil (p. ej. Alumno → Egresado, Docente que deja el instituto) los **reportan las aplicaciones** responsables (ver §5).
- Historial completo de estados de la cuenta: quién, cuándo y por qué.

### 11.1 Alta y baja automática en Google Workspace
- Al activarse una cuenta que requiere correo institucional (p. ej. alumno inscrito, docente o administrativo), el SSO **crea automáticamente** su cuenta en Google Workspace (vía OAuth 2.0 / API de administración).
- Al **inactivarse** la cuenta en el SSO, la cuenta de Google Workspace se **suspende** (no se elimina).
- Al **reactivarse**, se reactiva también en Google Workspace.

---

## 12. Control de acceso físico (torniquetes Hikvision)

Los torniquetes y terminales de acceso son **Hikvision**. Estos equipos **deciden el acceso localmente** con su propia lista de personas (foto, tarjeta, vigencia y permisos), incluso sin red. Por eso el SSO **no se consulta en cada paso**: el SSO es la **fuente de verdad** y **sincroniza** la información hacia Hikvision.

### 12.1 Esquema de integración
```
SSO  ──(alta/cambio/baja)──►  HikCentral (por UA o central)  ──►  Terminales y torniquetes
SSO  ◄──(eventos de paso)───  HikCentral
```
- **Opción recomendada:** integración con **HikCentral** (Professional / Access Control) mediante su **OpenAPI**; un solo punto para las 13 Unidades Académicas.
- **Alternativa:** integración directa con cada equipo vía **ISAPI** (más trabajo de mantenimiento al haber muchos equipos).
- Se desarrolla un **conector Hikvision** (microservicio) que traduce los cambios del SSO al formato de Hikvision.

### 12.2 Qué sincroniza el SSO
| Dato del SSO | Equivalente en Hikvision |
|---|---|
| Identificador interno de la cuenta (no la CURP) | Número de empleado / ID de persona |
| Nombre | Nombre de la persona |
| Fotografía validada | Foto de rostro |
| Credencial (tarjeta/QR), si aplica | Tarjeta |
| Cuenta activa + vigencia del perfil | Fecha de inicio y fin de validez |
| Unidad Académica / instalaciones permitidas | Grupo / nivel de acceso |
| Horarios permitidos por rol (si aplica) | Plantilla de horario |
| Vehículos validados (placa, vigencia, estacionamientos) | Lista de placas autorizadas (ver §12.4) |

- **Inactivar** una cuenta o vencer su vigencia **retira el acceso** en los torniquetes en un tiempo máximo configurable (objetivo: menos de 5 minutos).
- **Sincronización incremental** en cada cambio y **conciliación completa** diaria (detecta y corrige diferencias).
- Las personas **no se dan de alta ni se modifican directamente en HikCentral**: todo cambio se hace en el SSO.
- Accesos **temporales** para organismos, proveedores y visitantes, con vigencia.

### 12.3 Eventos de acceso
- El conector **recibe los eventos de paso** de Hikvision (permitido / rechazado, persona, torniquete, Unidad Académica, fecha y hora) y los registra en la bitácora de seguridad (§13.2) y el dashboard (§13.4).

### 12.4 Vehículos y control de plumas (lectura de placas)
El acceso vehicular se controla con **cámaras Hikvision de lectura de placas** que abren o mantienen cerrada la **pluma** según la placa capturada en la foto. Para ello el SSO lleva el **registro de vehículos** de cada usuario.

**Registro de vehículos**
- Un usuario puede registrar **uno o varios vehículos** (límite configurable por rol).
- Datos por vehículo: **placa**, estado/país de emisión, marca, modelo, color, tipo (auto, moto, etc.) y **fotografía del vehículo con la placa visible** (y, opcional, tarjeta de circulación), almacenadas en el **CDN institucional** (§3.5).
- El usuario puede registrar sus vehículos desde su cuenta; quedan **pendientes hasta que un administrador** (global o de la Unidad Académica) los valide.
- Una placa **no puede estar activa en dos cuentas** a la vez (salvo autorización expresa, p. ej. vehículo compartido).
- Cada vehículo tiene **vigencia** y **Unidades Académicas / estacionamientos permitidos**; al inactivarse la cuenta, sus vehículos se inactivan.
- Accesos vehiculares **temporales** para visitantes, organismos y proveedores.
- Historial de cambios de cada vehículo (alta, validación, cambio, baja).

**Sincronización con Hikvision**
- El conector envía a HikCentral la **lista de placas autorizadas** (lista permitida) con su vigencia y estacionamientos; las placas inactivas o vencidas se retiran.
- Se puede manejar una **lista de placas bloqueadas** (p. ej. vehículos con reporte).

**Eventos vehiculares**
- Cada lectura (entrada/salida, permitida/rechazada) se registra con: placa leída, **foto capturada por la cámara**, cuenta asociada, pluma/acceso, Unidad Académica, fecha y hora.
- Opcional: ocupación del estacionamiento en tiempo real en el dashboard.

*Nota:* la lectura de placa por sí sola puede engañarse con placas clonadas; para zonas sensibles se puede exigir además la credencial o el rostro del conductor.

### 12.5 Seguridad de la red de equipos
- Los equipos Hikvision y HikCentral en una **red (VLAN) aislada**, sin exposición a Internet, con **firmware actualizado** y contraseñas de fábrica cambiadas.
- Las credenciales de la OpenAPI de HikCentral se guardan como secretos, fuera del código.

---

## 13. Sesiones, bitácora y monitoreo

### 13.1 Registro de sesiones
- Fecha/hora de **inicio** y **cierre**, y **motivo de cierre** (manual, inactividad, sesión nueva, bloqueo, administrador).
- **Dirección IP** de origen.
- **Datos del navegador**: navegador y versión, sistema operativo, tipo de dispositivo, idioma y zona horaria.

### 13.2 Bitácora de seguridad del SSO
El SSO registra **sus propios eventos**, ligados a la sesión:
- Inicios de sesión exitosos y fallidos, cierres, bloqueos y desbloqueos.
- Registro, validación, activación, rechazo, inactivación y reactivación de cuentas.
- Cambios de contraseña, recuperaciones y configuración de 2FA.
- Cambios en aplicaciones, módulos, roles, grupos, permisos, alcances, vigencias, suplencias y llaves.
- Accesos por torniquete y por pluma vehicular; registro y validación de vehículos.
- Todas las acciones del root y de los administradores.

La bitácora **no se puede editar ni borrar** desde la aplicación.

### 13.3 "Ver como" (soporte)
- Un administrador autorizado puede **ver el sistema como lo ve otro usuario** para diagnosticar problemas, **sin poder hacer cambios**, y queda registrado.

### 13.4 Dashboard de monitoreo en tiempo real
- Cuentas con **sesión abierta en este momento**, desde dónde y desde cuándo.
- **Actividad en vivo** de cada sesión (accesos a aplicaciones, uso de permisos y llaves).
- Alertas: intentos fallidos, bloqueos, accesos fuera de horario o desde ubicaciones inusuales.
- Acciones inmediatas: **cerrar una sesión** o **inactivar una cuenta**.

---

## 14. Gobierno de accesos

- **Revisión periódica de accesos:** cada semestre (configurable), los responsables de cada área confirman qué personas deben conservar sus permisos; lo no confirmado se marca para revisión.
- **Reportes:** quién tiene acceso a qué, permisos por rol/grupo, cuentas inactivas, cuentas sin 2FA, permisos por vencer; exportables (Excel/PDF) para auditorías.

---

## 15. Requerimientos no funcionales

- **Alta disponibilidad:** el SSO es el punto de entrada de todo el ERP; debe operar en al menos dos instancias, con base de datos replicada, respaldos diarios y plan de recuperación ante desastres.
- **Rendimiento:** soportar los picos de inscripciones y admisión (meta a definir de usuarios concurrentes).
- **Seguridad:** todo el tráfico por **HTTPS**; secretos y llaves fuera del código.
- **Parámetros configurables** desde la consola: intentos, minutos de bloqueo, inactividad, caducidad de enlaces/llaves, sesiones por rol, 2FA por rol.
- **Protección de datos personales** (*Ley General de Protección de Datos Personales en Posesión de Sujetos Obligados*):
  - **Evaluación de impacto en la protección de datos** antes de poner en operación la fotografía, el reconocimiento facial, las placas y los registros de entrada/salida.
  - **Aviso de privacidad** integral y simplificado; **consentimiento expreso** para fotografía/rostro (dato biométrico sensible) desde la etapa 1, ya que las terminales Hikvision pueden hacer reconocimiento facial.
  - **Cifrado en reposo** de fotografías, datos biométricos, placas y datos de contacto (los archivos, en el CDN institucional con URLs firmadas); **acceso restringido** solo a roles autorizados y registrado en bitácora.
  - **Minimización:** a Hikvision se envía solo lo necesario (identificador interno, nombre, foto, vigencia); nunca la CURP.
  - **Plazos de conservación** definidos y depuración automática (propuesta, a validar con el área jurídica): eventos de acceso peatonal/vehicular y sus fotos 1 año; bitácora de seguridad 5 años; fotos de cuentas inactivas se conservan mientras la cuenta exista.
  - Atención a **derechos ARCO** (acceso, rectificación, cancelación y oposición) desde la sección *Mi cuenta* de la Consola del SSO.
- **Pruebas de seguridad:** pruebas de conformidad OpenID, análisis de vulnerabilidades del código y **prueba de penetración** externa antes de cada fase en producción.
- **Notificaciones:** el envío de correos y SMS/WhatsApp se hace a través de un **servicio de notificaciones** compartido del ERP.
- **Ambientes** separados de desarrollo, pruebas y producción; API versionada.

---

## 16. Entregables

1. Microservicio backend del SSO (PostgreSQL, OAuth 2.0 / OIDC).
2. **Interfaces del SSO (dos):**
   - **Acceso del SSO** (público): login (incluye Google Workspace), 2FA, recuperación de contraseña y **creación de contraseña** al confirmar el correo. Es la pantalla a la que redirigen todas las aplicaciones.
   - **Consola del SSO** (usuarios autenticados; opciones según el rol): mis datos, contraseña, 2FA, sesiones abiertas, foto, vehículos, derechos ARCO, administración (punto 5) y dashboard en tiempo real (punto 8).
3. **API de registro** de cuentas para las aplicaciones autorizadas *(el formulario de registro lo implementa cada aplicación con el SDK)*.
4. **SDK MiTSJ** (backend en C#/.NET como paquete NuGet, y frontend), API de permisos (§6.12) y **motor de API keys** con su API de validación (§3.7), para que las aplicaciones validen sesiones y apliquen permisos, filtros y campos visibles.
5. Sección de *Administración* de la Consola: aplicaciones, **API keys**, módulos, formularios, campos, acciones, roles, grupos, cuentas, organismos, alcances, vigencias, suplencias, llaves, separación de funciones y reportes.
6. Conector Hikvision (sincronización de personas, fotos, permisos y placas vehiculares hacia HikCentral y recepción de eventos de acceso peatonal y vehicular).
7. Módulo de registro y validación de vehículos.
8. Dashboard de monitoreo en tiempo real (dentro de la Consola del SSO).
9. Documentación: API, manual de administrador, guía de integración y uso del SDK para aplicaciones.

La entrega de cada componente sigue el plan por fases (§19).

---

## 17. Decisiones pendientes

- [ ] **RENAPO:** mecanismo de acceso (convenio institucional con RENAPO/SEGOB u otro proveedor autorizado), cupo y costo de consultas, datos que devuelve el servicio y vigencia de reutilización de una validación.
- [ ] **CDN:** tecnología en que está hecho (¿puede validar JWT / usar el SDK MiTSJ?), cómo distingue hoy archivos públicos y privados (carpeta, bandera, cubeta), qué aplicaciones usan sus API keys y calendario de migración.
- [ ] Framework del frontend (para el SDK de frontend y las interfaces del SSO).
- [ ] Validar con el área jurídica los plazos de conservación propuestos y el aviso de privacidad (§15).
- [ ] Ante una segunda sesión del mismo tipo de dispositivo: ¿rechazar el nuevo inicio o cerrar el anterior?
- [ ] Valores por defecto: N intentos, minutos de bloqueo, tiempo de inactividad, caducidad de enlaces/llaves.
- [ ] Canal de verificación de celular (SMS, WhatsApp) y proveedor.
- [ ] Niveles jerárquicos que pueden generar llaves excepcionales y para qué acciones.
- [ ] Dominio(s) de Google Workspace a integrar y qué perfiles reciben correo institucional.
- [ ] **Hikvision:** ¿se cuenta con HikCentral (versión y licencia de OpenAPI) o se integrará equipo por equipo (ISAPI)? ¿Un HikCentral central o uno por Unidad Académica?
- [ ] **Hikvision:** modelos de terminales instaladas (¿reconocimiento facial, tarjeta, QR?) y requisitos de foto de esos modelos.
- [ ] Medio de identificación en torniquetes (rostro, tarjeta, QR en celular).
- [ ] **Vehículos:** número máximo de vehículos por rol; qué roles tienen acceso vehicular; ¿se exige tarjeta de circulación?; modelos de cámaras de lectura de placas instaladas.
- [ ] Perfiles que requieren fotografía obligatoria.
- [ ] Meta de usuarios concurrentes para el dimensionamiento.

---

## 18. Fuera del alcance del SSO

- **Ciclo académico de la persona** (Interesado → Aspirante → Alumno → Egresado, bajas) y sus reglas: Admisión / Servicios Escolares. El SSO solo recibe los cambios.
- **Reglas de negocio por periodo** (p. ej. periodo de bajas): cada aplicación.
- **Registro de cambios en los datos de cada aplicación** (valor anterior / nuevo): **bitácora central** del ERP.
- **Envío de correos y mensajes:** servicio de **notificaciones** del ERP.
- **Calendario institucional de periodos:** lo manejan las aplicaciones que lo necesiten (p. ej. Servicios Escolares); el SSO no depende de él. Las vigencias del SSO se definen solo con fechas de inicio y fin.
- **Almacenamiento de archivos:** **CDN institucional** (API propia del ERP, registrada como aplicación del SSO); el SSO solo guarda referencias (§3.5).
- **Reconocimiento facial** (etapa 2).

---

## 19. Plan de implementación por fases

| Fase | Objetivo | Alcance |
|---|---|---|
| **0. Levantamiento** | Conocer el punto de partida antes de diseñar a detalle | Inventario de equipos Hikvision (terminales, cámaras de placas, ¿HikCentral?); sistemas actuales y su formato de contraseñas; número de usuarios por tipo y UA; dominios de Google Workspace; catálogo inicial de roles por UA; evaluación de impacto en protección de datos. |
| **1. Núcleo (MVP para Servicios Escolares)** | Que Servicios Escolares pueda operar con el SSO | Cuentas e identidad (CURP/RFC); validación de CURP con RENAPO; API de registro (con el registro de Admisión para aspirantes), validaciones y activación; login con contraseña y Google; 2FA; política y almacenamiento de contraseñas; recuperación; roles, grupos, alcance por UA y permisos por registro asignado; API de permisos y SDK MiTSJ; sesiones, inactividad y bloqueo; bitácora de seguridad; sección *Mi cuenta*; consola de administración básica; carga masiva/migración.; registro del CDN como aplicación y cuenta de servicio del SSO en el CDN; **motor de API keys** (para el CDN y sistemas externos). |
| **2. Gobierno y control fino** | Control detallado y supervisión | Permisos por campo; llaves excepcionales; suplencias; separación de funciones; "ver como"; dashboard en tiempo real; revisión periódica de accesos y reportes; alta/suspensión automática en Google Workspace; organismos multiusuario.; migración de las apps existentes de API keys del CDN a tokens del SSO. |
| **3. Acceso físico** | Torniquetes y plumas | Fotografía con validación; conector Hikvision (personas, vigencias, niveles de acceso); registro y validación de vehículos; placas autorizadas; eventos peatonales y vehiculares en bitácora y dashboard. |
| **4. Biometría y mejoras** | Reconocimiento facial y extensiones | Reconocimiento facial (si no se cubrió en la fase 3 con las terminales); doble verificación placa + rostro en zonas sensibles; nuevas aplicaciones del ERP. |

Cada fase termina con pruebas de seguridad (§15) antes de pasar a producción.

---

## Bitácora del documento

| Fecha | Cambio |
|---|---|
| 14-oct-2026 | v0.1: Versión inicial a partir de los requerimientos dictados por Gabriel. |
| 14-oct-2026 | v0.2: Se elimina la captura de MAC Address; se registran IP y datos del navegador. Se define quién puede auto-registrarse y qué administrador activa cada cuenta. |
| 14-oct-2026 | v0.3: Una sesión por tipo de dispositivo, configurable por rol. Se retiran del SSO responsabilidades ajenas (ciclo académico, reglas por periodo, bitácora de datos). |
| 14-oct-2026 | v0.4: RFC como identificador alterno; organismos con varios usuarios; Google Workspace vía OAuth 2.0 con alta/suspensión automática; 2FA configurable por cuenta y rol; fotografía ligada a la cuenta y control de acceso por torniquetes; política de contraseñas NIST; ciclo de vida sin eliminación (inactivación/reactivación por administrador); perfil Egresado; permisos por registro asignado y por campo; suplencias; separación de funciones; registro de aplicaciones cliente y cuentas de servicio; "ver como"; revisión de accesos y reportes; carga masiva; corrección/fusión de identidad; alta disponibilidad y LGPDPPSO. |
| 14-oct-2026 | v0.5: Torniquetes **Hikvision**: el SSO sincroniza personas, fotos, vigencias y niveles de acceso hacia HikCentral (OpenAPI) o equipos (ISAPI) y recibe los eventos de paso (§12). |
| 14-oct-2026 | v0.6: Registro de **vehículos y placas** por usuario, validación por administrador y sincronización de placas autorizadas con las cámaras Hikvision que controlan las plumas; eventos vehiculares con foto (§12.4). |
| 14-oct-2026 | v0.7: Desarrollo propio inspirado en Keycloak (§3.3); cómo se aplican los permisos en las aplicaciones: API de permisos, relaciones persona↔registro, SDK MiTSJ, caché e invalidación (§6.11); contraseñas en **Argon2id** con migración desde SHA-1 (§8.3); protección de datos reforzada y pruebas de seguridad (§15); plan por fases 0–4 (§19). |
| 15-oct-2026 | v0.8: Backend en **C# / .NET (ASP.NET Core)**; tecnología de referencia (§3.4); sección del **SDK MiTSJ** con sus funciones y presentación como paquete NuGet (§6.12). |
| 15-oct-2026 | v0.9: Validación de CURP con **RENAPO** como último paso, después de validaciones internas (formato, dígito verificador, duplicados, validación previa), con control de consumo, protección de datos y manejo de indisponibilidad (§4.4, §9.2). |
| 15-oct-2026 | v0.10: El **registro de cuentas se implementa en cada aplicación autorizada** (Admisión, Vinculación, RH, Compras…) con el SDK y la **API de registro del SSO**; el SSO conserva validaciones de identidad, creación de cuenta y **creación de contraseña** (enlace por correo al portal del SSO); cada app solo registra los tipos de cuenta permitidos (§3.1, §9, §6.12, §16). |
| 15-oct-2026 | v0.11: Las interfaces del SSO se unifican en **una sola Consola del SSO** (Acceso · Mi cuenta · Administración + Dashboard, según el rol) (§3, §3.3, §16). |
| 15-oct-2026 | v0.12: Todos los archivos (fotos, documentos) se almacenan en el **CDN institucional** (API propia externa al SSO); el SSO solo guarda identificadores; archivos privados con URLs firmadas (§3.5). |
| 15-oct-2026 | v0.13: Las interfaces del SSO quedan en **dos**: **Acceso del SSO** (login, 2FA, recuperación, crear contraseña) y **Consola del SSO** (mis datos, contraseña, foto, vehículos, sesiones, administración y dashboard según rol). |
| 15-oct-2026 | v0.14: El **CDN institucional se registra como aplicación gestionada por el SSO** (acepta tokens, permisos por módulo/acción, públicos sin token, privados con URL firmada) y a la vez es el **almacén de archivos del SSO**; se documenta cómo evitar la dependencia circular y la migración de sus API keys (§3.5). |
| 15-oct-2026 | v0.15: Se agrega al núcleo el **motor de API keys** (emisión, permisos, hash, vigencia, rotación, revocación, validación, bitácora) para el CDN y otros sistemas externos; las API keys del CDN pasan a ser gestionadas por el SSO (§3.2, §3.5, §3.7). |
| 15-oct-2026 | v0.16: Se quita la dependencia del calendario institucional: las **vigencias de permisos se definen solo con fechas de inicio y fin** (§6.7); el calendario queda fuera del SSO (§18). |


---

### Fuente 2: arquitectura.md (v0.9)

# Arquitectura general – SSO MiTSJ

> Complemento de [`sso.md`](sso.md) · Versión 0.9 · 15-oct-2026
> Vista de alto nivel; los detalles técnicos se confirman en el diseño técnico.

---

## 1. Diagrama general

El **SSO es el proveedor de identidad directo de las aplicaciones**: cada aplicación del ERP delega en él el inicio de sesión (OAuth 2.0 / OpenID Connect) y le consulta los permisos. El SSO tiene **dos interfaces**: el **Acceso del SSO** (login, 2FA, recuperación, crear contraseña) y la **Consola del SSO** (mis datos, contraseña, foto, vehículos, sesiones, administración y dashboard, según el rol). El **registro de cuentas** lo implementan las aplicaciones autorizadas con el SDK.

### 1.1 Imagen

![Arquitectura general del SSO MiTSJ](arquitectura.png)

> Fuente editable: [`arquitectura.svg`](arquitectura.svg) (se abre en el navegador, Inkscape, draw.io o Figma).

### 1.2 Vista por capas (texto)

```
┌─ SSO MiTSJ · PROVEEDOR DE IDENTIDAD (C# / .NET · ≥2 instancias) ─────────────────────────┐
│ Interfaces del SSO:                                                                      │
│ ┌────────────────────────┐  ┌────────────────────────────────────────────────────────┐   │
│ │     ACCESO DEL SSO     │  │                    CONSOLA DEL SSO                     │   │
│ │      login · 2FA       │  │       mis datos · contraseña · foto · vehículos        │   │
│ │      recuperación      │  │         sesiones · administración · dashboard          │   │
│ │    crear contraseña    │  │            (opciones visibles según el rol)            │   │
│ └────────────────────────┘  └────────────────────────────────────────────────────────┘   │
│                                                                                          │
│ Núcleo:                                                                                  │
│ ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐  │
│ │Servidor OAuth/OIDC│ │ Identidad/registro│ │   Autenticación   │ │ Motor de permisos │  │
│ │    (OpenIddict)   │ │ CURP/RFC · RENAPO │ │Argon2id·Google·2FA│ │puede·cuáles·campos│  │
│ └───────────────────┘ └───────────────────┘ └───────────────────┘ └───────────────────┘  │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │
│ │  Admin API   │ │Motor API keys│ │   Bitácora   │ │  Monitoreo   │ │ Procesos 2º  │     │
│ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘     │
└──────────────┬────────────────────────────────┬────────────────────────────────┬─────────┘
               │ OIDC: login y tokens           │ lee / escribe                  │ eventos (bus)
               │ API de permisos                │                                │
               ▼                                ▼                                ▼
┌─ APLICACIONES DEL ERP ──────┐ ┌─ DATOS DEL SSO ───────────────┐ ┌─ CONECTORES ───────────────┐
│ Servicios Escolares         │ │ PostgreSQL replicado          │ │ Google Workspace           │
│ Finanzas · Otras apps…      │ │ Redis: caché · sesiones       │ │ alta·suspensión·reactiv.   │
│ Registro: Admisión, RH, etc.│ │ Referencias a archivos (CDN)  │ │                            │
│ Cada app: su propia BD      │ │ Bus de mensajes (RabbitMQ)    │ │ Hikvision                  │
│ + SDK MiTSJ (NuGet):        │ │  → invalida caché de apps     │ │ personas·fotos·placas      │
│ · valida token              │ │  → avisa a conectores         │ │ ← eventos de paso          │
│ · ¿puede?/¿cuáles?/¿campos? │ │                               │ │                            │
│ · registro → API del SSO    │ │                               │ │                            │
└──────────────┬──────────────┘ └───────────────────────────────┘ └─────────────┬──────────────┘
               ▼                                                                 ▼
┌─ SERVICIOS COMPARTIDOS ERP ─┐ ┌─ SISTEMAS EXTERNOS ────────────────────────────────────────┐
│ Bitácora central            │ │ RENAPO (CURP) · Google Workspace                           │
│ Notificaciones              │ │ ┌─ Red aislada (VLAN) ───────────────────────────────────┐ │
│ CDN (app gestionada por SSO)│ │ │ HikCentral → torniquetes · cámaras de placas / plumas  │ │
└─────────────────────────────┘ │ └────────────────────────────────────────────────────────┘ │
                                └────────────────────────────────────────────────────────────┘
```

---

## 2. Capas y componentes

### 2.1 Interfaces del SSO (dos · frontend por definir)

| Interfaz | Para quién | Qué contiene |
|---|---|---|
| **Acceso del SSO** | Cualquier persona (pública) | Login (contraseña / Google), 2FA, recuperación y **creación de contraseña** al confirmar el correo. Es la pantalla a la que **todas las aplicaciones** redirigen para iniciar sesión. *No incluye registro.* |
| **Consola del SSO** | Usuarios autenticados — las opciones visibles dependen del rol | **Mis datos · contraseña · foto · vehículos · sesiones** (todos), derechos ARCO; **administración** (apps, módulos, formularios, roles, grupos, cuentas, activaciones, llaves, reportes) y **dashboard en tiempo real** (sesiones, alertas, consumo RENAPO, accesos físicos) para administradores |

Las pantallas de cada aplicación del ERP pertenecen a esa aplicación (ver §2.5).

### 2.2 Relación SSO ↔ aplicaciones
- El SSO actúa como **proveedor de identidad (IdP)**; cada aplicación es un **cliente registrado** (§3.1 de sso.md).
- **Inicio de sesión:** la app redirige al **Acceso del SSO** → el SSO autentica → devuelve el token a la app.
- **Permisos:** la app consulta al motor de permisos del SSO mediante el SDK MiTSJ.
- **Registro:** las apps autorizadas (Admisión, Vinculación, RH, Compras…) tienen su propio formulario y envían los datos de identidad a la **API de registro del SSO**; cada app solo puede crear los tipos de cuenta que tiene permitidos.
- **De regreso:** la app informa al SSO las relaciones (docente ↔ grupo) y los cambios de perfil (alumno → egresado).
- **Invalidación:** el bus de mensajes avisa a las apps cuando cambia un permiso o se inactiva una cuenta.

### 2.3 SSO MiTSJ (C# / .NET)
Un **servicio desplegable** organizado en **módulos internos** (más sencillo de operar que dividirlo en muchos microservicios), con al menos **dos instancias** para alta disponibilidad.

| Módulo | Responsabilidad | Ref. sso.md |
|---|---|---|
| Servidor OAuth 2.0 / OIDC | Emite y revoca tokens; registro de apps cliente y cuentas de servicio | §3.1–3.3 |
| Identidad y registro | Cuentas, perfiles, organismos, validación CURP (interna → RENAPO), activaciones, ciclo de vida | §4, §5, §9, §11 |
| Autenticación | Contraseña (Argon2id), Google Workspace, 2FA, acciones obligatorias, bloqueo, sesiones, inactividad | §8, §10 |
| Motor de permisos | Decide ¿puede? / ¿sobre cuáles? / ¿qué campos?; roles, grupos, alcances, relaciones, vigencias, llaves, suplencias, separación de funciones | §6 |
| API de administración | Respaldo de la consola | §16 |
| Bitácora de seguridad | Eventos de seguridad inalterables | §13.2 |
| Monitoreo | Envía al dashboard la actividad en vivo (SignalR) | §13.4 |
| Motor de API keys | Emite, valida, rota y revoca API keys para sistemas que no usan tokens (p. ej. el CDN); permisos del catálogo, hash, vigencia, bitácora | §3.7 |
| Procesos en segundo plano | Vencimiento de vigencias, reintentos RENAPO, conciliación con Hikvision y Google | §6.7, §4.4, §12 |

### 2.4 Datos e infraestructura
| Componente | Uso |
|---|---|
| **PostgreSQL** (replicado) | Base de datos propia del SSO |
| **Redis** | Caché de permisos, sesiones activas, límites de intentos |
| **Referencias a archivos** | El SSO solo guarda el identificador y metadatos de cada archivo; los archivos viven en el **CDN institucional** (§2.6) |
| **Bus de mensajes** (RabbitMQ) | Eventos: cambio de permisos (invalidación de caché), alta/baja de cuentas, sincronización con conectores |

### 2.5 Aplicaciones del ERP
- Cada app tiene **sus propias pantallas** (con el SDK MiTSJ de frontend), **su propia base de datos** y usa el **SDK MiTSJ (NuGet)** en su backend.
- El SDK valida el token, consulta al motor de permisos, aplica filtros en EF Core y reporta relaciones (docente↔grupo) y cambios de perfil (alumno→egresado).

### 2.6 Servicios compartidos del ERP (fuera del SSO)
- **Bitácora central:** cambios de datos de las apps (valor anterior / nuevo).
- **Notificaciones:** correos y mensajes (el SSO lo usa para verificaciones, recuperaciones, activaciones).
- **CDN institucional (API propia) — doble relación con el SSO:**
  - Es una **aplicación registrada y gestionada por el SSO**: acepta tokens del SSO y consulta permisos (subir · leer · reemplazar · eliminar · hacer público). Archivos **públicos** con enlace directo sin token; **privados** con URL firmada de corta duración solo si el SSO lo permite.
  - Es el **almacén de archivos del SSO**: fotos de cuentas y vehículos, documentos; el SSO y el conector Hikvision lo usan con cuentas de servicio.
  - Sus **API keys** pasan a ser **emitidas y gestionadas por el motor de API keys del SSO** (sso.md §3.5, §3.7).

### 2.7 Conectores
- **Google Workspace:** login federado, alta del correo institucional, suspensión/reactivación.
- **Hikvision:** sincroniza personas, fotos, vigencias, niveles de acceso y placas hacia HikCentral; recibe eventos de paso peatonal y vehicular.

### 2.8 Sistemas externos
- **RENAPO:** validación de CURP (último paso del registro).
- **Google Workspace.**
- **HikCentral → torniquetes y cámaras de placas**, en **red aislada**.

---

## 3. Flujos principales

### 3.1 Inicio de sesión
1. El usuario abre una app → el SDK lo envía al **login del SSO**.
2. El SSO autentica (contraseña o Google) → aplica **2FA** y **acciones obligatorias** si corresponde → revisa **sesiones concurrentes**.
3. El SSO devuelve un **token ligero** a la app; registra la sesión (IP, navegador) en la bitácora y en el dashboard.

### 3.2 Uso de una aplicación (permisos)
1. El jefe de carrera abre "Grupos" en Servicios Escolares.
2. El SDK pregunta al motor: **¿sobre cuáles?** → recibe "carrera = Sistemas, UA = Tala" y lo aplica a la consulta.
3. Al editar, el backend verifica **¿puede?**; el formulario muestra solo los **campos** permitidos.
4. Si la acción está fuera de periodo, el SDK pide una **llave excepcional** y la valida con el SSO.

### 3.3 Registro de una cuenta
1. El usuario llena el formulario **en la aplicación** que corresponde a su tipo de cuenta (p. ej. Admisión para aspirantes); la app guarda sus datos propios en su BD.
2. El SDK envía los datos de identidad a la **API de registro del SSO** → CAPTCHA → validaciones internas de CURP (formato, dígito verificador, duplicado, validación previa).
3. Solo si pasan → **RENAPO**.
4. Verificación en **Google Workspace** (cuenta existente o nueva).
5. El SSO envía el correo **"Confirma tu correo y crea tu contraseña"** → el usuario crea su contraseña en el **Acceso del SSO** (la app nunca la ve); código de verificación al celular.
6. Aspirante → activo; otros → pendiente de activación por admin global o de UA.

### 3.4 Cambio de permisos / inactivación
1. El administrador inactiva una cuenta o quita un permiso.
2. El SSO publica el evento en el **bus de mensajes**:
   - Las apps **invalidan su caché** (efecto en < 1 min).
   - El conector **Google** suspende el correo.
   - El conector **Hikvision** retira el acceso de torniquetes y plumas (< 5 min).

### 3.5 Archivos (CDN)
1. La app (o el SSO) envía el archivo **directo al CDN** con su token del SSO.
2. El CDN valida el token localmente y pregunta al motor de permisos si puede subirlo / leerlo.
3. Públicos: enlace directo. Privados: **URL firmada** de corta duración.
4. Quien guarda el archivo (app o SSO) solo conserva el **identificador del CDN**.
5. Si el CDN no está disponible, el login y los permisos siguen funcionando; solo se afectan fotos y documentos (reintentos).

### 3.6 Acceso físico
1. El torniquete / cámara decide **localmente** con la lista que sincronizó el SSO.
2. HikCentral envía el **evento de paso** al conector → bitácora de seguridad y dashboard.

---

## 4. Principios de la arquitectura

1. **El SSO es la fuente de verdad** de identidades y permisos; nadie da de alta personas en otro sistema.
2. **El SSO decide, las apps aplican** (vía SDK); el backend siempre verifica.
3. **Denegado por omisión.**
4. **Cada app con su base de datos**; el SSO nunca accede a ellas.
5. **Comunicación por eventos** para propagar cambios (bus de mensajes).
6. **Servicios externos aislados en conectores**, con reintentos y *circuit breaker*.
7. **Datos sensibles cifrados** y mínimos hacia terceros (nunca la CURP a Hikvision).
8. **Archivos fuera del SSO:** todo archivo va al CDN institucional, que a su vez es una aplicación gestionada por el SSO; el SSO solo guarda referencias y **no depende del CDN para autenticar**.
9. **Alta disponibilidad:** mínimo dos instancias del SSO y base de datos replicada.
10. **Infraestructura de red:** todo el tráfico por HTTPS detrás de un proxy inverso / balanceador (detalle de despliegue, no forma parte de la lógica del SSO).

---

## Historial

| Fecha | Cambio |
|---|---|
| 15-oct-2026 | v0.1: primera versión (diagrama Mermaid, capas, flujos y principios). |
| 15-oct-2026 | v0.2: el SSO se muestra como **proveedor de identidad directo** de las aplicaciones; se quita la capa de usuarios/navegador y el API Gateway del diagrama (queda como detalle de infraestructura); nuevo diagrama en imagen + SVG editable y vista en texto. |
| 15-oct-2026 | v0.3: el **registro** sale de las interfaces del SSO y lo implementan las aplicaciones autorizadas con el SDK + API de registro; el portal agrega **crear contraseña**; diagramas actualizados. |
| 15-oct-2026 | v0.4: las interfaces del SSO se unifican en **una sola Consola del SSO** (Acceso · Mi cuenta · Administración + Dashboard según rol); diagramas actualizados. |
| 15-oct-2026 | v0.5: se sustituye el almacén cifrado propio por el **CDN institucional** (API externa) para todos los archivos; diagramas actualizados. |
| 15-oct-2026 | v0.5.1: el diagrama muestra la Consola del SSO como **una sola caja** con todas sus funciones (sin divisiones entre acceso, Mi cuenta y administración). |
| 15-oct-2026 | v0.6: las interfaces del SSO quedan en **dos cajas**: Acceso del SSO y Consola del SSO; diagramas actualizados. |
| 15-oct-2026 | v0.7: el CDN pasa a ser **aplicación gestionada por el SSO** (tokens y permisos) además de almacén de archivos del SSO; flujo de archivos y diagrama actualizados. |
| 15-oct-2026 | v0.8: se agrega el **motor de API keys** al núcleo del SSO (diagramas y tabla de módulos). |
| 15-oct-2026 | v0.9: se quita el **Calendario** de los servicios compartidos (el SSO no depende de él); diagramas actualizados. |

---

## Pendientes

- [ ] Cerrar las decisiones de §17 de `sso` (RENAPO, CDN, valores por defecto, segunda sesión, verificación de celular, dominios de Google, Hikvision, niveles para llaves excepcionales). Bloquean historias concretas.
- [ ] Actualizar `backlog-sso` y `sprints` a la versión 0.16 de `sso` (siguen en 0.7). Agregar las historias nuevas (E3-09 a E18-05) y quitar E10-03.
- [ ] Definir el dominio del SSO y el proxy inverso (afecta cookies y CSP).
- [ ] Confirmar el gestor de paquetes y el registro del frontend (ya decidido: Bun; registro pendiente de confirmar).
- [ ] Pedir a diseño el estado de los componentes en Figma y los colores pendientes de `@mitsj/ui`.

## Estado Actual (resumen ejecutivo)

- **Requerimientos:** `sso` v0.16 y `arquitectura` v0.9, en borrador. Copias en las secciones de Fuentes de esta nota.
- **Código del SSO:** no existe todavía. `~/Documents/Developer/SSO` solo tiene documentos.
- **Frontend:** la librería `@mitsj/ui` está en `0.1.0-beta.1` (sin publicar), en el repo `mitsj-ui`. Tiene 13 componentes para la pantalla de Acceso.
- **Backend:** no iniciado.
- **Backlog y sprints:** desactualizados (v0.7 y borrador). Antes del planning hay que actualizarlos.

## Decisiones de Diseño

| Tema | Decisión | Fuente |
|---|---|---|
| Backend | Microservicio en C# / .NET LTS con ASP.NET Core, un solo servicio con módulos internos | `sso` §3; `arquitectura` §2.3 |
| Protocolo | OAuth 2.0 / OIDC con OpenIddict y JWT de corta duración | `sso` §3, §3.3 |
| Base de datos | PostgreSQL replicado con EF Core + Npgsql; Redis para caché y sesiones; RabbitMQ para eventos | `arquitectura` §2.4 |
| Contraseñas | Argon2id con sal única; parámetros guardados junto al hash | `sso` §8.3 |
| 2FA | TOTP (RFC 6238) con código de respaldo por correo y códigos de recuperación | `sso` §8.4 |
| Permisos | Motor propio: ¿puede?, ¿sobre cuáles?, ¿qué campos?; el token solo lleva identidad y roles resumidos | `sso` §6.11 |
| Alta disponibilidad | Mínimo dos instancias; base de datos replicada | `sso` §15 |
| Archivos | El CDN institucional guarda los archivos; el SSO solo guarda referencias | `sso` §3.5 |
| Frontend | Astro + islas React; el Acceso es estático; la Consola es una isla `client:only` | `planing front end` §1, §5 |
| Tokens en navegador | Ninguno en `localStorage`; cookie `HttpOnly` y `Secure` | `planing front end` §5.6 |
| Librería UI | `@mitsj/ui`, con Tailwind 4 y componentes React y Astro | Plan de la librería v0.1 |
| Hikvision | HikCentral OpenAPI como opción recomendada; ISAPI como alternativa | `sso` §12.1 |

## Arquitectura y Notas Técnicas

Detalle completo en `arquitectura` §2 a §5 y en [[sso-diagramas-wiki]] (diagramas). Resumen:

- **Interfaces del SSO:** Acceso (público: login, 2FA, recuperación, crear contraseña) y Consola (mis datos, administración y dashboard según el rol).
- **Despliegue bajo un solo dominio:** `/` (Acceso estático), `/consola/*` (estático), `/connect/*` (OpenIddict), `/api/*` (API), `/hubs/*` (SignalR). Sin problemas de CORS ni cookies entre dominios.
- **El login no depende del CDN.** Si el CDN cae, el SSO sigue autenticando; solo fallan fotos y documentos.
- **Registro de cuentas:** lo implementan las apps autorizadas con el SDK. El SSO valida identidad, crea la cuenta y muestra la pantalla de crear contraseña.
- **Riesgos** (de `planing front end` §11 y `sso` §17):
  - La Consola crece dentro de una sola isla.
  - Las versiones de Astro cambian entre apps.
  - El Acceso depende de que cargue la isla del formulario.
  - Un solo mantenedor para `@mitsj/ui`.
  - Los valores de RENAPO y del CDN no están definidos.

## Archivos Clave

- `requerimientos/sso.md` (v0.16) y `requerimientos/arquitectura.md` (v0.9): fuentes principales.
- `requerimientos/planing front end.md` (v0.2): decisión de frontend y plan de islas.
- `requerimientos/SISTEMA-DE-DISENO.md`: componentes, tokens y definición de terminado.
- `backlog-sso.md` y `sprints.md`: historias, prioridades y sprints (requieren actualización).
- [[sso-diagramas-wiki]]: diagramas Mermaid de arquitectura y de flujos (copia de `diagramas-sso.md`).
- `~/Documents/Developer/mitsj-ui`: librería de componentes.
- `~/Documents/Developer/SSO`: carpeta de documentos del SSO (sin código).

## FASE 0 · Levantamiento (Sprint 0)

Sin esta fase no se puede cerrar el alcance de Fase 1.

- **Decisiones bloqueantes:** E0-08 (valores por defecto), E0-09 (segunda sesión), E0-10 (proveedor de celular), E0-04 (dominios de Google).
- **Levantamiento:** E0-01 y E0-15 (Hikvision), E0-02 (sistemas actuales y formato de contraseñas), E0-03 (usuarios por tipo y UA), E0-06 y E0-07 (impacto de datos y jurídico).
- **Técnico:** E1-01 (microservicio con API y PostgreSQL), E1-02 (ambientes y CI/CD).
- **Meta de salida:** decisiones tomadas o con responsable y fecha; un servicio "hola mundo" desplegado en pruebas.

## FASE 1 · Núcleo MVP para Servicios Escolares

Sprints según `sprints.md`. Cada sprint dura dos semanas (supuesto por confirmar).

| Sprint | Objetivo | Historias |
|---|---|---|
| S1 · Servidor OIDC e identidad base | Una app de prueba obtiene un token | E1-03, E1-04, E2-01, E2-03, E4-11, E4-04, E4-03, E1-08 |
| S2 · Autenticación completa | Login, recuperación y 2FA | E4-01, E8-07, E4-10, E4-06, E4-07, E2-05, E7-01, E1-05 |
| S3 · Catálogo y permisos | El administrador define apps, roles, grupos y alcances | E5-01 a E5-04, E5-06, E7-02, E7-03, E4-08, E4-09 |
| S4 · API de permisos y primer SDK | Una app consulta ¿puede?, ¿sobre cuáles? y ¿qué campos? | E5-05, E5-07 a E5-10, E6-01, E6-02, E6-04 |
| S5 · Registro: API y validación | Crear cuenta desde una app autorizada, hasta antes de RENAPO | E3-09, E3-10, E3-02, E3-03, E3-13, E6-03, E6-07 |
| S6 · RENAPO, activación y Google | Aspirantes activos; los demás pendientes de activación | E3-11, E3-12, E3-04 a E3-08, E4-02 |
| S7 · Consola, perfiles y CDN | Consola usable y archivos contra el CDN | E8-01 a E8-05, E8-08, E2-02, E2-04, E6-08, E18-01 a E18-04 |
| S8 · API keys y migración | Carga de cuentas existentes y emisión de API keys | E9-01 a E9-03, E4-05, E17-01 a E17-04, E2-06, E2-07, E1-06 |
| S9 · Piloto y cierre | Servicios Escolares opera con el SSO; pruebas de seguridad aprobadas | E6-06, E6-05, E1-07, E16-01 a E16-04, E8-06, E4-12 |

**Criterio de salida de Fase 1:** pruebas de conformidad OpenID, análisis de vulnerabilidades y prueba de penetración externa (E16).

## FASE 2 · Gobierno y control fino

Aproximadamente 3 a 4 sprints (E10, E11, E12, E18-05).

- Permisos por campo, vigencias ligadas a periodos, llaves excepcionales, suplencias, separación de funciones, "ver como".
- Dashboard en tiempo real con SignalR (`@microsoft/signalr` en isla `client:visible`).
- Google Workspace: alta y suspensión automática; organismos con varios usuarios.
- Pendiente antes de planear: criterios de aceptación y E0-11 (quién genera llaves excepcionales).

## FASE 3 · Acceso físico

Aproximadamente 3 a 4 sprints (E13, E14).

- Conector Hikvision (personas, fotos, vigencias, niveles de acceso) y registro de vehículos.
- Bloqueado por: E0-01, E0-12, E0-13, E0-15 y la evaluación de impacto de datos aprobada.

## FASE 4 · Biometría y mejoras

E15: reconocimiento facial si las terminales no lo cubren en Fase 3; doble verificación placa y rostro; nuevas apps del ERP.

## Dependencias y Orden de Ejecución

1. **Fase 0 antes que todo.** Sin decisiones de §17, Fase 1 no tiene alcance cerrado.
2. **Orden técnico de Fase 1:** E1 → E2 → E4 → E5 → E6 → E3 → E7 → E8 → E9 (según `backlog-sso`). El CDN (E17, E18) va al final porque su tecnología no está definida.
3. **Frontend del Acceso (E8-07)** depende de que la librería `@mitsj/ui` esté publicada. Se pueden usar versiones de prueba desde el repo, pero la publicación va antes de la integración en el SSO.
4. **SDK frontend (E6-03)** depende de `@mitsj/ui`.
5. **Cada fase cierra con pruebas de seguridad** (E16) antes de producción.

**Hitos para medir el avance:**
- **H1:** una app de prueba obtiene token (fin de S1).
- **H2:** login completo con 2FA y recuperación (fin de S2).
- **H3:** registro de aspirante con RENAPO (fin de S6).
- **H4:** Servicios Escolares opera con el SSO (fin de S9).
