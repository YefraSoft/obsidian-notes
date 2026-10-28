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
| 2026-10-28 | Hikvision | HikCentral único central | RN-FIS-02 |
| 2026-10-28 | Foto obligatoria | Alumno, Docente, Administrativo | RN-FIS-01 |
| 2026-10-28 | Concurrencia | ~1,000; el SSO es puerta, no tráfico sostenido | RN-DIM-01 |
| 2026-10-28 | API keys | Backend a backend, una key por app, validada contra el SSO | RN-KEY-01..03 |

### 2. Decisiones abiertas

| Tema | Pendiente | Responsable | Fecha |
|---|---|---|---|
| Celular | Canal (WhatsApp/SMS) y proveedor | por asignar | por definir |
| Proveedor RENAPO | Cuál, costo y cupo | por asignar | por definir |
| CDN | Tecnología, público/privado, apps que usan sus keys, calendario de migración | por asignar | por definir |
| Google WS | Dominio(s) | por asignar | por definir |
| Hikvision | Versión/licencia de HikCentral, modelos de terminal, medio de identificación | por asignar | Fase 0 |
| Vehículos | Máximo por rol, roles con acceso, tarjeta de circulación | por asignar | por definir |
| Jurídico | Plazos de conservación y aviso de privacidad | por asignar | por definir |
| Dominio del SSO | Dominio y proxy inverso (afecta cookies y CSP) | por asignar | por definir |

## Relacionado

- [[sso-reglas-negocio-wiki]]
- [[sso-diseno-base-wiki]]
