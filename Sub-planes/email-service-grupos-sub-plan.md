---
tipo: sub-plan
titulo: Slice 6 — Grupos, miembros y asignaciones
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-11-09
estado: implementado
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - grupos
  - asignaciones
  - slice-6
---

# Slice 6 — Grupos, miembros y asignaciones

> **Plan padre:** [[email-service-plan]]
> Crear y gestionar grupos de contactos (con dueño), sus miembros, y decidir a qué grupos puede enviar cada consumidor.

## Pendientes

- [x] Definir alcance con Efraín (fase de detallado, 2026-10-30).
- [x] Esquema: `groups.id_manager`, `created_at`, `uq_groups_name` (2026-10-30).
- [x] API: grupos (alta, detalle, edición, baja, restauración), miembros (agregar por tres orígenes con exclusiones, quitar) y asignación de grupos a consumidores; lista de grupos y de miembros por GraphQL. Verificado con curl (2026-10-30).
- [x] Panel: `/grupos`, `/grupos/[id]` (miembros) y botón "Grupos" en la ficha del consumidor. Verificado en Chrome (2026-10-30).
- [x] Ajustes pedidos tras la prueba (2026-10-30): "Población completa" como lista marcada con posibilidad de desmarcar (`excludeContactIds`) y modal de alto fijo; borradores de plantilla en `localStorage` (ver [[email-service-plantillas-sub-plan]]).
- [x] Documentación (wiki, AGENTS, README) — 2026-11-03.
- [ ] Deploy a testing al cerrar la fase (junto con los slices 3 a 5, con sus `ALTER`) y E2E.

## Estado Actual

| Área | Estado |
|------|--------|
| Tablas `groups` (con dueño, `created_at` y nombre único), `contact_groups` y `consumer_group_roles` | ✅ |
| Roles `GroupSender` y `GroupManager` con sus políticas | ✅ |
| `GroupPermissions` con puertos en `Application/Permissions` (alcance por dueño) | ✅ (el de `Platform` se eliminó) |
| Servicios, queries, repositorios, contratos y builders | ✅ |
| Endpoints REST y campos GraphQL (`groups`, `groupMembers`) | ✅ |
| Panel (`/grupos`, miembros, asignación desde consumidores) | ✅ |
| Deploy a testing y E2E | ⏳ |

## Decisiones (fase de detallado, 2026-10-30)

1. **Dueño en vez de asignación para managers.** `GroupManager` es un rol del dominio Manager y `consumer_group_roles` solo referencia a `consumer`, así que el modelo original (fila `GroupManager` por grupo) no encajaba. Cada grupo tiene dueño (`groups.id_manager`): el manager con `GroupManager` que lo creó, que solo ve y gestiona los suyos; el administrador gestiona todos, sus grupos no tienen dueño (`NULL`) y es el único que puede indicar o cambiar el dueño (`ownerId`). Grupo ajeno → `403 ResourceNotGranted`.
2. **`consumer_group_roles` se queda** pero solo con filas `GroupSender`: a qué grupos puede enviar cada consumidor. Lo asigna el administrador desde la ficha del consumidor (`GET/PUT consumers/{id}/groups`, reemplazo completo, como las plantillas).
3. **Nombre único global** entre grupos vigentes, sin distinguir mayúsculas (`409`); índice parcial `uq_groups_name` como red de seguridad. `created_at` para ordenar la lista.
4. **Miembros:** el backend recibe ids o un origen que resuelve él mismo. `POST groups/{id}/members` con `{ contactIds?, kind?, fromGroupId?, excludeContactIds? }` (al menos un origen; `kind` = población completa; `fromGroupId` = otro grupo que el actor pueda gestionar; `excludeContactIds` descarta contactos de esos dos orígenes, los que el usuario desmarca en el panel). Solo entran contactos vigentes y suscritos, los ya miembros se ignoran; responde `{ added, skipped }`. Quitar: `POST groups/{id}/members/remove` con `{ contactIds }`. La lista de miembros va por GraphQL (`groupMembers`).
5. **Salida automática:** un contacto dado de baja (REST o sincronización) o no suscrito sale de todos los grupos; restaurarlo no lo devuelve. La baja de suscripción (slice 9) debe llamar al mismo puerto (`IGroupMemberRepository.RemoveContactsAsync`).
6. **Baja lógica del grupo** (`deleted_at`, idempotente) conservando miembros y asignaciones; `restore` la deshace (`409` si otro grupo vigente ya usa el nombre; se puede editar un grupo dado de baja para renombrarlo). Nada se borra físicamente (`batches` referencia grupos).
7. Sin límite de miembros (inserciones por lotes de 1000); un contacto dado de baja nunca aparece como miembro (respuestas a las preguntas abiertas).
8. **Panel:** página nueva `/grupos` (Admin y `GroupManager`) con lista y acciones por fila (Miembros, Editar, Dar de baja / Restaurar), `/grupos/[id]` con la tabla de miembros, "Quitar seleccionados" y el modal "Agregar miembros" con tres orígenes (Población completa, De otro grupo, Buscar contactos); botón "Grupos" en la ficha del consumidor para asignar a qué grupos puede enviar.

## Ajustes durante la implementación (2026-10-30)

- `PUT groups/{id}` del administrador reemplaza el dueño (`ownerId` ausente = sin dueño); un manager no puede indicarlo y conserva el suyo.
- "Población completa" en el modal de miembros se muestra como la misma lista de "Buscar contactos" sin buscador, con todos marcados y posibilidad de desmarcar; el modal tiene alto fijo (`60dvh`) para no saltar al cambiar de origen.
- `ContactRepository.Query` se extrajo a `ContactQueryable` (filtro y orden de contactos) para reutilizarlo en la página de miembros.
- Verificación local: 29 casos con curl (acceso, dueño, nombre único, miembros por los tres orígenes, salida automática, GraphQL y asignaciones) y flujo completo en Chrome como Admin y como manager.

## Alcance

Cubre grupos (con dueño), sus miembros y la asignación consumidor × grupo (`GroupSender`). No cubre el envío a grupos (slice 8), la suscripción (slice 9), la población filtrada como origen de miembros (v2) ni grupos dinámicos (roadmap).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: `IGroupService`, `IGroupGrantService`, `IGroupQueries`; puertos `IGroupRepository`, `IGroupMemberRepository` e `IGroupGrantRepository`; `GroupPermissions` pasa a `Application/Permissions` con puertos (se borra el de `Platform`). REST para escrituras y detalle; GraphQL para las listas (`groups`, `groupMembers`). Ciclo CREAR → MODIFICAR → ENLAZAR con un sub-agente por paso.

## Implementación

### Paso 1 — Base de datos
- [x] `ALTER TABLE groups ADD COLUMN id_manager UUID` (FK a `manager`, `ON DELETE SET NULL`), `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `ix_groups_manager` y `CREATE UNIQUE INDEX uq_groups_name ON groups (LOWER(name)) WHERE deleted_at IS NULL;` en `infra/db.sql` y como ALTER en la wiki. Entidad `Group`/`Manager` y `EmailDbContext` editados a mano.

### Paso 2 — API
- [x] `Application/Groups`: contratos (`GroupRequest`, `GroupResponse`, `AddGroupMembersRequest/Response`, `RemoveGroupMembersRequest`, `ConsumerGroupsRequest/Response`…), builder, puertos, `GroupService`, `GroupGrantService`, `GroupQueries`; `Application/Permissions/GroupPermissions` (`RequireManageAsync` por dueño, `RequireSendAsync` para el slice 8).
- [x] `Infrastructure`: `GroupRepository`, `GroupMemberRepository` (inserción por lotes con `ON CONFLICT DO NOTHING`), `GroupGrantRepository`; `ContactQueryable` extraído de `ContactRepository`.
- [x] Hooks de salida automática en `ContactService.DeleteAsync` y en la aplicación de bajas de `SyncService`.
- [x] `Api/Groups`: `GroupController` (`GroupManagerPolicy`, Moderate), `ConsumerGroupController` (`AdminPolicy`), `GroupModule` con los campos GraphQL; `JsonRegistry`; tipos GraphQL.

### Paso 3 — Panel
- [x] `api/groups`, `actions/groups`, `PanelAccess` (`/grupos` → `GroupManager`), `GroupsScreen`, `GroupFormModal`, `GroupMembersScreen`, `AddMembersModal`, `ConsumerGroupsModal` y botón "Grupos" en consumidores.

### Paso 4 — Documentación
- [x] Wiki (API, Modelo de datos, Reglas de negocio, Seguridad, Panel, Capas, Despliegue con el ALTER), AGENTS (quitar la deuda de `GroupPermissions`, nota del slice 9) y README.

## Criterios de Aceptación

- [x] Un `GroupManager` crea grupos y solo gestiona los suyos; en un grupo ajeno recibe `403` con `code: ResourceNotGranted`; sin el rol `403`; sin key `401`.
- [x] El administrador gestiona cualquier grupo y puede fijar el dueño.
- [x] Un nombre repetido entre grupos vigentes (con otra capitalización) da `409`.
- [x] Agregar miembros por ids, por población y desde otro grupo propio devuelve `added`/`skipped`; los dados de baja, no suscritos e inexistentes se omiten; un grupo ajeno como origen da `403`.
- [x] Dar de baja un contacto (REST o sincronización) lo saca de sus grupos y restaurarlo no lo devuelve (probado por REST; la sincronización usa el mismo puerto).
- [x] Dar de baja un grupo conserva el historial de lotes, sus miembros y asignaciones.
- [x] Asignar a un consumidor un grupo inexistente o dado de baja da `400`.
- [x] `dotnet build` y `bun run build` correctos.

## Nota para la v2

Origen de miembros por población filtrada (por ejemplo, alumnos de un plantel), apoyado en la profundidad de filtros de las consultas del Core (nota v2 de [[email-service-sync-core-sub-plan]]).

## Relacionado

- [[email-service-plan]]
- [[email-service-contactos-sub-plan]]
- [[email-service-envio-grupal-sub-plan]]
