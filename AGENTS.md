# AGENTS — Raíz de la bóveda Obsidian

> Norma de trabajo para **esta bóveda de documentación** y para los **proyectos de código** que la acompañan. Léelo completo antes de crear, mover o editar cualquier nota.

## Ubicación y contexto

- Bóveda: `/Users/efraintics/Documents/Obsidian Vault/` (repo git, bajo `Documents/Obsidian Vault`).
- Las notas se organizan **por clase** en carpetas (ver esquema abajo).
- Las plantillas están en `plantillas/` y tienen su propio `plantillas/AGENTS.md`.
- Los proyectos de código viven fuera de la bóveda (p. ej. `/Users/efraintics/proyects/pase-directo/`) y sus entregables se documentan aquí.

---

## 1. Esquema de carpetas (separación por clase)

| Carpeta | Clase (`tipo`) | Contenido |
|---------|----------------|-----------|
| `Manuales/` | `manual` | Cómo operar o usar algo: API, deploys, servicios, scripts. |
| `Planes/` | `plan` | Plan de trabajo / plan de desarrollo de un proyecto completo. |
| `Sub-planes/` | `sub-plan` | Pieza o entregable de un plan padre (obligatorio `padre:`). |
| `Wiki/` | `wiki` | Referencia, catálogos y conocimiento reutilizable. |
| `Proyectos/` | `proyecto` | Estatus, pendientes y acuerdos del equipo. |
| `Metodología/` | `metodologia` | Técnicas de productividad y segunda mente (Second Brain, PARA…). |
| `plantillas/` | — | Solo plantillas + `AGENTS.md`. Nunca contenido final. |

Regla: cada nota vive en la carpeta de su clase. `plantillas/` nunca guarda notas de trabajo.

## 2. Nomenclatura de archivos

- `kebab-case`, minúsculas, sin espacios ni tildes en el nombre (la bóveda usa UTF-8 para el contenido, pero el nombre de archivo simple).
- Patrón: `<proyecto>-<tema>-<clase>.md`
  - `nummex-app-plan.md`, `backend-manual.md`, `to-production-manual.md`, `rh-cotla-plan.md`.
- Los índices/estatus se permiten sin prefijo de tema: `Proyecto.md`, `proyectos.md`.

## 3. Frontmatter estándar (todas las notas)

```yaml
---
tipo: manual            # proyecto | plan | sub-plan | manual | wiki | metodologia
titulo: "Nombre de la nota"
autor: "Efraín García"
creado: YYYY-MM-DD
actualizado: YYYY-MM-DD
estado: vigente         # vigente | en-progreso | pendiente | draft | completado
proyecto: ""            # nummex | tsj-web | rh-cotla | tecmm | pase-directo | ...
padre: "[[<plan-padre>]]"  # SOLO en sub-plan (obligatorio)
tags:
  - <kebab-case>
---
```

Reglas del frontmatter:
- `tipo` es el vocabulario cerrado de clases: `proyecto`, `plan`, `sub-plan`, `manual`, `wiki`, `metodologia`.
- `estado` usa solo esos valores estándar; si necesitas matices, ponlos en el cuerpo.
- `padre` **solo** y **siempre** en `sub-plan`, con wiki-link al plan padre.
- `autor` por defecto "Efraín García"; otros autores si se indica.
- `creado` = fecha de alta, `actualizado` = fecha de última edición; `YYYY-MM-DD`.

## 4. Estructura del cuerpo por clase

- **manual** — `Resumen` · `Requisitos previos` · `Uso / Pasos` · `Troubleshooting` · `Relacionado`.
- **plan** — `Pendientes` · `Estado Actual (resumen ejecutivo)` · `Decisiones de Diseño` · `Arquitectura y Notas Técnicas` · `Archivos Clave` · `FASE 1..N` · `Dependencias y Orden de Ejecución`.
- **sub-plan** — igual que plan, más cabecera `> Plan padre: [[...]]` y `Alcance`.
- **wiki** — `Contenido` numerado por secciones y `Relacionado`.
- **proyecto** — `Estatus` (tabla), `Pendientes`, `Acuerdos`, `Equipo`, `Relacionado`.
- **metodologia** — `Qué es`, `Por qué/Cuándo usarla`, `Flujo`, `Estructura en esta bóveda`, `Relacionado`.

## 5. Enlaces `[[...]]`

- Usar wiki-links `[[Nombre-de-nota]]` para conectar relacionadas (no rutas completas).
- Los sub-planes SIEMPRE enlazan a su padre con `padre: "[[...]]"` y en la cabecera.
- Se prefiere un solo enlace relevante claro a múltiples enlaces sueltos.

## 6. Crear / actualizar notas

1. **Crear**: copiar la plantilla de la clase desde `plantillas/Template-<Clase>.md` → carpeta de su clase → llenar frontmatter y cuerpo → enlazar relacionadas.
2. **Actualizar**: respetar el formato de la sección correspondiente y subir `actualizado` a la fecha del día.
3. **Mover**: si una nota cambia de clase, moverla de carpeta con `git mv` (preserva historial) y ajustar frontmatter/tipo.
4. **Nunca** inventar URLs, secretos o datos que no estén en el contenido.

## 7. Proyectos de código (además de la bóveda)

Convenciones que aplican al trabajar sobre código fuente del equipo:

- **Gestión de tareas**: en **Plane**, workspace `sitemas-tecmm` (https://testing.tecmm.mx), proyecto `SITEM` para documentación del código; seguir el identificador `SITEM-<n>`.
- **Documentación por proyecto**: cada repo mantiene sus pendientes en Markdown dentro del repo, p. ej. `tareas-bakcned.md` (backend) y `tareas-fontend.md` (frontend) en `pase-directo/`. Esa lista es la fuente de verdad del sprint a corto plazo; Plane lo es de la planificación.
- **Modelos de solución**: separar frontend y backend; versionar en git; los despliegues y flujos de subida se documentan como `manual` en la bóveda (`to-production-manual.md`, `backend-manual.md`).
- **Nuevo proyecto**: crear su `<proyecto>-<tema>-plan.md` en `Planes/`, documentar manuales en `Manuales/`, y publicar su estado en `Proyectos/Proyecto.md`.
- **Estándares de código**: preferir documentar decisiones en las secciones `Decisiones de Diseño` / `Arquitectura y Notas Técnicas` del plan, no en comentarios dispersos.

## 8. Archivos AGENTS en cascada

- **`/AGENTS.md`** (este): reglas globales de la bóveda + código.
- **`plantillas/AGENTS.md`**: cómo usar y verificar plantillas.
- Los repos de código pueden tener su propio `AGENTS.md`/`opencode.json` con MCP y comandos (ver `pase-directo/opencode.json`); no se rescriben aquí.

## Verificación final

Antes de terminar una tarea sobre la bóveda:
- [ ] Cada nota tocada tiene `tipo`, `titulo`, `autor`, `creado`, `actualizado`, `estado` y, si aplica, `proyecto` / `padre`.
- [ ] `estado` y `tipo` son del vocabulario cerrado.
- [ ] La nota está en la carpeta de su clase.
- [ ] Sub-planes con `padre:` correcto.
- [ ] Enlaces `[[...]]` no rotos (por nombre).
- [ ] Git: cambios versionados (el vault es repo).