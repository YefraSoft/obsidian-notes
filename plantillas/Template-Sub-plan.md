---
tipo: sub-plan            # destino: Sub-planes/<plan-padre>/ (carpeta = nombre del plan padre sin .md)
titulo: "{{title}}"
autor: ""
creado: {{date}}
actualizado: {{date}}
estado: pending           # vigente | en-progreso | pendiente | draft | completado
proyecto: ""
padre: "[[<plan-padre>]]"
tags:
  - plan
  - sub-plan
---

# {{title}}

> **Plan padre:** [[<plan-padre>]]
> <Resumen: pieza o entregable específico cubierto por este sub-plan (1-2 líneas).>

## Pendientes

- [ ] <Pendiente>

## Estado Actual

| Área | Estado |
|------|--------|
| <Área> | <Estado> |

## Alcance

<Qué cubre y qué NO cubre este sub-plan.>

## Implementación

### Paso 1 — <Nombre>
- [ ] ...

### Paso 2 — <Nombre>
- [ ] ...

## Criterios de Aceptación

- [ ] <Criterio>

## Relacionado

- [[<plan-padre>]]
- [[<otra nota relacionada>]]