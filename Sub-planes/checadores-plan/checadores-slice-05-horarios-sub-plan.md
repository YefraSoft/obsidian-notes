---
tipo: sub-plan
titulo: Slice 5 — Horarios, tolerancias y calendario
autor: Efraín García
creado: 2026-11-23
actualizado: 2026-11-23
estado: pendiente
proyecto: checadores
padre: "[[checadores-plan]]"
tags:
  - checadores
  - sub-plan
  - horarios
  - slice-5
---

# Slice 5 — Horarios, tolerancias y calendario

> **Plan padre:** [[checadores-plan]] · **Rama:** `feat/slice-05-horarios` · Plane: CHECADORES-6 (tareas 42–46)
> Cuándo debe checar cada persona y con qué tolerancias; calendario institucional.

## Pendientes

- [ ] Entrevista previa: reglas del calendario (días hábiles, bloqueos) y cómo se importan los horarios existentes.
- [ ] Interpretar los campos de tolerancia del legacy sin significado claro: `RESTRINGIR`, `FORZARSALIDAPORDURACION`, `TIEMPOAPLICAFORZARSALIDA`, `TIEMPOBONIFICACIONFORZARSALIDA`, `REAGENDARSALIDAPORDURACION`.
- [ ] Contrato de la API de horarios que consume el sistema externo.
- [ ] Forma de aviso cuando un horario está por vencer.

## Estado Actual

| Área | Estado |
|------|--------|
| Requisitos | 🟡 Parciales |
| Implementación | ⏳ |

## Legacy → nuevo

| Legacy | Nuevo |
|--------|-------|
| `agregarHorario` (230–235) y tolerancias en centésimas + 1 | Horarios por bloque con vigencia; tolerancias en minutos |
| Horario sin vencimiento claro | Vigencia (p. ej. semestre) con aviso y reasignación |
| Tolerancia única con campos ambiguos | Grupos de tolerancia por bloque y ventanas anterior/posterior por tipo |
| Festivos en tablas de Inspector RT | Calendario institucional editable con bloqueo de periodos |

## Decisiones

1. **Fijos** (administrativos, p. ej. 9:00–17:00) y **dinámicos con historial** (docentes) con vigencia; relación 1–N: varios bloques por persona, cada uno con sus reglas.
2. Al vencer sin asignación se **exige asignar**; las checadas quedan con aviso y se revalidan al asignar; se permite asignar **antes** de que venza.
3. **Grupos de tolerancia por bloque**; para cada tipo (entrada, salida) ventana con tolerancia **anterior y posterior**; fuera de rango se descarta o genera incidencia (configurable).
4. Límite de llegada muy tarde configurable por grupo; redondeo configurable (1 min por defecto) solo antes de la hora de salida.
5. Calendario institucional editable; periodos **bloqueables** para solicitar permisos.
6. Toda la configuración se sirve desde la caché en memoria (slice 0).

## Alcance

Cubre horarios, bloques, vigencias, tolerancias y calendario con sus pantallas y API. No evalúa checadas (slice 6).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[checadores-plan]]. Reglas de ventana como funciones puras en `Core`. Ciclo CREAR → MODIFICAR → ENLAZAR.

## Implementación

### Paso 1 — Base de datos
- [ ] Tablas (propuesta) `horarios`, `bloques_horario` (día, entrada, salida, grupo de tolerancia), `grupos_tolerancia`, `calendario_dias`, `periodos_bloqueados`.

### Paso 2 — API
- [ ] CREAR: servicios y queries; `VentanaChecada` pura (anterior/posterior, límite tarde, redondeo).
- [ ] MODIFICAR: integrar con empleados y caché de configuración.
- [ ] ENLAZAR: `/horarios`, `/empleados/{id}/horarios`, `/grupos-tolerancia`, `/calendario`, `/periodos-bloqueados`; API compatible con el sistema externo.

### Paso 3 — Front
- [ ] Pantallas de horarios (asignación anticipada), tolerancias y calendario; alerta de vencimiento próximo.

## Criterios de Aceptación

- [ ] Un docente con dos turnos separados tiene ambos bloques evaluables.
- [ ] Un horario vencido sin reasignar produce aviso y no pierde checadas.
- [ ] Cambiar una tolerancia en el panel se refleja sin reiniciar.
- [ ] Un periodo bloqueado rechaza solicitudes (slice 7).

## Relacionado

- [[checadores-plan]]
- [[checadores-slice-03-migracion-sub-plan]]
- [[checadores-slice-06-incidencias-sub-plan]]
