---
tipo: metodologia
titulo: "Metodología Second Brain"
autor: "Efraín García"
creado: 2026-09-23
actualizado: 2026-09-23
estado: vigente
tags:
  - metodologia
  - second-brain
  - infraestructura-del-conocimiento
---

# Metodología Second Brain

> Segundo cerebro (Second Brain / *Building a Second Brain*, Tiago Forte). Sistema para capturar, organizar y recuperar conocimiento desde esta bóveda de Obsidian de forma consistente.

## Qué es

Un segundo cerebro almacena conocimiento en notas enlazadas para no depender de la memoria. La bóveda actúa como repositorio de documentación técnica, manuales y planes de trabajo del equipo.

## Por qué / Cuándo usarla

- Cada proyecto (nummex, tsj-web, rh-cotla, tecmm, pase-directo…) tiene planes, sub-planes y manuales propios.
- Necesitamos que un agente (AGENTS.md) o un compañero encuentre rápido la fuente de verdad de cada proyecto.
- El esquema por **clases** (manual, plan, sub-plan, wiki, proyecto, metodología) hace predecible dónde va cada nota.

## Flujo

1. **Capturar** — las notas se crean en la carpeta `plantillas/`-ready: se parte de una plantilla por clase.
2. **Organizar** — cada nota vive en la carpeta de su clase (`Manuales/`, `Planes/`, `Sub-planes/`, `Wiki/`, `Proyectos/`, `Metodología/`).
3. **Revisar** — las notas `Proyectos/proyectos.md` y `Proyectos/Proyecto.md` actúan como índices de estatus y pendientes; se mantienen al día.
4. **Usar** — los enlaces `[[...]]` conectan planes con sus sub-planes (`padre: "[[nummex-app-plan]]"`) y con manuales relacionados.

## Estructura recomendada en esta bóveda

| Carpeta | Contenido |
|---------|-----------|
| `Manuales/` | Cómo operar o usar algo (API, deploys, servicios). |
| `Planes/` | Plan de trabajo de un proyecto completo. |
| `Sub-planes/` | Piezas de un plan padre (enlazadas con `padre:`). |
| `Wiki/` | Referencia, catálogos y conocimiento reutilizable. |
| `Proyectos/` | Estatus, pendientes y acuerdos del equipo. |
| `Metodología/` | Técnicas y convenciones de esta bóveda. |
| `plantillas/` | Templates por clase + AGENTS.md de uso de plantillas. |

## Cómo crear una nota nueva

1. Copiar la plantilla de su clase desde `plantillas/`.
2. Guardarla en la carpeta de su clase con nombre `kebab-case` (`<proyecto>-<tema>-<clase>.md`).
3. Completar el frontmatter (tipo, titulo, autor, creado/actualizado, estado, proyecto, padre si aplica).
4. Llenar el cuerpo siguiendo las secciones de la plantilla y enlazar lo relacionado.

## Relacionado

- [[AGENTS.md]] — norma de trabajo para notas y código.
- [[plantillas/AGENTS.md]] — cómo se usan las plantillas.