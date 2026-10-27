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
> Que la API reconozca a quien llama por su `X-Api-Key`: **Admin**, **Consumer** (app) o **Manager** (persona con email), con los roles de su dominio, y que el panel entre solo con la key de administrador. La separación de dominios se construye en [[email-service-fase0-temporal]].

## Pendientes

- [ ] Fase 0 temporal: dominios Consumer y Manager, key cifrada con su dominio y una key vigente por dueño ([[email-service-fase0-temporal]]).
- [ ] Migrar el login del panel a `GET /whoami` y exigir el rol `Admin` en la respuesta.
- [ ] Leer los `errors` de validación (400) en el cliente del panel.
- [ ] Mostrar el tipo (`kind`) y los roles en el dashboard.
- [ ] Documentar el procedimiento de rotación de la admin key y de la clave de cifrado (solo en la wiki).
- [ ] Límite de uso por consumidor (rate limit por consumidor).
- [ ] Decidir si el 403 de las políticas de rol lleva `code: RoleNotAllowed`.

## Estado Actual

| Área | Estado |
|------|--------|
| Cabecera única `X-Api-Key` | ✅ |
| Admin: igualdad con el secreto de OpenBao | ✅ |
| Consumidor: SHA-256 en `keys` | ✅ (se reemplaza por la key cifrada con su dominio) |
| Key cifrada (AES-256-GCM) con dominio + ids; una consulta a la tabla del dominio | ⏳ Fase 0 temporal |
| Dominios `consumer` y `manager` con sus tablas de keys | ⏳ Fase 0 temporal |
| Roles por dominio sin mezcla (Consumer: `Sender`, `GroupSender`; Manager: `GroupManager`, `TemplateManager`) | ⏳ Fase 0 temporal |
| `Auth:AdminRoles` con `IOptions` y validación al arrancar | ✅ |
| Políticas `Admin` y una por rol | ✅ |
| 401 uniforme en las rutas de gestión | ✅ |
| `GET /whoami` abierto a cualquier key válida (con `kind`) | ✅ (`kind` llega en la Fase 0 temporal) |
| Handler por el puerto `IApiKeyRepository` | ✅ |
| Rate limiting antes de la autenticación; política de respaldo "autenticado" | ✅ |
| Login del panel contra `/whoami` | ⏳ |
| Límite de uso por consumidor | ⏳ |

## Alcance

Cubre la autenticación y autorización por API key, `whoami` y el login del panel. No cubre la creación de dueños y keys (Slice 2) ni los permisos sobre recursos concretos (Slices 3 a 6).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]. El handler prueba primero la admin key (igualdad, sin base); si no, descifra la key con `IKeyProtector` (si está alterada, 401 sin tocar la base), consulta **una vez** la tabla de keys del dominio por el puerto `IApiKeyRepository` y emite los claims con el dominio y solo los roles válidos de ese dominio. `whoami` se queda en REST.

## Implementación

### Paso 1 — Base de datos
- [ ] `consumer_keys` y `manager_keys` con índice único parcial (una key vigente por dueño); ver [[email-service-fase0-temporal]].

### Paso 2 — API
- [x] `ApiKeyAuthenticationHandler`, `AuthOptions`/`AuthOptionsValidator`/`SecretsOptions`, políticas en `AuthModule`.
- [x] `GET /whoami` con `{ id, name, roles[] }`.
- [ ] `IKeyProtector` (AES-256-GCM), claim de dominio y `kind` en `whoami`.
- [ ] Rate limit por consumidor (partición por id tras la autenticación).
- [ ] Definir el `code` del 403 de las políticas de rol.

### Paso 3 — Panel
- [ ] `getMe` → `getWhoAmI` (`/whoami`) en el cliente de la API.
- [ ] En `login`: si responde 200 pero `roles` no incluye `Admin`, rechazar con "La key no es válida." y no guardar la cookie.
- [ ] En `dashboard.astro`: misma comprobación en cada carga; mostrar nombre, tipo y roles.
- [ ] Leer `errors` por campo de los 400 de validación.

### Paso 4 — Documentación
- [ ] README y wiki: roles y permisos al día; rotación de la admin key (`make init FORCE=1` y reiniciar la API).

## Criterios de Aceptación

- [ ] Con la admin key, `/whoami` devuelve `Admin` y los roles de `Auth:AdminRoles`; con la key de un consumidor o manager, solo los suyos y su `kind`.
- [ ] Key falsa, alterada, vacía, ausente, rotada o revocada → 401.
- [ ] Una key de otro dominio en una ruta de gestión recibe el mismo 401 que una key inválida.
- [ ] El panel entra solo con la admin key; con otra key muestra "La key no es válida." y no crea cookie.
- [ ] `dotnet build` sin advertencias y `bun run build` correcto. (Los tests los agrega el usuario.)

## Relacionado

- [[email-service-plan]]
- [[email-service-fase0-temporal]]
- [[email-service-consumidores-keys-sub-plan]]
