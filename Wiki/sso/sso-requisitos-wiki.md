---
tipo: wiki
titulo: "Requisitos del SSO MiTSJ"
autor: "Efraín García"
creado: 2026-10-28
actualizado: 2026-10-28
estado: draft
proyecto: sso
tags:
  - sso
  - requisitos
---

# Requisitos del SSO MiTSJ

## Contenido

> Derivados de [[sso-reglas-negocio-wiki]]. Cada requisito cita las reglas que lo originan. Prioridad: **MVP** = para que Servicios Escolares opere; **F2** = gobierno y control fino. Los detalles técnicos (librerías, despliegue) están en [[sso-diseno-base-wiki]].

### 1. Requisitos funcionales

| ID | Requisito | Reglas | Prio. |
|---|---|---|---|
| RF-01 | Autenticar por CURP/RFC o correo + contraseña (Argon2id) y por Google Workspace; emitir tokens OAuth 2.0 / OIDC (JWT). El **acceso emitido depende de la credencial** (CURP+contraseña = Aspirante; correo institucional = Alumno). | RN-REG-02a | MVP |
| RF-02 | Bloquear temporalmente tras 5 intentos fallidos (15 min), contando cuenta e IP; valores configurables. | RN-AUT-01 | MVP |
| RF-03 | Sesiones: 1 por tipo de dispositivo (la nueva cierra la anterior del mismo tipo), configurable por rol. | RN-AUT-02 | MVP |
| RF-04 | Inactividad: global de 4 h por defecto; cada app puede definir la suya y prevalece. | RN-AUT-03 | MVP |
| RF-05 | Recuperación de contraseña por enlace de un solo uso, 30 min; enlace de crear contraseña, 12 h. Cierra sesiones abiertas al cambiar. | RN-AUT-04, 05 | MVP |
| RF-06 | 2FA configurable por cuenta y por rol (obligatorio para root y administradores). | diseño base §8.4 | MVP |
| RF-07 | API de registro para apps autorizadas: validar CAPTCHA/IP, CURP (formato, verificador, coherencia, duplicado, RENAPO vía proveedor tercero, **una sola vez**), correo y celular; crear la cuenta. | RN-IDE-01..06, RN-AUT-06 | MVP |
| RF-08 | Activación: Aspirante automática; Docente, Administrativo, Organismo y Proveedor pendientes; el administrador aprueba/rechaza con motivo en **5 días**, o pasa a Rechazada por vencimiento. Para Docente/Administrativo valida nómina RH, UA/área y rol inicial. | RN-REG-01, 03, 04 | MVP |
| RF-09 | Varios perfiles por persona, sumados por un administrador o por reporte de la app responsable; un cambio de perfil retira permisos del anterior y asigna los del nuevo. | RN-REG-05, 06 | MVP |
| RF-10 | Organismos y proveedores: entidad con RFC; el creador es ADMON y da de alta a su personal sin aprobación; cambio de ADMON lo autoriza un admin con evidencia. | RN-ORG-01..04 | F2 |
| RF-11 | Catálogo de aplicaciones: registrar apps y su matriz (módulo/formulario/acción); roles, grupos, alcance Global → UA → Área/Carrera con herencia hacia abajo. | RN-PER-01, 02 | MVP |
| RF-12 | API de permisos: la app pregunta "¿qué permisos tiene X?" y el SSO responde con la matriz **de esa app**. Las relaciones persona↔registro las resuelve cada app. | RN-PER-02, 06 | MVP |
| RF-13 | Administración por **CLI y Consola** (interfaz). | RN-PER-05 | MVP |
| RF-14 | Vigencia (inicio/fin) de permisos y suplencias (titular o superior, máx. 7 días, registradas "como suplente de"). | RN-PER-03 | F2 |
| RF-15 | Permisos por campo (visible / solo lectura / oculto). | RN-PER-05 | F2 |
| RF-16 | Motor de API keys: una key por app backend-to-backend; el destino (p. ej. CDN) la valida contra el SSO. Emisión, hash, vigencia, rotación, revocación y bitácora de uso. | RN-KEY-01..04 | MVP |
| RF-17 | Ciclo de vida: cuentas nunca eliminadas; inactivar/reactivar con motivo; efecto en apps < 1 min; suspender/reactivar la cuenta de Google; notificar a Accesos. | RN-CIC-01..03, RN-GWS-03 | MVP |
| RF-18 | Alta de correo institucional en Google Workspace para Alumno (al inscribirse), Docente y Administrativo. | RN-GWS-01, 02 | F2 |
| RF-19 | Corrección de CURP/RFC (revalida con RENAPO) y fusión de cuentas: solo admin global con motivo. | RN-CIC-04 | MVP |
| RF-20 | Bitácora de seguridad inalterable, consultable por administrador y por un rol de auditoría (solo lectura); dashboard de sesiones. | RN-BIT-01 | MVP (bitácora) / F2 (dashboard) |
| RF-21 | Migración inicial de cuentas existentes al crear el sistema. | RN-MIG-01 | MVP |

### 2. Requisitos no funcionales

| ID | Requisito | Reglas |
|---|---|---|
| RNF-01 | Dimensionar para ~1,000 usuarios concurrentes; el SSO es puerta de identidad (login y redirección), sin tráfico sostenido. | RN-DIM-01 |
| RNF-02 | Alta disponibilidad (al menos dos instancias, BD replicada, respaldos). | diseño base §15 |
| RNF-03 | HTTPS, secretos fuera del código, criptografía con librerías reconocidas; pruebas de conformidad OpenID y pentest antes de producción. | diseño base §15 |
| RNF-04 | Parámetros configurables (intentos, bloqueo, inactividad, caducidad de enlaces, sesiones por rol, 2FA por rol). | RN-AUT-01..05 |
| RNF-05 | Conservación de la bitácora 5 años (provisional, validar con jurídico). | RN-BIT-02 |
| RNF-06 | El login no depende del CDN ni de servicios externos no críticos (RENAPO caído → cuenta pendiente, no se pierde el registro). | RN-IDE-06 |

## Relacionado

- [[sso-reglas-negocio-wiki]]
- [[sso-alcance-wiki]]
- [[sso-decisiones-wiki]]
