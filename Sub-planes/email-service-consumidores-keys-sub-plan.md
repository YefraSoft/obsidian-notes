---
tipo: sub-plan
titulo: Slice 2 — Consumidores, managers y keys
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: en-progreso
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - consumidores
  - managers
  - keys
  - slice-2
---

# Slice 2 — Consumidores, managers y keys

> **Plan padre:** [[email-service-plan]]
> Que el administrador cree consumidores (apps) y managers (personas con email) con los roles de su dominio, genere o rote su key y los gestione desde el panel. La base del modelo se construye en [[email-service-fase0-temporal]].

## Pendientes

- [ ] Fase 0 temporal: dominios, key cifrada y una key por dueño ([[email-service-fase0-temporal]]).
- [ ] `PATCH /consumers/{id}` y `PATCH /managers/{id}`: `active` y roles (validados contra el dominio).
- [ ] Rotar la key (`POST …/{id}/key`) con `expireAt` opcional y revocarla (`POST …/{id}/key/revoke`).
- [ ] Listas por GraphQL con `filter` (nombre o email) y `orderBy`.
- [ ] Panel: pantallas de consumidores y managers (alta, activar/desactivar, roles, rotar key con vencimiento, errores por campo).

## Estado Actual

| Área | Estado |
|------|--------|
| `POST /consumers` (nombre + roles, key en claro una sola vez) | ✅ |
| `GET /consumers/{id}` | ✅ |
| Lista de consumidores por GraphQL (paginada) | ✅ |
| Validación nativa del nombre y los roles | ✅ |
| `POST /consumers/{id}/keys` y `POST /keys/{id}/revoke` | ✅ (se reemplazan por rotar/revocar por dominio) |
| Dominio Manager (tablas, servicios, rutas, GraphQL) | ⏳ Fase 0 temporal |
| Una key vigente por dueño y rotación que reemplaza al instante | ⏳ Fase 0 temporal |
| Activar/desactivar y editar roles (`PATCH`) | ⏳ |
| Filtro y orden en GraphQL | ⏳ |
| Panel | ⚠ desalineado con la API; pendiente de migrar |

## Alcance

Cubre consumidores, managers y sus keys, y su pantalla de administración. No cubre la asignación de plantillas (Slice 3) ni de grupos (Slice 6); esas se agregan en sus slices, enlazadas al dueño que corresponda.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: un servicio por agregado (`ConsumerService`, `ManagerService`, servicios de keys por dominio), `Queries` para lecturas, repositorios por dominio. La **lista va por GraphQL**; GET por id y escrituras, por REST. Rotar es reemplazar: la key anterior se revoca en el acto y el consumidor se actualiza por su cuenta; `expire_at` opcional para keys temporales (por ejemplo, una campaña).

## Implementación

### Paso 1 — Base de datos
- [ ] `consumer`, `manager`, `consumer_keys` y `manager_keys` en `infra/db.sql`, con índice único parcial por dueño (`WHERE revoked_at IS NULL`).

### Paso 2 — API
- [x] `ConsumerController`, `KeyController`, servicios, queries, contratos y builders por capas.
- [ ] `ManagerController`, `PATCH` por dominio, rotar y revocar por dominio.
- [ ] GraphQL: `consumers` y `managers` con `filter` y `orderBy`.
- [ ] Un consumidor o manager con mensajes o lotes no se borra: se desactiva.

### Paso 3 — Panel
- [ ] Formularios de consumidor y manager con los roles de su dominio, listado, rotación con vencimiento e interruptor activar/desactivar.
- [ ] Mostrar los `errors` por campo de los 400.
- [ ] Arquitectura tomada del frontend de `bolsa-trabajo-tsj` (ver `AGENTS.md`).

### Paso 4 — Documentación
- [ ] README y wiki: tabla de endpoints al día.

## Criterios de Aceptación

- [ ] Crear un consumidor o manager devuelve la key una sola vez; los listados nunca traen keys ni hashes.
- [ ] Un rol de otro dominio → 400 con `errors`; nombre o email repetido → 409.
- [ ] Rotar deja la key anterior inservible al instante; revocar dos veces es idempotente; una key revocada o vencida responde 401.
- [ ] Un dueño desactivado deja de autenticarse (401) sin tocar su key.
- [ ] El panel refleja los cambios y la key nueva se muestra una sola vez.
- [ ] `dotnet build` y `bun run build` correctos.

## Relacionado

- [[email-service-plan]]
- [[email-service-fase0-temporal]]
- [[email-service-auth-sub-plan]]
- [[email-service-plantillas-sub-plan]]
