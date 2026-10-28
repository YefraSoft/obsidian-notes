---
tipo: wiki
titulo: "Registro de decisiones del SSO MiTSJ"
autor: "Efraín García"
creado: 2026-10-28
actualizado: 2026-10-28
estado: en-progreso
proyecto: sso
tags:
  - sso
  - decisiones
---

# Registro de decisiones del SSO MiTSJ

## Contenido

### 1. Decisiones cerradas

| Fecha | Tema | Decisión | Regla |
|---|---|---|---|
| 2026-10-28 | Sesiones | Mismo tipo de dispositivo cierra la anterior; tipos distintos conviven | RN-AUT-02 |
| 2026-10-28 | Bloqueo | 5 intentos / 15 min | RN-AUT-01 |
| 2026-10-28 | Inactividad | Global 4 h; la app puede definir el suyo | RN-AUT-03 |
| 2026-10-28 | Enlaces | Recuperación 30 min; crear contraseña 12 h | RN-AUT-04, 05 |
| 2026-10-28 | RENAPO | Proveedor tercero; validación única persistida | RN-IDE-03, 04 |
| 2026-10-28 | Google Workspace | Alumno, Docente, Administrativo | RN-GWS-01, 02 |
| 2026-10-28 | Llaves excepcionales | Diferidas (idea) | RN-DIF-01 |
| 2026-10-28 | Acceso físico | Sale del SSO a un microservicio de Accesos independiente (Hikvision, vehículos, fotos de acceso, eventos) | RN-FIS-01..03 |
| 2026-10-28 | Foto obligatoria | Alumno, Docente, Administrativo (se traslada al plan de Accesos) | RN-FIS-03 |
| 2026-10-28 | Concurrencia | ~1,000; el SSO es puerta, no tráfico sostenido | RN-DIM-01 |
| 2026-10-28 | API keys | Backend a backend, una key por app, validada contra el SSO | RN-KEY-01..03 |
| 2026-10-28 | Aspirante → Alumno | Proceso largo; no chocan datos; se notifica al aspirante | RN-REG-02 |
| 2026-10-28 | Aspirante/Alumno | Misma identidad, acceso según credencial (CURP+pass = aspirante; correo @tecmm.edu.mx = alumno) | RN-REG-02a |
| 2026-10-28 | Pendientes de activación | 5 días; al vencer pasa a Rechazada por vencimiento | RN-REG-03 |
| 2026-10-28 | Multi-perfil | Lo suma admin o app, no auto-registro | RN-REG-06 |
| 2026-10-28 | Validación de Docente/Administrativo | Nómina RH + UA/área + rol inicial | RN-REG-04 |
| 2026-10-28 | Cambio de perfil | Retira permisos anteriores y asigna los nuevos | RN-REG-05 |
| 2026-10-28 | Alcance | Hereda hacia abajo | RN-PER-01 |
| 2026-10-28 | Suplencias | Titular o superior, máx. 7 días | RN-PER-03 |
| 2026-10-28 | Separación de funciones | Diferida | RN-PER-04 |
| 2026-10-28 | Rol del SSO | Proveedor de identidad / API gateway; entrega la matriz de permisos que cada app pide; administración por CLI y Consola | Objetivo rector |
| 2026-10-28 | Conflicto de permisos | Se resuelve por app; no hay conflicto entre apps | RN-PER-02 |
| 2026-10-28 | Permisos por campo | Fase 2 | RN-PER-05 |
| 2026-10-28 | Registros asignados | Los resuelve cada app; el SSO no guarda relaciones | RN-PER-06 |
| 2026-10-28 | Organismos/Proveedores | Mismo esquema; creador = ADMON; cambio de representante con evidencia | RN-ORG-01..04 |
| 2026-10-28 | Personal de organismo | El ADMON lo da de alta sin aprobación | RN-ORG-03 |
| 2026-10-28 | Inactivación | Sin login, suspende Google, notifica a Accesos (microservicio aparte) | RN-CIC-02 |
| 2026-10-28 | Propagación | < 1 min | RN-CIC-03 |
| 2026-10-28 | Corrección/fusión de identidad | Solo admin global | RN-CIC-04 |
| 2026-10-28 | Bitácora | La consultan admin y un rol encargado (auditoría); conservación 5 años provisional | RN-BIT-01, 02 |
| 2026-10-28 | Migración | Se hace al crear el sistema | RN-MIG-01 |

### 2. Decisiones abiertas

| Tema | Pendiente | Responsable | Fecha |
|---|---|---|---|
| Celular | Canal (WhatsApp/SMS) y proveedor | por asignar | por definir |
| Proveedor RENAPO | Cuál, costo y cupo | por asignar | por definir |
| CDN | Tecnología, público/privado, apps que usan sus keys, calendario de migración | por asignar | por definir |
| Google WS | Dominio(s) | por asignar | por definir |
| Accesos (microservicio aparte) | Hikvision/HikCentral, vehículos, plazos de eventos: se definen en su propio plan | por asignar | por definir |
| Jurídico | Plazos de conservación y aviso de privacidad | por asignar | por definir |
| Dominio del SSO | Dominio y proxy inverso (afecta cookies y CSP) | por asignar | por definir |

## Relacionado

- [[sso-reglas-negocio-wiki]]
- [[sso-diseno-base-wiki]]
