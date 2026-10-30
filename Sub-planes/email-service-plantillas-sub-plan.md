---
tipo: sub-plan
titulo: Slice 3 — Plantillas
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-29
estado: en-progreso
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
> Gestionar las plantillas de correo y decidir qué consumidores pueden usar cada una, y para qué tipo de envío.

## Pendientes

- [x] Detallado del plan (fase 1 del ciclo): decisiones tomadas, ver *Decisiones*.
- [x] Esquema: `content_json`, `scope` en `consumer_templates`, restricción `uq_template_name`.
- [x] CRUD de plantillas, activar/desactivar y vista previa (API).
- [x] Asignar plantillas a consumidores con tipo (`MESSAGE` | `GROUP`), solo Admin.
- [x] `TemplatePermissions` a `Application/Permissions` con puerto y `scope`.
- [x] Panel: lista, editor (`@maily-to/core`), variables, secciones prediseñadas, ayuda, vista previa y asignación.
- [ ] Deploy a testing (publicar con `push-harbor.sh` y aplicar el `ALTER` en su base).
- [ ] Fase de dudas y refinamiento (revisión visual en móvil, textos, GraphQL por campo para managers).

## Estado Actual

| Área | Estado |
|------|--------|
| Tablas `templates` (con `content_json`) y `consumer_templates` (con `scope`) | ✅ |
| `TemplatePermissions` (`CanManage`, `RequireManage`, `RequireUseAsync`) en `Application/Permissions` | ✅ (aún nadie llama a `RequireUseAsync`: lo usan los slices de envío) |
| Endpoints REST, GraphQL `templates`, contratos, builders, sanitizador y renderizador | ✅ verificados con curl |
| Panel: `/plantillas`, `/plantillas/nueva`, `/plantillas/[id]` y modal de asignación | ✅ verificado en Chrome |
| Deploy a testing | ⏳ |

## Alcance

Cubre plantillas y su asignación a consumidores. No cubre el envío (Slices 7 y 8), que solo las consume, ni el render en el worker (usará el mismo `ITemplateRenderer` creado aquí).

## Decisiones

1. **Asignación con tipo, también para el 1 a 1.** Para enviar, la plantilla debe estar en `consumer_templates` con el tipo del envío: `MESSAGE` (1 a 1, rol `Sender`) o `GROUP` (a grupos, rol `GroupSender`). Así una plantilla de notificaciones no se usa por error en, por ejemplo, un envío de contraseña. El Admin no necesita asignación. La plantilla debe estar activa. El tipo es un enum en mayúsculas (`TemplateScope`), igual que los demás enums de la base.
2. **Sin versionado.** Se edita en el lugar; el nombre sigue siendo único (`uq_template_name`) y editar cambia también lo que esté en cola.
3. **Variables:** marcadores `{{clave}}` en asunto y cuerpo; lista `[{key, required}]` (sin `type`: todo viaja como texto). Clave `^[a-z][a-zA-Z0-9_]{0,29}$`, única. Un marcador no declarado da 400 (mensaje en `detail`). Sustitución simple con codificación HTML del valor; sin motor de plantillas. El panel normaliza la clave al escribir (minúsculas, sin acentos, espacios a `_`).
4. **Editor y XSS.** `@maily-to/core` (el autor no toca HTML ni CSS). El panel renderiza el JSON del editor con `@maily-to/render` en el servidor de Astro y obtiene un solo HTML con el CSS en línea; el navegador nunca maneja HTML. La API no confía en el HTML recibido: lo pasa por un sanitizador de lista blanca (`HtmlSanitizer`; sin `script`, sin eventos `on*`, URLs solo `http`, `https`, `mailto`). Los valores de variables se codifican al sustituir y la vista previa del panel va en un `iframe sandbox=""` sin scripts.
5. **Reedición.** Se guarda también el JSON del documento (`content_json`); `body_html` es el archivo final que se envía.
6. **Sin `DELETE`:** los mensajes referencian la plantilla; se desactiva.
7. **Vista previa de borrador:** `POST /templates/preview` recibe el contenido (no necesita estar guardado) y valores de ejemplo.
8. **Secciones prediseñadas:** "Encabezado institucional" y "Pie institucional" como bloques HTML de Maily (Insertar o Copiar HTML).
9. **GraphQL** sigue siendo un solo endpoint solo para Admin; cada dominio aporta sus campos con `IQueryMapper` desde su módulo.

## Esquema (ALTER)

```sql
ALTER TABLE templates RENAME CONSTRAINT uq_template_version TO uq_template_name;
ALTER TABLE templates ADD COLUMN content_json JSONB NOT NULL DEFAULT '{}';
ALTER TABLE consumer_templates ADD COLUMN scope VARCHAR(10) NOT NULL DEFAULT 'MESSAGE';
ALTER TABLE consumer_templates DROP CONSTRAINT consumer_templates_pkey;
ALTER TABLE consumer_templates ADD PRIMARY KEY (id_consumer, id_template, scope);
ALTER TABLE consumer_templates ADD CONSTRAINT chk_ctpl_scope CHECK (scope IN ('MESSAGE', 'GROUP'));
```

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: `Application/Templates` con `ITemplateService`, `ITemplateQueries` e `ITemplateRepository`. La **lista de plantillas va por GraphQL**; las escrituras y el GET por id, por REST.

## Implementación

### Pasos (CREAR → MODIFICAR → ENLAZAR)
- [x] C1 Esquema y entidades (`db.sql`, scaffold de `Template` y `ConsumerTemplate`, restaurar `EmailDbContext`).
- [x] C2 Contratos, builder, puertos, `TemplateScope`, `ITemplateRenderer` y `ITemplateHtmlSanitizer`.
- [x] C3 Repositorios y sanitizador en `Infrastructure`.
- [x] C4 `TemplateService`, `TemplateQueries`, asignaciones y validaciones.
- [x] M1 `TemplatePermissions` a `Application/Permissions` con puerto y `scope` (hecho dentro de C4).
- [x] E1 Controller REST, módulo y GraphQL; asignación solo Admin.
- [x] E2 Verificación con curl.
- [x] E3 Panel: capa `api/templates`, Actions y dependencia.
- [x] E4a–E4d Panel: lista, editor con variables, vista previa y asignación.
- [x] E5 Documentación (README, `AGENTS.md`, wiki y esta nota).

Refactors hechos en el camino: `RegexLibrary`/`IRegexLibrary` (todas las regex en un solo lugar), consultas GraphQL por dominio con `IQueryMapper`, `AdminModule` separado en `ConsumerModule` y `ManagerModule` y módulos descubiertos en orden alfabético.

### API
- `POST /templates`, `GET /templates/{id}`, `PUT /templates/{id}`, `PATCH /templates/{id}` (`active`), `POST /templates/preview` (devuelve `subject`, `html` y `missing`), política `TemplateManager`.
- Lista: GraphQL `templates(page, pageSize, filter{search, active}, orderBy{NAME|CREATED_AT})`.
- Asignación solo Admin: `GET` y `PUT /consumers/{id}/templates` (`{ message: [ids], group: [ids] }`, reemplaza el conjunto).
- Validación: nombre ≤ 50, asunto ≤ 100, cuerpo ≤ 200 000 caracteres.

## Criterios de Aceptación

- [x] Un `TemplateManager` crea, edita y desactiva plantillas; un consumidor o manager sin ese rol recibe 403 (las rutas de gestión de plantillas devuelven 403; el 401 uniforme es solo de las de Admin).
- [x] Solo el Admin asigna plantillas a consumidores, con su tipo (un manager recibe 401).
- [ ] Una plantilla inactiva no se ofrece para enviar; el 1 a 1 y el envío a grupos exigen la asignación del tipo correspondiente (la regla está en `TemplatePermissions.RequireUseAsync`; se ejercita en los slices de envío).
- [x] La vista previa rellena las variables, codifica los valores y avisa de las que faltan.
- [x] El HTML guardado no contiene scripts, eventos `on*` ni URLs `javascript:`.
- [x] `dotnet build` y `bun run build` correctos.

## Pendientes para otros slices

- Volver a sanear el HTML tras sustituir variables en el envío (un valor puede acabar en un `href` o `src`).
- Si los managers deben listar plantillas, abrir GraphQL con autorización por campo.
- Los endpoints se guardan en Insomnia al cerrar la Fase 2.

## Relacionado

- [[email-service-plan]]
- [[email-service-consumidores-keys-sub-plan]]
- [[email-service-envio-individual-sub-plan]]
