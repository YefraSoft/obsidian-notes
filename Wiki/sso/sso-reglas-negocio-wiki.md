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

### 5. Acceso físico y fotografía

| ID | Regla | Estado |
|---|---|---|
| RN-FIS-01 | La fotografía es obligatoria para **Alumno, Docente y Administrativo**. | cerrada |
| RN-FIS-02 | Integración con un **HikCentral único central** (OpenAPI) para las 13 UA; el conector sincroniza y recibe eventos. | cerrada |
| RN-FIS-03 | Medio de identificación (rostro, tarjeta, QR), modelos de terminal y de cámaras de placas. | abierta (levantamiento Fase 0) |
| RN-FIS-04 | Vehículos: máximo por rol, roles con acceso vehicular, tarjeta de circulación. | abierta |

### 6. Dimensionamiento

| ID | Regla | Estado |
|---|---|---|
| RN-DIM-01 | Meta ~1,000 usuarios concurrentes. El SSO actúa como puerta (login y distribución a las apps), sin tráfico sostenido. | cerrada |

### 7. Diferidas

| ID | Regla | Estado |
|---|---|---|
| RN-DIF-01 | Llaves de autorización excepcional de un solo uso (§6.8): se difieren; quedan como idea sin reglas detalladas hasta que una app las pida. | diferida |

### 8. Áreas pendientes de ronda

Registro y activación de cuentas · perfiles y cambios de perfil · permisos (alcance, registros asignados, campos, vigencias, suplencias, separación de funciones) · organismos · ciclo de vida y reactivación · vehículos · privacidad y plazos de conservación · carga masiva y migración SHA-1 · bitácora y "ver como".

## Relacionado

- [[sso-decisiones-wiki]]
- [[sso-requisitos-wiki]]
- [[sso-alcance-wiki]]
- [[sso-diseno-base-wiki]]
