# AGENTS — `plantillas/`

Este archivo norma cómo se **usan las plantillas** de esta carpeta. Las reglas globales de la bóveda viven en `/AGENTS.md`; aquí solo lo específico de las plantillas.

## Regla de oro

- **No escribir nunca contenido en `plantillas/`**: es una carpeta de solo plantillas.
- Las plantillas se **copian** a la carpeta de su clase y ahí se llenan. Nunca se edita la plantilla para un caso puntual.

## Plantillas disponibles (una por clase)

| Plantilla | Clase (`tipo`) | Destino |
|-----------|----------------|---------|
| `Template-Proyecto.md` | `proyecto` | `Proyectos/` |
| `Template-Plan-de-trabajo.md` | `plan` | `Planes/` |
| `Template-Sub-plan.md` | `sub-plan` | `Sub-planes/` |
| `Template-Manual.md` | `manual` | `Manuales/` |
| `Template-Wiki.md` | `wiki` | `Wiki/` |
| `Template-Metodologia.md` | `metodologia` | `Metodología/` |

## Cómo se usa una plantilla

1. Copiar el archivo `Template-<Clase>.md` a la carpeta de destino con el nombre real de la nota: `kebab-case`, patrón `<proyecto>-<tema>-<clase>.md` (p. ej. `nummex-app-plan.md`, `backend-manual.md`).
2. Completar el `<frontmatter>`:
   - `titulo`: el `{{title}}` se reemplaza por el H1 de la nota.
   - `creado` / `actualizado`: fecha `YYYY-MM-DD`.
   - `estado`: usar uno de `vigente | en-progreso | pendiente | draft | completado`.
   - `proyecto`: slug del proyecto (nummex, tsj-web, rh-cotla, tecmm, pase-directo…).
   - `padre`: **obligatorio en `sub-plan`** → `"[[<plan-padre>]]"`.
   - `tags`: agrupados por contexto en `kebab-case`.
3. Llenar el cuerpo respetando las secciones de la plantilla; borrar los comentarios `<...>`.
4. Cuando se quiera una nueva plantilla o clase, **proponerla aquí primero** (no improvisar una nota fuera de esquema).

## Tipos de nota (vocabulario cerrado)

`proyecto` · `plan` · `sub-plan` · `manual` · `wiki` · `metodologia`

- `sub-plan` **requiere** la clave `padre`.
- Los planes completos van a `Planes/`; sus piezas, a `Sub-planes/` apuntando al padre con `padre:`.

## Verificación

Antes de continuar, comprobar que:
- [ ] La nota copiada tiene su frontmatter completo (sin `{{...}}` sin resolver).
- [ ] El `tipo` coincide con la clase de la plantilla usada.
- [ ] Si es `sub-plan`, `padre` apunta a un plan existente.
- [ ] La nota quedó en la carpeta de su clase, no aquí.