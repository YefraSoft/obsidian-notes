---
tipo: sub-plan
titulo: "Asistencias — Slice 7: Consulta, evidencia y reportes RH"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-18
estado: pendiente
proyecto: asistencias
padre: "[[asistencias-plan]]"
tags:
  - plan
  - sub-plan
  - asistencias
  - slice
  - reportes
---

# Asistencias — Slice 7: Consulta, evidencia y reportes RH

> **Plan padre:** [[asistencias-plan]] · **Rama:** `feat/slice-7-consulta-reportes`
> Consulta de checadas e intentos por rol y alcance, lectura de evidencia y reportes en Excel para RH y nómina.

## Pendientes

- [ ] Detallar el slice con el responsable y crear sus sub-tareas en Plane.
- [ ] Definir las columnas exactas de cada reporte con RH/nómina y el calendario de quincenas.
- [ ] Confirmar si `RH_ENLACE` ve evidencia y puede rechazar checadas de su UA.
- [ ] Definir el rechazo por inconsistencia de `RH_DIR` (motivo, auditoría y efecto en incidencias).

## Estado Actual

| Área | Estado |
|------|--------|
| Requisitos | ✅ Definidos |
| Implementación | ⏳ Pendiente |

## Alcance (vertical)

- **BD:** vistas o consultas sobre checadas, intentos e incidencias; marca y motivo de rechazo por inconsistencia con auditoría.
- **API:** consultas paginadas con filtros (UA, área, periodo, persona) con **alcance obligatorio por rol** (`RH_ENLACE` su UA; `RH_DIR` y `ADMIN` todas); exportación `.xlsx` en el servidor; rechazo por inconsistencia para `RH_DIR`.
- **Pantalla:** consulta de checadas, intentos e incidencias; visor de evidencia a través del **proxy de lectura en el servidor de Astro** (agrega `X-Api-Key`, `X-User-Key` y `X-Private-Route-Key`, que nunca salen del servidor); descarga de reportes.
- **Reportes (solo datos, nunca imágenes):** checadas por persona y periodo, incidencias por quincena o periodo, e intentos rechazados con su motivo, con filtros por UA y área.
- **Docs:** nota de testing.

## Implementación

### Bloque CREAR
- [ ] Consultas con alcance; generador de Excel; endpoint de evidencia y proxy de lectura.

### Bloque MODIFICAR
- [ ] Ajustes a checadas e incidencias para el rechazo por inconsistencia.

### Bloque ENLAZAR
- [ ] Actions, pantallas por rol y verificación de alcance con usuarios de prueba de cada nivel.

## Entrega

1. `feat/slice-7-consulta-reportes` desde `QA` → PR a `QA` → **testing**.
2. `Testing/asistencias-slice-7-testing`; ajustes `fix/*`.
3. PR a `staging` → PR a `main`.

## Criterios de Aceptación

- [ ] El reporte de checadas no incluye imágenes ni URLs de evidencia.
- [ ] `RH_ENLACE` solo obtiene datos de su UA en consultas, evidencia y reportes.
- [ ] La `private_route_key` y las demás keys no salen del servidor de Astro.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-roles-permisos-wiki]]
- [[asistencias-arquitectura-wiki]]
- [[asistencias-slice-4-checada-valida-sub-plan]]
- [[asistencias-slice-5-horarios-incidencias-sub-plan]]
