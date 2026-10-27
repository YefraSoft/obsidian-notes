---
tipo: sub-plan
titulo: Slice 6 — Grupos, miembros y asignaciones
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: pendiente
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
> Crear y gestionar grupos de contactos y decidir qué consumidores pueden enviar a ellos o administrarlos.

## Pendientes

- [ ] CRUD de grupos con soft delete.
- [ ] Altas y bajas de miembros.
- [ ] Asignaciones del administrador (`GroupSender` / `GroupManager`).
- [ ] Pantallas del panel.

## Estado Actual

| Área | Estado |
|------|--------|
| Tablas `groups`, `contact_groups` y `consumer_group_roles` | ✅ |
| `GroupPermissions` (`CanCreate`, `CanManage`, `RequireSendAsync`, `RequireManageAsync`) | ✅ |
| Roles `GroupSender` y `GroupManager` con sus políticas | ✅ |
| Endpoints, contratos y builders | ⏳ |
| Panel | ⏳ |

## Alcance

Cubre grupos, sus miembros y las asignaciones consumidor × grupo × rol. No cubre el envío a grupos (Slice 8) ni grupos dinámicos (roadmap). Quien crea un grupo y tiene el rol `GroupManager` queda asignado a él con ese rol; el administrador pasa el alcance sin fila.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: servicios, queries y repositorios de grupos y asignaciones; `GroupPermissions` pasa a usar puertos en lugar de `EmailDbContext`. La **lista de grupos con sus miembros va por GraphQL**; las escrituras y el GET por id, por REST.

## Implementación

### Paso 1 — Base de datos
- [x] `consumer_group_roles(id_consumer, id_group, role)` con clave primaria compuesta.
- [ ] Revisar índices y la política de borrado (soft delete del grupo, cascada de asignaciones y miembros).

### Paso 2 — API
- [ ] `GroupController` y tipo GraphQL en `Api`; `GroupService`, `GroupQueries`, contratos y builder en `Application/Groups`; repositorios en `Infrastructure`.
- [ ] CRUD de grupos; al crear, insertar la fila `(consumidor, grupo, GroupManager)`.
- [ ] `PUT`/`DELETE` de miembros (`GroupPermissions.RequireManageAsync`).
- [ ] Asignaciones del Admin: listar, crear y quitar filas de `consumer_group_roles` validando el rol con `IsEnum`.

### Paso 3 — Panel
- [ ] Lista de grupos, miembros y selector de contactos.
- [ ] Asignación de grupos y roles desde la ficha del consumidor.

### Paso 4 — Documentación
- [ ] README: matriz de dos capas y endpoints de grupos.

## Criterios de Aceptación

- [ ] Un `GroupManager` solo gestiona grupos con fila `GroupManager`; en otro grupo recibe 403 con `code: ResourceNotGranted`.
- [ ] El administrador gestiona cualquier grupo sin fila.
- [ ] Asignar un rol que no sea `GroupSender` o `GroupManager` da 400.
- [ ] Borrar un grupo conserva el historial de lotes.
- [ ] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- ¿Hay un límite de miembros por grupo?
- ¿Un contacto dado de baja sigue apareciendo como miembro?

## Relacionado

- [[email-service-plan]]
- [[email-service-contactos-sub-plan]]
- [[email-service-envio-grupal-sub-plan]]
