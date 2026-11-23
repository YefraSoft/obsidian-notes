---
tipo: sub-plan
titulo: Slice 4 — Sincronización con el Core
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-11-09
estado: hecho
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
- [x] Consultas SQL de las cuatro poblaciones (aspirantes entregada el 2026-10-30).
- [x] Lector del Core (`MySqlConnector`) con credenciales en OpenBao / `Secrets`.
- [x] Revisión (`POST sync/{kind}`) y aplicación (`POST sync/{kind}/apply`) por población; rate limit `Sync` por IP y endpoint.
- [x] Lista de contactos por GraphQL (adelantada del slice 5) y sección Contactos del panel con el modal de revisión.
- [x] Documentación (wiki, AGENTS, README) — 2026-10-30.
- [x] Permisos: el `GroupManager` sincroniza y entra a `/contactos` (2026-10-30).
- [x] Deploy a testing (imagen 0.0.0.03, base regenerada con `infra/db.sql`, variables `EMAIL_SENDER_CORE_DB_*` en Dokploy) y prueba end-to-end desde el panel (2026-10-30).

## Estado Actual

| Área | Estado |
|------|--------|
| Tabla `contacts` con `core_id`, `source`, `subscribed`, `unsubscribed_*`, `deleted_at` | ✅ |
| `contacts.kind` (población) y unicidad `(kind, core_id)` | ✅ |
| Conexión al Core, revisión y aplicación por población (probado en local contra el Core real) | ✅ |
| Panel (sección Contactos con sincronización) | ✅ |
| Deploy a testing y prueba end-to-end | ✅ |

## Ajustes tras la primera prueba (2026-10-30)

- Rate limit `Sync` independiente por endpoint (IP + ruta): agotar estudiantes no afecta a aspirantes.
- La revisión devuelve las filas omitidas con motivo (`EMPTY_EMAIL`, `INVALID_EMAIL`, `EMAIL_TOO_LONG`, `DUPLICATE_CORE_ID`); el panel las muestra en una vista aparte ("Contactos omitidos").
- Modal rediseñado con la guía `frontend-design`: resumen en cifras (acento `primary`, sin rojo ni verde), buscador y chips de filtro en cliente, lista `[casilla] [nombre] [correo]` con alto fijo y paginación de 25, icono para ver diferencias.
- `FetchService` del panel admite timeout por petición (sincronización 180 s) y registra las peticiones fallidas (`[api] …`).
- Primera revisión real: ~87.900 estudiantes leídos, ~4.400 omitidos por correo vacío; la consulta tarda menos de un segundo.
- **El `GroupManager` también sincroniza** (revisa y aplica) y entra a `/contactos`; se descartó un rol `ContactManager` con alcances: para enviar a un subconjunto (p. ej. Finanzas) se crea un grupo.

## Cierre (2026-10-30)

Slice cerrado tras el E2E en testing (API 0.0.0.03, panel contra `http://192.168.14.131:5037`):
- Revisión y aplicación de aspirantes contra el Core real desde el panel; los contactos quedan listados.
- Managers y consumidor de prueba creados, plantilla con variable creada, previsualizada y asignada; acceso por rol verificado (GroupManager → Contactos, TemplateManager → Plantillas).
- Incidencias resueltas durante el deploy: Dokploy reutilizaba la imagen `latest` en caché (`pull_policy: always` en el compose); la contraseña del Core con `#` se truncaba en el `.env` de Dokploy (se cambió la contraseña); bucle de redirección del panel ante un 401 (el middleware cierra la sesión al llegar a `/` con `notice`).
- Criterios de aceptación cumplidos: segunda revisión sin cambios devuelve listas vacías; la baja local sobrevive a la aplicación; los contactos que desaparecen del Core quedan con `deleted_at`; sin Core la API sigue funcionando (503 en `sync/*`, `coredb` Degraded); la cuarta petición por endpoint en 5 minutos recibe 429.

## Nota para la v2 — profundidad de los filtros

Añadir a las consultas del Core columnas de contexto (plantel, carrera y grupo para estudiantes; departamento para administrativos; materia para docentes) con sus `JOIN`, copiarlas a `contacts` como columnas nullable y exponerlas como filtros de la lista, para que el `GroupManager` arme grupos filtrados (p. ej. estudiantes de Sistemas). Solo añade campos al modelo que se lee del Core; la revisión y la aplicación no cambian.

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
- [x] `contacts.kind` con `CHECK`, unicidad `(kind, core_id)` e índice `(kind, deleted_at)`; `ALTER` en la wiki; scaffold de `Contact`.

### Paso 2 — API
- [x] `Application/Sync` (puerto, diff, servicio, contratos) y `Application/Contacts` (queries y contratos).
- [x] `Infrastructure/CoreSync` (opciones, factory, sentencias SQL, lector, health check `coredb`) y repositorios.
- [x] `SyncController` + `SyncModule`, `ContactModule` (GraphQL `contacts`), rate limit `Sync`, excepciones `CoreNotConfigured`, `CoreEmpty`, `CoreUnavailable`.

### Paso 3 — Panel
- [x] `api/contacts`, `api/sync`, Actions, `ContactsScreen`, `SyncReviewModal`, página `/contactos`, acceso en `PanelAccess`.

### Paso 4 — Documentación
- [x] Wiki (API, Modelo de datos, Despliegue, Panel, Capas, Seguridad), AGENTS y README.

## Criterios de Aceptación

- [x] Una segunda revisión sin cambios en el Core devuelve las tres listas vacías.
- [x] Una baja local (`subscribed = false`) sobrevive a una aplicación.
- [x] Un contacto que desaparece del Core aparece en `removed` y, al aplicarlo, queda con `deleted_at` sin perder su historial.
- [x] Si el Core no responde o no está configurado, la API sigue funcionando y responde 503 con su `code`.
- [x] La cuarta petición de sincronización en 5 minutos recibe 429.
- [x] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- ¿`CONCAT_WS` en lugar de `CONCAT` para que `secondName` NULL no anule el nombre? (las consultas no se cambian sin el visto bueno de Efraín). Aplazado a la versión 1.1.0 (decisión 2026-11-09).
- ¿Filtrar vigencia (`status`/`opc`) en las consultas del Core o dejar que entren todos los registros? Aplazado a la versión 1.1.0 (decisión 2026-11-09).

## Relacionado

- [[email-service-plan]]
- [[email-service-contactos-sub-plan]]
