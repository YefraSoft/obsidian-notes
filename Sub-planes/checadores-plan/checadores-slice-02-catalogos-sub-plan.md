---
tipo: sub-plan
titulo: Slice 2 — Catálogos y organización
autor: Efraín García
creado: 2026-11-23
actualizado: 2026-11-23
estado: pendiente
proyecto: checadores
padre: "[[checadores-plan]]"
tags:
  - checadores
  - sub-plan
  - catalogos
  - slice-2
---

# Slice 2 — Catálogos y organización

> **Plan padre:** [[checadores-plan]] · **Rama:** `feat/slice-02-catalogos` · Plane: CHECADORES-3 (tareas 25–28)
> Datos maestros que referencian los demás slices: departamentos, carreras, empleados, puntos de registro y dispositivos. Los datos iniciales llegan de CORE legacy en el slice 3.

## Pendientes

- [ ] Entrevista previa: campos del empleado a conservar de `empleados_dat_asis`; estructura carreras vs departamentos.
- [ ] Definir si los puntos de registro se sincronizan desde HIK o se capturan a mano.
- [ ] Reglas de baja de empleado (qué pasa con sus horarios e incidencias).

## Estado Actual

| Área | Estado |
|------|--------|
| Requisitos | 🟡 Parciales |
| Implementación | ⏳ |

## Legacy → nuevo

| Legacy | Nuevo |
|--------|-------|
| `empleados_dat_asis`, `puntos_registro_xgrpo`, `AdministracionIDs` | `empleados`, `puntos_registro`, `dispositivos` con claves únicas |
| `personCode` de HIK buscado por ID o tarjeta; sin match → `codEmpleado` nulo | `codigo_hik` único; sin match → bandeja de no identificadas (slice 4) |
| Personal administrativo fuera del CORE legacy | Alta desde el panel, mismo trato que el resto |

## Decisiones

1. El `personCode` de HIK es el **código HIK** del empleado; único.
2. Tipos de personal: **docente** y **administrativo**; condicionan horarios, jerarquía y paquete de prestaciones.
3. Los administrativos se dan de alta **desde el panel**.
4. Puntos de registro y dispositivos con los datos de HIK por ahora; se prevé cambio al SDK propio.
5. Los datos iniciales se cargan en el slice 3; este slice solo define el modelo y la gestión.

## Alcance

Cubre catálogos y sus pantallas. No cubre horarios (slice 5) ni la carga masiva (slice 3).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[checadores-plan]]: un servicio por agregado (`DepartamentoService`, `EmpleadoService`, `PuntoRegistroService`…), lecturas por `IXQueries`. Ciclo CREAR → MODIFICAR → ENLAZAR.

## Implementación

### Paso 1 — Base de datos
- [ ] Tablas (propuesta) `departamentos`, `carreras`, `empleados`, `puntos_registro`, `dispositivos`; únicos en `codigo_hik` y número de empleado.

### Paso 2 — API
- [ ] CREAR: contratos con validación, servicios, queries y repositorios por agregado.
- [ ] MODIFICAR: relación con usuarios (slice 1) y con responsables.
- [ ] ENLAZAR: `GET/POST/PUT /departamentos`, `/carreras`, `/empleados`, `/puntos-registro`, `/dispositivos` con paginación `{ items, page, pageSize, total }`.

### Paso 3 — Front
- [ ] Listados con búsqueda, alta y edición de cada catálogo; filtro por tipo de personal.

## Criterios de Aceptación

- [ ] El código HIK no se repite entre empleados (409).
- [ ] Un administrativo dado de alta desde el panel queda identificable en las checadas.
- [ ] Listados paginados con el contrato estándar.

## Relacionado

- [[checadores-plan]]
- [[checadores-slice-03-migracion-sub-plan]]
- [[checadores-slice-04-captura-sub-plan]]
