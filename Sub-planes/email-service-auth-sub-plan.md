---
tipo: sub-plan
titulo: "Slice 1 — Auth: aceptar la API key"
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: en-progreso
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - auth
  - api-key
  - slice-1
---

# Slice 1 — Auth: aceptar la API key

> **Plan padre:** [[email-service-plan]]
> Que la API reconozca a quien llama por su `X-Api-Key` (administrador o consumidor), con sus roles, y que el panel entre solo con la key de administrador.

## Pendientes

- [ ] Migrar el login del panel a `GET /whoami` y exigir el rol `Admin` en la respuesta.
- [ ] Leer los `errors` de validación (400) en el cliente del panel.
- [ ] Mostrar los roles en el dashboard.
- [ ] Actualizar la tabla de permisos del README (aún usa `SEND_MESSAGE / CREATE_GROUP / READ_MESSAGES`).
- [ ] Documentar el procedimiento de rotación de la admin key.
- [ ] Límite de uso por consumidor (rate limit por consumidor).
- [ ] Decidir si el 403 de las políticas de rol lleva `code: RoleNotAllowed`.
- [ ] Que el handler lea las keys mediante un puerto `IApiKeyRepository` (hoy usa `EmailDbContext`).

## Estado Actual

| Área | Estado |
|------|--------|
| Cabecera única `X-Api-Key` | ✅ |
| Admin: igualdad con el secreto `ADMIN_API_KEY` de OpenBao | ✅ |
| Consumidor: SHA-256 en `keys` (no revocada, no vencida, consumidor activo) | ✅ |
| Claims de rol filtrados por el enum `ConsumerRole` | ✅ |
| `Auth:AdminRoles` con `IOptions` y validación al arrancar | ✅ |
| Políticas `Admin` y una por rol (`Sender`, `GroupSender`, `GroupManager`, `TemplateManager`) | ✅ |
| 401 uniforme en las rutas de gestión | ✅ |
| `GET /whoami` abierto a cualquier key válida | ✅ |
| Rate limiting antes de la autenticación; política de respaldo "autenticado" | ✅ |
| Login del panel contra `/whoami` | ⏳ |
| Límite de uso por consumidor | ⏳ |

## Alcance

Cubre la autenticación y autorización por API key, `whoami` y el login del panel. No cubre la creación de keys y consumidores (Slice 2) ni los permisos sobre recursos concretos (Slices 3 a 6).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]. El handler de API key pasará a leer las keys mediante un puerto `IApiKeyRepository` (en `Application`) con su implementación en `Infrastructure`, en lugar de usar `EmailDbContext` directo; `whoami` se queda en REST.

## Implementación

### Paso 1 — Base de datos
- [x] Tablas `consumer` (con `capabilities` libre) y `keys` (`key_hash` único, `expire_at`, `revoked_at`).
- [ ] Si se agrega límite por consumidor: definir dónde se guarda (columna en `consumer` o configuración).

### Paso 2 — API
- [x] `ApiKeyAuthenticationHandler`, `AuthOptions`/`AuthOptionsValidator`/`SecretsOptions`, políticas en `AuthModule`.
- [x] `GET /whoami` con `{ id, name, roles[] }`.
- [ ] Rate limit por consumidor (partición por id de consumidor tras la autenticación).
- [ ] Definir el `code` del 403 de las políticas de rol.

### Paso 3 — Panel
- [ ] `getMe` → `getWhoAmI` (`/whoami`) en `src/lib/api.ts`.
- [ ] En `login`: si responde 200 pero `roles` no incluye `Admin`, rechazar con "La key no es válida." y no guardar la cookie.
- [ ] En `dashboard.astro`: misma comprobación en cada carga; mostrar nombre y roles.
- [ ] Leer `errors` por campo de los 400 de validación.

### Paso 4 — Documentación
- [ ] README: tabla de roles y permisos al día, rotación de la admin key (`make init FORCE=1` y reiniciar la API).

## Criterios de Aceptación

- [ ] Con la admin key, `/whoami` devuelve `Admin` y los roles de `Auth:AdminRoles`; con la key de un consumidor, solo los suyos.
- [ ] Key falsa, vacía, ausente o revocada → 401; `X-Admin-Key` ya no autentica.
- [ ] Un consumidor en una ruta de gestión recibe el mismo 401 que una key inválida.
- [ ] El panel entra solo con la admin key; con la key de un consumidor muestra "La key no es válida." y no crea cookie.
- [ ] `dotnet build` sin advertencias y `bun run build` correcto. (Los tests los agrega el usuario.)

## Relacionado

- [[email-service-plan]]
- [[email-service-consumidores-keys-sub-plan]]
