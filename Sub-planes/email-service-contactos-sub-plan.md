---
tipo: sub-plan
titulo: Slice 5 — Contactos
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-30
estado: implementado
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - contactos
  - slice-5
---

# Slice 5 — Contactos

> **Plan padre:** [[email-service-plan]]
> Administrar contactos: alta, edición y baja de contactos locales, corrección local de cualquier contacto (también los del Core), detalle por id y exportación a Excel. La lista por GraphQL y la pantalla Contactos con la sincronización ya existen (adelantadas en el slice 4).

## Pendientes

- [x] Definir alcance con Efraín (fase de detallado, 2026-10-30).
- [x] Esquema: índice único de correo para locales (`uq_contacts_local_email`).
- [x] API: `GET /contacts/{id}`, `POST`, `PUT`, `DELETE` (baja lógica), `POST /contacts/{id}/restore`, `GET /contacts/export` (verificado con curl).
- [x] Panel: "Nuevo contacto", "Editar", "Dar de baja"/"Restaurar", "Exportar" (xlsx con `exceljs`); "Sincronizar" pasa a "Obtener" (verificado en Chrome).
- [x] Documentación (wiki, AGENTS, README) — 2026-10-30.
- [ ] Deploy a testing (imagen nueva, `CREATE UNIQUE INDEX uq_contacts_local_email …` en la base) y prueba end-to-end desde el panel.

## Estado Actual

| Área | Estado |
|------|--------|
| Tabla `contacts` con `kind`, `source`, `subscribed` y campos de baja | ✅ |
| Lista paginada por GraphQL (`contacts`) y `ContactPermissions.CanList` | ✅ (slice 4) |
| Pantalla Contactos con tabla por población y sincronización | ✅ (slice 4) |
| Alta, edición, baja, restauración y detalle por REST | ✅ |
| Exportación a Excel (API en JSON, xlsx en el panel) | ✅ |
| Deploy a testing y E2E | ⏳ |

## Ajustes durante la implementación (2026-10-30)

- El contexto local lee `LOCAL_CORE_DB_PASSWORD` del `.env` (`Secrets__CORE_DB_PASSWORD`); una variable `Secrets__*` con valor manda sobre la clave de OpenBao (antes OpenBao pisaba el entorno).
- `tsconfig.json` del panel sin `baseUrl` (en desuso en TypeScript 6).
- La insignia de estado distingue "Baja" (local) de "Baja del Core".

## Decisiones (fase de detallado, 2026-10-30)

1. **Quién:** Admin y Manager con `GroupManager` (`ContactPermissions.CanManage`); `CanList` se mantiene.
2. **Se puede editar cualquier contacto** (`PUT /contacts/{id}`: correo y nombre), también los del Core, para corregir en local un dato desactualizado. La siguiente revisión del Core lo mostrará en "con diferencias" y el administrador decide si lo pisa. No se editan `kind`, `source`, `coreId` ni la suscripción.
3. **Correo único para los locales frente a todos los contactos** (sin distinguir mayúsculas) → 409 al crear o editar. Entre poblaciones del Core sigue habiendo un contacto por población. Índice único parcial `uq_contacts_local_email` (`LOWER(email)` donde `kind = 'LOCAL'`).
4. **Baja lógica solo de locales** (`DELETE` → `deleted_at`; `restore` la deshace). Un contacto del Core se da de baja desde la sincronización (409 si se intenta por REST). Nada se borra físicamente.
5. **Suscripción:** fuera de este slice (slice 9).
6. **Exportación a Excel:** la API devuelve todas las filas filtradas en JSON (`GET /contacts/export`, sin paginar) y el **panel** genera el `.xlsx` en el servidor de Astro (`exceljs`) desde `GET /contactos/exportar`. Columnas: Nombre, Correo, Población, Origen, Suscrito, Sincronizado, Estado.
7. **Importación CSV/Excel:** nota para la v2.
8. **Panel:** misma pantalla Contactos, con acciones por fila como en Managers; el botón "Sincronizar {población}" pasa a llamarse "Obtener {población}".

## Alcance

Cubre la gestión de contactos (locales y corrección de cualquiera), la baja lógica de locales y la exportación. No cubre la suscripción ni el enlace de baja (slice 9), los grupos (slice 6) ni la importación (v2).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: `IContactService`, `IContactQueries` e `IContactRepository` (patrón de managers); REST para escrituras y detalle; GraphQL para la lista. Ciclo CREAR → MODIFICAR → ENLAZAR con un sub-agente por paso.

## Implementación

### Paso 1 — Base de datos
- [ ] `CREATE UNIQUE INDEX uq_contacts_local_email ON contacts (LOWER(email)) WHERE kind = 'LOCAL';` en `infra/db.sql` y como ALTER en la wiki.

### Paso 2 — API
- [ ] `Application/Contacts`: `ContactRequest`, `ContactResponse`, `ContactCreatedResponse`, `ContactService`, `ContactQueries.GetAsync/StreamAsync`, `ContactPermissions.CanManage`.
- [ ] `Infrastructure`: `ContactRepository` (`EmailExistsAsync`, `FindAsync`, `FindForUpdateAsync`, `Add`, `StreamAsync`).
- [ ] `Api/Contacts/ContactController` (`GroupManagerPolicy`, rate limit Moderate) y `ContactModule` como `IEndpointMapper`; `JsonRegistry`.

### Paso 3 — Panel
- [ ] `api/contacts` (contratos y servicio), Actions `contacts.get/create/update/remove/restore`, `ContactFormModal`, acciones por fila, botones "Obtener", "Nuevo contacto" y "Exportar", ruta `pages/contactos/exportar.ts` con `exceljs`.

### Paso 4 — Documentación
- [ ] Wiki (API, Modelo de datos, Reglas de negocio, Panel, Capas, Seguridad), AGENTS y README.

## Criterios de Aceptación

- [ ] Un `GroupManager` crea, edita, da de baja y restaura contactos locales; un manager sin ese rol recibe 403 y sin key 401.
- [ ] Un correo repetido (con otra capitalización o de un contacto del Core) da 409.
- [ ] Editar un contacto del Core cambia correo y nombre en local y la siguiente revisión lo muestra como diferencia.
- [ ] Dar de baja un contacto del Core por REST da 409.
- [ ] La exportación descarga un `.xlsx` con las filas del filtro actual.
- [ ] `dotnet build` y `bun run build` correctos.

## Nota para la v2 — importación

Importar contactos locales desde CSV o Excel (mismas columnas que la exportación), con validación por fila, informe de omitidos y la misma regla de correo único.

## Relacionado

- [[email-service-plan]]
- [[email-service-sync-core-sub-plan]]
- [[email-service-grupos-sub-plan]]
