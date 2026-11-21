---
tipo: sub-plan
titulo: "Asistencias — Slice 5: Horarios e incidencias"
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
  - incidencias
---

# Asistencias — Slice 5: Horarios e incidencias

> **Plan padre:** [[asistencias-plan]] · **Rama:** `feat/slice-5-incidencias`
> Horarios por bloque y cálculo de incidencias a partir de las checadas. Reglas: [[asistencias-reglas-de-negocio-wiki]].

## Pendientes

- [ ] Detallar el slice con el responsable y crear sus sub-tareas en Plane.
- [ ] Confirmar si `RH_DIR` administra horarios con un panel mínimo en la v1 o solo se migran del Core (hoy el panel completo es v2).
- [ ] Definir cuándo se ejecuta el cálculo (al checar, tarea programada o ambos) y la zona horaria.
- [ ] Definir turnos que cruzan medianoche y cómo se recalcula si cambia un horario.

## Estado Actual

| Área | Estado |
|------|--------|
| Reglas de incidencias | ✅ Definidas |
| Origen de horarios | Migrados del Core en el registro; administración por `RH_DIR` por confirmar |
| Implementación | ⏳ Pendiente |

## Alcance (vertical)

- **BD:** `horario_bloque` por usuario e `incidencia` (persona, bloque, fecha, tipo, estado) en `new-core`.
- **API:** motor de cálculo por **bloque** (retardo ≥ 15 min, falta ≥ 1 h, falta de entrada o salida, salida anticipada, ausencia del día; un docente puede tener varias faltas el mismo día) y **tarea programada** con `pg_advisory_lock` entre réplicas; la incidencia que requiere aviso escribe una fila en la outbox (slice 6). Administración de horarios por `RH_DIR`/`ADMIN` según se confirme.
- **Pantalla:** consulta y verificación de incidencias por rol y alcance; edición de horarios si aplica.
- **Docs:** nota de testing y ajuste de la wiki de reglas. No cubre días inhábiles, permisos, incapacidades ni comisiones (v2).

## Implementación

### Bloque CREAR
- [ ] Entidades y migración; servicio de horarios; motor de cálculo; tarea programada.

### Bloque MODIFICAR
- [ ] Ajustes para vincular checadas del slice 4 con su bloque.

### Bloque ENLAZAR
- [ ] Endpoints, Actions y pantallas; verificación con datos de prueba.

## Entrega

1. `feat/slice-5-incidencias` desde `QA` → PR a `QA` → **testing**.
2. `Testing/asistencias-slice-5-testing`; ajustes `fix/*`.
3. PR a `staging` → PR a `main`.

## Criterios de Aceptación

- [ ] Un bloque con entrada a 15 min o más genera retardo; a 1 h o más, falta.
- [ ] Un docente con falta matutina y asistencia vespertina conserva ambos resultados.
- [ ] El cálculo es repetible y no duplica incidencias.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-reglas-de-negocio-wiki]]
- [[asistencias-slice-4-checada-valida-sub-plan]]
- [[asistencias-slice-6-notificaciones-sub-plan]]
