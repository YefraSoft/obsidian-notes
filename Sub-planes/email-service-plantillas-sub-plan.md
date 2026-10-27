---
tipo: sub-plan
titulo: Slice 3 — Plantillas
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: pendiente
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - plantillas
  - slice-3
---

# Slice 3 — Plantillas

> **Plan padre:** [[email-service-plan]]
> Gestionar las plantillas de correo y decidir qué consumidores pueden usar cada una.

## Pendientes

- [ ] Definir el motor de render y el formato de `variables`.
- [ ] CRUD de plantillas y activar/desactivar.
- [ ] Asignar plantillas a consumidores.
- [ ] Vista previa.
- [ ] Pantalla del panel con editor.

## Estado Actual

| Área | Estado |
|------|--------|
| Tablas `templates` y `consumer_templates` en `db.sql` | ✅ |
| Rol `TemplateManager`, política y `TemplatePermissions` (`CanManage`/`RequireManage`) | ✅ |
| `TemplatePermissions.RequireUseForGroupAsync` (restricción del envío a grupos) | ✅ |
| Endpoints, contratos y builders | ⏳ |
| Panel (editor `@maily-to/core` ya instalado) | ⏳ |

## Alcance

Cubre plantillas y su asignación a consumidores. No cubre el envío (Slices 7 y 8), que solo las consume. El envío a grupos exige la plantilla en `consumer_templates`; el envío 1 a 1 usa cualquier plantilla activa.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: `Application/Templates` con `ITemplateService`, `ITemplateQueries` e `ITemplateRepository`. La **lista de plantillas va por GraphQL**; las escrituras y el GET por id, por REST.

## Implementación

### Paso 1 — Base de datos
- [ ] Revisar `templates`: `name` único, `subject`, `body_html`, `variables` (JSONB), `transactional`, `active`.
- [ ] Decidir si se versionan las plantillas (hoy el nombre es único).

### Paso 2 — API
- [ ] `TemplateController` y tipo GraphQL en `Api`; `TemplateService`, `TemplateQueries`, contratos y builder en `Application/Templates`; repositorio en `Infrastructure`; política `TemplateManager`.
- [ ] CRUD con validación nativa (nombre, asunto, esquema de `variables`).
- [ ] Activar/desactivar y vista previa con variables de ejemplo.
- [ ] Asignación `consumer_templates` (solo Admin) con listado por consumidor.

### Paso 3 — Panel
- [ ] Lista de plantillas, editor y vista previa.
- [ ] Asignación de plantillas desde la ficha del consumidor.

### Paso 4 — Documentación
- [ ] README: endpoints y reglas de uso de plantillas.

## Criterios de Aceptación

- [ ] Un `TemplateManager` crea, edita y desactiva plantillas; un consumidor sin ese rol recibe 403 con `code: RoleNotAllowed`.
- [ ] Solo el Admin asigna plantillas a consumidores.
- [ ] Una plantilla inactiva no se ofrece para enviar.
- [ ] La vista previa rellena las variables y avisa de las que faltan.
- [ ] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- ¿Qué motor de render se usa y cuál es el formato exacto de `variables` (clave, tipo, obligatoria)?
- ¿Se necesita versionado o historial de cambios de una plantilla?

## Relacionado

- [[email-service-plan]]
- [[email-service-consumidores-keys-sub-plan]]
- [[email-service-envio-individual-sub-plan]]
