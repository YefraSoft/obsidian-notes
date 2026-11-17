---
tipo: sub-plan
titulo: "Asistencias — App web"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-17
estado: pendiente
proyecto: asistencias
padre: "[[asistencias-plan]]"
tags:
  - plan
  - sub-plan
  - asistencias
  - app-web
---

# Asistencias — App web

> **Plan padre:** [[asistencias-plan]]
> Cliente web de checadas (migrará a móvil): captura GPS y foto en vivo y las envía a la API.

## Pendientes

- [ ] Definir el stack del frontend (referencia: Astro + React como en `email-service`).
- [ ] Definir la estrategia de migración a móvil (PWA o app nativa).

## Estado Actual

| Área | Estado |
|------|--------|
| Stack | ⏳ Por definir |
| Pantallas | ⏳ Pendiente |

## Alcance

Pantalla de checada (GPS + cámara en vivo, sin selector de archivos), historial de la persona y administración básica. La validación nunca se decide en el cliente.

## Implementación

### Paso 1 — Captura
- [ ] Permisos de ubicación y cámara; foto tomada con la cámara del dispositivo.

### Paso 2 — Checada
- [ ] Envío a la API y presentación del resultado (válida / rechazada con motivo).

### Paso 3 — Administración
- [ ] Centros de trabajo, personal y consulta de checadas.

## Criterios de Aceptación

- [ ] No hay forma de elegir un archivo de la galería como evidencia.
- [ ] El flujo funciona en navegador móvil.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-checada-valida-sub-plan]]
