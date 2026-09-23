---
tipo: plan
titulo: "{{title}}"
autor: ""
creado: {{date}}
actualizado: {{date}}
estado: pendiente         # vigente | en-progreso | pendiente | draft | completado
proyecto: ""
tags:
  - plan
---

# {{title}} — Plan de Desarrollo

> <Resumen ejecutivo: objetivo del proyecto y stack principal (1-3 líneas).>

## Pendientes

- [ ] <Pendiente>

## Estado Actual (resumen ejecutivo)

| Área | Estado |
|------|--------|
| <Área> | <Estado> |

| Fase | Estado |
|------|--------|
| FASE 1 — <Nombre> | ⏳ Pendiente |

## Decisiones de Diseño (vigentes)

- <Decisión y justificación.>

## Arquitectura y Notas Técnicas

- **Stack:** <lenguaje / framework / bd / infra>
- <Nota técnica o links a archivos clave>

## Archivos Clave del Proyecto

| Archivo | Propósito |
|---------|-----------|
| <ruta> | <propósito> |

## FASE 1 — <Nombre>

> **Objetivo:** ...
> **Estado:** ⏳ Pendiente

### 1.1 <Subsección>
- [ ] <Tarea>

### 1.2 Validación
- [ ] <Criterio de aceptación>

## FASE 2 — <Nombre>

...

## Dependencias y Orden de Ejecución

```
FASE 1 — <Nombre>
    │
    └── FASE 2 — <Nombre>
         │
         └── FASE 3 — <Nombre>
```

## Relacionado

- [[<nota relacionada>]]