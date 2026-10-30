---
tipo: wiki
titulo: "Reglas de negocio del SSO MiTSJ"
autor: "Efraín García"
creado: 2026-10-28
actualizado: 2026-10-28
estado: en-progreso
proyecto: sso
tags:
  - sso
  - reglas-de-negocio
---

# Reglas de negocio del SSO MiTSJ

## Contenido

> **Objetivo rector:** el SSO es un **proveedor de identidad** (puerta / *API gateway* de identidad) entre los microservicios de MiTSJ. Autentica y entrega a cada aplicación la **matriz de permisos que esa aplicación le solicita**; **no decide** las reglas de negocio de las aplicaciones. Se administra por **dos vías: CLI y Consola (interfaz)**.

> Se depura por rondas de preguntas. Cada regla: `RN-<área>-nn` · enunciado · valor por defecto · quién lo configura · fuente en [[sso-diseno-base-wiki]] · estado (**cerrada** / **abierta** / **diferida**). Las áreas sin reglas aún se listan como pendientes de ronda.

### 1. Autenticación y sesiones

| ID | Regla | Valor por defecto | Configura | Fuente | Estado |
|---|---|---|---|---|---|
| RN-AUT-01 | Tras N intentos fallidos la cuenta se bloquea temporalmente; el conteo considera cuenta **y** IP. | 5 intentos / 15 min | Admin SSO (Consola) | §8.5 | cerrada |
| RN-AUT-02 | Una segunda sesión del **mismo tipo** de dispositivo cierra la anterior y abre la nueva. Tipos distintos (PC + móvil) conviven. | 1 PC + 1 móvil | Admin SSO, por rol | §8.6 | cerrada |
| RN-AUT-03 | Inactividad: el SSO define un valor global; cada aplicación puede definir el suyo y **prevalece**; si la app no define, aplica el global. | Global 4 h | Admin SSO / cada app | §8.7 | cerrada |
| RN-AUT-04 | Enlace de recuperación de contraseña: un solo uso, caduca a los 30 min. | 30 min | Admin SSO | §10 | cerrada |
| RN-AUT-05 | Enlace de confirmar correo / crear contraseña: un solo uso, caduca a las 12 h. | 12 h | Admin SSO | §9.2 | cerrada |
| RN-AUT-06 | Verificación del celular en el registro por WhatsApp y/o SMS. | — | — | §9.2 | abierta (canal y proveedor) |

### 2. Identidad y RENAPO

| ID | Regla | Estado |
|---|---|---|
| RN-IDE-01 | Identificador: CURP; sin CURP, RFC; organismo/persona moral, RFC del organismo (con CURP/RFC del representante). | cerrada (diseño base §4.1) |
| RN-IDE-02 | Orden de validación de CURP: formato → dígito verificador → coherencia → ¿ya existe? → ¿ya validada? → RENAPO. | cerrada (§4.4) |
| RN-IDE-03 | RENAPO se consulta a través de un **proveedor tercero autorizado**. | cerrada |
| RN-IDE-04 | La validación RENAPO se hace **una sola vez** por CURP; el resultado se persiste y no se vuelve a consultar (sustituye la "vigencia configurable" del §4.4 paso 5). | cerrada |
| RN-IDE-05 | El usuario no ve los datos devueltos por RENAPO antes de registrarse; solo "coincide / no coincide". | cerrada (§4.4) |
| RN-IDE-06 | Si el proveedor no responde, la cuenta queda *Pendiente de validación RENAPO* y se reintenta. | cerrada (§4.4) |

### 2.1 Registro, activación y perfiles

| ID | Regla | Estado |
|---|---|---|
| RN-REG-01 | El Aspirante se activa solo al validar correo y celular. | cerrada (§9.2) |
| RN-REG-02 | Aspirante → Alumno no es automático: requiere un proceso largo (validación, examen y creación de la cuenta de estudiante). Los datos de aspirante y alumno no chocan. Al crearse la cuenta de estudiante se **notifica** al aspirante (a su correo o a su panel). | cerrada |
| RN-REG-02a | Una persona = una identidad (mismos datos), pero **el acceso emitido depende de la credencial con que entra**: CURP + contraseña → acceso de **Aspirante**; correo institucional `@tecmm.edu.mx` (Google) → acceso de **Alumno**. Cada acceso lleva solo los permisos de su perfil. *Cambia el §5 del diseño base ("misma cuenta").* Pendiente definir si esto se generaliza a docentes/administrativos con varios perfiles. | cerrada (aplicable a Aspirante/Alumno) |
| RN-REG-03 | Cuentas pendientes de activación (docente, administrativo, organismo, proveedor): plazo de revisión de **5 días**; al vencer pasan a *Rechazada por vencimiento* (no se elimina; el usuario debe registrarse de nuevo; queda historial y se le notifica). | cerrada |
| RN-REG-06 | Una persona puede tener varios perfiles a la vez; se suman por un administrador o por reporte de la app responsable, **no por auto-registro**. | cerrada |
| RN-REG-04 | Antes de activar a Docente/Administrativo, el administrador valida: (a) que exista en nómina/plantilla de RH, (b) Unidad Académica y área correctas, (c) que se asigne un rol inicial. | cerrada |
| RN-REG-05 | Cambio de perfil reportado por una app: se retiran los permisos del perfil anterior y se asignan los del nuevo; se conservan los de otro perfil que la persona siga teniendo activo (p. ej. docente + alumno de posgrado). Se registra qué app lo reportó. | cerrada |

### 2.2 Permisos

| ID | Regla | Estado |
|---|---|---|
| RN-PER-01 | Alcance jerárquico Global → UA → Área/Carrera; el permiso **hereda hacia abajo** (nunca hacia arriba ni lateral). Un rol puede tener varios alcances. | cerrada |
| RN-PER-02 | Los permisos se resuelven **por aplicación**: cuando una app pregunta "¿qué permisos tiene X?", el SSO responde con la matriz de **esa app** (p. ej. "editar horarios"). Lo que otra app defina no afecta a la solicitante. Si una app no otorga un permiso, simplemente no aparece en su matriz. Denegado por omisión. Una persona en dos grupos de la **misma** app: se **suman** salvo denegación individual explícita (que gana). | cerrada (el conflicto entre apps no existe) |
| RN-PER-05 | Los permisos y roles se administran por **CLI y Consola**. Permisos por campo: **Fase 2**. | cerrada |
| RN-PER-06 | Relaciones persona↔registro (docente↔grupo, tutor↔tutorado): **las resuelve cada app** con su propia base. El SSO entrega identidad, matriz y alcance UA/área; **no** guarda relaciones ni expone "API de relaciones" (sustituye §6.5 y §6.11.3 del diseño base). | cerrada |

### 2.3 Organismos y proveedores

| ID | Regla | Estado |
|---|---|---|
| RN-ORG-01 | Proveedor usa el **mismo esquema que Organismo**: una entidad con tipo Empresa / Institución / Proveedor, RFC y usuarios. | cerrada |
| RN-ORG-02 | Quien crea el registro del organismo queda con el **subrol ADMON** del organismo; los demás usuarios son **personal** del organismo, con permisos asignables. | cerrada |
| RN-ORG-03 | El ADMON del organismo da de alta a su personal **sin aprobación** de un administrador del SSO; el SSO valida CURP/RFC y correo de cada alta. | cerrada |
| RN-ORG-04 | Cambio de ADMON/representante: lo autoriza un administrador de UA o global **con evidencia** (documento); se conserva historial y cuentas. | cerrada |

### 2.4 Ciclo de vida e identidad

| ID | Regla | Estado |
|---|---|---|
| RN-CIC-01 | Las cuentas nunca se eliminan: Pendiente → Activa → Inactiva (+ Bloqueada temporal, Rechazada). Historial de estados: quién, cuándo, por qué. | cerrada (§11) |
| RN-CIC-02 | Cuenta inactiva: no inicia sesión; Google Workspace suspendida; el SSO notifica el cambio de estado al microservicio de Accesos (que retira accesos y vehículos; fuera del SSO). Reactivación: la solicita el usuario y la aprueba un administrador (reactiva también Google). | cerrada |
| RN-CIC-03 | Un cambio de permisos, inactivación o cierre de sesión surte efecto en las apps en **menos de 1 minuto**. | cerrada |
| RN-CIC-04 | Corrección de CURP/RFC y fusión de cuentas duplicadas: solo el **administrador global**, con motivo; la CURP corregida se revalida con RENAPO; la fusión conserva el historial de ambas. | cerrada |
| RN-PER-03 | Suplencia: la puede dar el **titular o su superior**, por un máximo de **7 días**; solo permisos que el titular posee; fin automático; acciones registradas "como suplente de". | cerrada |
| RN-PER-04 | Separación de funciones (permisos incompatibles): **diferida**, fuera del MVP. | diferida |

### 3. Google Workspace

| ID | Regla | Estado |
|---|---|---|
| RN-GWS-01 | Reciben correo institucional: **Alumno** (al inscribirse, reportado por Servicios Escolares), **Docente** y **Administrativo**. | cerrada |
| RN-GWS-02 | No reciben: Interesado, Aspirante, Egresado, Organismo, Proveedor. | cerrada |
| RN-GWS-03 | Inactivar la cuenta suspende (no elimina) la de Google; reactivar la reactiva. | cerrada (§11.1) |
| RN-GWS-04 | Dominio(s) de Google Workspace a integrar. | abierta |

### 4. API keys y CDN (backend a backend)

| ID | Regla | Estado |
|---|---|---|
| RN-KEY-01 | Toda app que se comunica backend a backend **sin login** usa una API key emitida por el SSO; cada app tiene la suya. | cerrada |
| RN-KEY-02 | Flujo: la app pide su key al SSO → la presenta al CDN (u otra app destino) → el destino la valida contra el SSO (válida/no válida, dueño, permisos) y procesa o rechaza. | cerrada |
| RN-KEY-03 | El CDN es una **aplicación independiente**; se registra en el SSO como cualquier app (§3.5). | cerrada |
| RN-KEY-04 | Seguridad de la llave (hash, mostrar una vez, vigencia, rotación, revocación, bitácora): se conserva del §3.7. Parámetros (vigencia, límite de uso) por definir. | abierta |
| RN-KEY-05 | Tecnología del CDN y migración de sus API keys actuales. | abierta |

### 5. Acceso físico, fotografía y vehículos — **fuera del SSO**

| ID | Regla | Estado |
|---|---|---|
| RN-FIS-01 | El control de acceso físico (torniquetes, plumas, Hikvision/HikCentral, eventos de paso), el registro de vehículos y la fotografía para acceso son responsabilidad de un **microservicio de Accesos independiente**. El SSO **no** administra accesos ni guarda sus eventos. | cerrada |
| RN-FIS-02 | El microservicio de Accesos es una app más del SSO: se autentica con el SSO y consume su matriz de permisos y la identidad (estado activo/inactivo, vigencia). | cerrada |
| RN-FIS-03 | Las decisiones previas (foto obligatoria para Alumno/Docente/Administrativo; HikCentral único central) se **trasladan** al plan del microservicio de Accesos. Los plazos de conservación de eventos de acceso también. | cerrada (traslado) |

### 6. Dimensionamiento

| ID | Regla | Estado |
|---|---|---|
| RN-DIM-01 | Meta ~1,000 usuarios concurrentes. El SSO actúa como puerta (login y distribución a las apps), sin tráfico sostenido. | cerrada |

### 6.1 Bitácora y migración

| ID | Regla | Estado |
|---|---|---|
| RN-BIT-01 | La bitácora de seguridad del SSO (sesiones, cuentas, permisos, cambios de administración) no se edita ni se borra. La consultan el **administrador** (global todo; de UA, su alcance) y un **rol encargado** (auditoría, solo lectura). | cerrada (alcance fino por confirmar) |
| RN-BIT-02 | Plazo de conservación de la bitácora del SSO: 5 años (propuesta provisional, a validar con jurídico). | provisional |
| RN-MIG-01 | La migración de cuentas existentes se hace **al crear el sistema** (carga inicial). Detalle (formato de hash de origen, qué se importa, correo de crear contraseña) se define en el plan de migración. | cerrada (detalle abierto) |

### 7. Diferidas

| ID | Regla | Estado |
|---|---|---|
| RN-DIF-01 | Llaves de autorización excepcional de un solo uso (§6.8): se difieren; quedan como idea sin reglas detalladas hasta que una app las pida. | diferida |

### 8. Áreas pendientes de ronda

Privacidad y consentimiento (aviso, ARCO) · "ver como" · parámetros de API keys · detalle de migración.

## Relacionado

- [[sso-decisiones-wiki]]
- [[sso-requisitos-wiki]]
- [[sso-alcance-wiki]]
- [[sso-diseno-base-wiki]]
