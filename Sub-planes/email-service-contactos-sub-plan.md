---
tipo: sub-plan
titulo: Slice 5 — Contactos
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: pendiente
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
> Consultar los contactos (del Core y locales), dar de alta contactos propios y manejar la suscripción.

## Pendientes

- [ ] Listado con búsqueda y paginación.
- [ ] Alta, edición y baja de contactos locales.
- [ ] Suscripción y baja local.
- [ ] Pantalla de contactos en el panel.

## Estado Actual

| Área | Estado |
|------|--------|
| Tabla `contacts` con `source`, `subscribed` y campos de baja | ✅ |
| `ContactPermissions` (`CanCreate`/`RequireCreate`, rol `GroupManager`) | ✅ |
| Endpoints, contratos y builders | ⏳ |
| Panel | ⏳ |

## Alcance

Cubre los contactos ya sincronizados (Slice 4) y los locales (`source = LOCAL`). No cubre la pertenencia a grupos (Slice 6) ni el enlace de baja público (Slice 9). Un contacto del Core es de solo lectura salvo sus campos locales.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: `IContactService`, `IContactQueries` e `IContactRepository`. La **lista paginada de contactos va por GraphQL** (es la que más crece); el alta, la edición, la baja y el GET por id, por REST.

## Implementación

### Paso 1 — Base de datos
- [ ] Índices de búsqueda (correo, nombre) si hacen falta.

### Paso 2 — API
- [ ] `ContactController` y tipo GraphQL en `Api`; `ContactService`, `ContactQueries`, contratos y builder en `Application/Contacts`; repositorio en `Infrastructure`.
- [ ] `GET` paginado con filtro; `POST`/`PUT`/`DELETE` solo para `LOCAL` (`GroupManager`/Admin).
- [ ] Cambiar la suscripción de un contacto.
- [ ] Validación nativa: correo (`[EmailAddress]`), largos y duplicados.

### Paso 3 — Panel
- [ ] Tabla de contactos con búsqueda, alta local y estado de suscripción.

### Paso 4 — Documentación
- [ ] README: reglas de contactos locales y del Core.

## Criterios de Aceptación

- [ ] Un `GroupManager` crea contactos locales; un consumidor sin ese rol recibe 403 con `code: RoleNotAllowed`.
- [ ] Editar un contacto del Core solo permite sus campos locales.
- [ ] Un correo repetido da 409.
- [ ] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- ¿Quién puede listar contactos (todos los roles, solo `GroupManager`/Admin)?
- ¿Se permite importar contactos locales en lote (CSV)?

## Relacionado

- [[email-service-plan]]
- [[email-service-sync-core-sub-plan]]
- [[email-service-grupos-sub-plan]]
