---
tipo: sub-plan
titulo: Slice 4 — Sincronización con el Core
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-29
estado: en-curso
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - core
  - sincronizacion
  - slice-4
---

# Slice 4 — Sincronización con el Core

> **Plan padre:** [[email-service-plan]]
> Traer a PostgreSQL los contactos del Core (EDCORE, MariaDB, esquema `core`, solo lectura) como **revisión asistida**: por población se piden los cambios al Core, la app muestra qué hay de nuevo, qué cambió y qué ya no existe, y el administrador autoriza qué aplicar.

## Pendientes

- [x] Definir tablas, campos y poblaciones del Core (fase de detallado, 2026-10-29).
- [ ] Consulta SQL de aspirantes (`CapAspirantes`); las de estudiantes, personal y administrativos ya están.
- [ ] Lector del Core (`MySqlConnector`) con credenciales en `Secrets`.
- [ ] Revisión (`POST sync/{kind}`) y aplicación (`POST sync/{kind}/apply`) por población.
- [ ] Lista de contactos por GraphQL (adelantada del slice 5) y sección Contactos del panel con el modal de revisión.
- [ ] Documentación (wiki, AGENTS, README).

## Estado Actual

| Área | Estado |
|------|--------|
| Tabla `contacts` con `core_id`, `source`, `subscribed`, `unsubscribed_*`, `deleted_at` | ✅ |
| `contacts.kind` (población) y unicidad `(kind, core_id)` | ⏳ |
| Conexión al Core, revisión y aplicación por población | ⏳ |
| Panel (sección Contactos con sincronización) | ⏳ |

## Decisiones (fase de detallado, 2026-10-29)

1. **Poblaciones:** estudiantes (`UserStudents`), personal (`UserPersonal`), administrativos (`UserAdmin`) y aspirantes (`CapAspirantes`). Las consultas SQL las entrega Efraín (tardan ~800 ms) y devuelven `id`, `email` y `display_name`; no filtran vigencia. Solo se copian esos tres datos (ni CURP ni teléfonos).
2. **Población en la copia local:** columna `contacts.kind` (`STUDENT`, `PERSONAL`, `ADMIN`, `APPLICANT`, `LOCAL`), enum `ContactKind` en mayúsculas. Es el filtro por categoría de la sección Contactos y la base de los grupos por población (slice 6).
3. **Identidad única por `(kind, core_id)`:** los ids del Core chocan entre tablas y una persona puede estar en dos poblaciones; se permite un contacto por población (mismo correo repetido con distinto `kind`). El envío a grupos deduplicará por correo (slice 8).
4. **Sin job periódico, sin tabla temporal ni `sync_runs`.** Todo es bajo demanda y síncrono; la comparación se hace en memoria (~95.500 estudiantes caben sin problema) y la escritura va con EF por lotes.
5. **Dos pasos por población, endpoints separados** (`students | personal | admins | applicants`, solo Admin, rate limit `Sync`: 3 peticiones por 5 minutos):
   - `POST sync/{kind}`: **revisión**, no escribe. Devuelve `added`, `changed` (incluye a los que reaparecen tras una baja) y `removed`, cada lista con `total` y recortada a `limit` (500 por defecto, tope 5000), más `read` y `skipped` (sin correo o inválido).
   - `POST sync/{kind}/apply`: recibe `{ added, changed, removed }` con `"all"` o una lista de `coreId`; vuelve a leer y comparar (sin estado entre llamadas) y aplica solo lo autorizado. Nunca toca `subscribed`, `unsubscribed_at` ni `unsubscribed_reason`. Si el Core devuelve 0 filas → 503 `CoreEmpty`.
6. **Conexión al Core:** `CoreDb:Host`, `CoreDb:Port`, `CoreDb:Database`, `CoreDb:CommandTimeoutSeconds` en configuración; `Secrets:CORE_DB_USER` y `Secrets:CORE_DB_PASSWORD` como secretos. Opcional mientras testing no llegue al Core: sin credenciales `POST sync/*` → 503 `CoreNotConfigured` y `health/ready` queda `Degraded`.
7. **Panel:** sección **Contactos** (`/contactos`, Admin): tabla por GraphQL filtrada por categoría, búsqueda y paginación; botón "Sincronizar <categoría>" con modal de nuevos, cambios y bajas, casillas, "todos" y "Aplicar". Agregar los nuevos a grupos es del slice 6.
8. Puerto `ICoreContactSource` con implementación `MySqlConnector` en `Infrastructure/CoreSync`.

## Alcance

Cubre la lectura del Core, la revisión y aplicación por población, la lista de contactos (lectura) y la sección Contactos del panel con la sincronización. No cubre el alta, edición ni suscripción de contactos locales (slice 5), los grupos (slice 6) ni un job periódico.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: `ICoreContactSource` (puerto en `Application/Sync`) con su implementación MySQL en `Infrastructure/CoreSync`; `ISyncService` calcula el diff con `ContactSyncDiffer` (puro) y escribe por `IContactSyncRepository`; `IContactQueries` para la lista. Ciclo CREAR → MODIFICAR → ENLAZAR con un sub-agente por paso.

## Implementación

### Paso 1 — Base de datos
- [ ] `contacts.kind` con `CHECK`, unicidad `(kind, core_id)` e índice `(kind, deleted_at)`; `ALTER` en la wiki; scaffold de `Contact`.

### Paso 2 — API
- [ ] `Application/Sync` (puerto, diff, servicio, contratos) y `Application/Contacts` (queries y contratos).
- [ ] `Infrastructure/CoreSync` (opciones, factory, sentencias SQL, lector, health check `coredb`) y repositorios.
- [ ] `SyncController` + `SyncModule`, `ContactModule` (GraphQL `contacts`), rate limit `Sync`, excepciones `CoreNotConfigured`, `CoreEmpty`, `CoreUnavailable`.

### Paso 3 — Panel
- [ ] `api/contacts`, `api/sync`, Actions, `ContactsScreen`, `SyncReviewModal`, página `/contactos`, acceso en `PanelAccess`.

### Paso 4 — Documentación
- [ ] Wiki (API, Modelo de datos, Despliegue, Panel, Capas, Seguridad), AGENTS y README.

## Criterios de Aceptación

- [ ] Una segunda revisión sin cambios en el Core devuelve las tres listas vacías.
- [ ] Una baja local (`subscribed = false`) sobrevive a una aplicación.
- [ ] Un contacto que desaparece del Core aparece en `removed` y, al aplicarlo, queda con `deleted_at` sin perder su historial.
- [ ] Si el Core no responde o no está configurado, la API sigue funcionando y responde 503 con su `code`.
- [ ] La cuarta petición de sincronización en 5 minutos recibe 429.
- [ ] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- Consulta SQL de aspirantes (pendiente de Efraín).
- ¿`CONCAT_WS` en lugar de `CONCAT` para que `secondName` NULL no anule el nombre?

## Relacionado

- [[email-service-plan]]
- [[email-service-contactos-sub-plan]]
