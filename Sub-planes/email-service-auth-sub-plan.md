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
> Que la API reconozca a quien llama por su `X-Api-Key`: **Admin**, **Consumer** (app) o **Manager** (persona con email), con los roles de su dominio, y que el panel entre solo con la key de administrador.

## Pendientes

- [x] Migrar el login del panel a `GET /whoami` y exigir `Admin` (`auth.login` y `middleware.ts`).
- [x] Leer los `errors` de validación (400) en el cliente del panel.
- [x] Mostrar el nombre del administrador en la barra del panel.
- [ ] Documentar el procedimiento de rotación de la admin key y de la clave de cifrado (solo en la wiki).
- [ ] Límite de uso por consumidor (rate limit por consumidor).
- [ ] Decidir si el 403 de las políticas de rol lleva `code: RoleNotAllowed`.

## Estado Actual

| Área | Estado |
|------|--------|
| Cabecera única `X-Api-Key` | ✅ |
| Admin: igualdad con el secreto de OpenBao | ✅ |
| Consumidor: SHA-256 en `keys` | ✅ (se reemplaza por la key cifrada con su dominio) |
| Key cifrada (AES-256-GCM) con dominio + ids; una consulta a la tabla del dominio | ✅ |
| Dominios `consumer` y `manager` con sus tablas de keys | ✅ |
| Roles por dominio sin mezcla (Consumer: `Sender`, `GroupSender`; Manager: `GroupManager`, `TemplateManager`) | ✅ |
| `Auth:AdminRoles` con `IOptions` y validación al arrancar | ✅ |
| Políticas `Admin` y una por rol | ✅ |
| 401 uniforme en las rutas de gestión | ✅ |
| `GET /whoami` abierto a cualquier key válida (con `kind`) | ✅ |
| Handler por el puerto `IApiKeyRepository` | ✅ |
| Rate limiting antes de la autenticación; política de respaldo "autenticado" | ✅ |
| Login del panel contra `/whoami` (solo Admin) | ✅ |
| Límite de uso por consumidor | ⏳ |

## Alcance

Cubre la autenticación y autorización por API key, `whoami` y el login del panel. No cubre la creación de dueños y keys (Slice 2) ni los permisos sobre recursos concretos (Slices 3 a 6).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]. El handler prueba primero la admin key (igualdad, sin base); si no, descifra la key con `IKeyProtector` (si está alterada, 401 sin tocar la base), consulta **una vez** la tabla de keys del dominio por el puerto `IApiKeyRepository` y emite los claims con el dominio y solo los roles válidos de ese dominio. `whoami` se queda en REST.

## Implementación

### Paso 1 — Base de datos
- [x] `consumer_keys` y `manager_keys` con índice único parcial (una key vigente por dueño).

### Paso 2 — API
- [x] `ApiKeyAuthenticationHandler`, `AuthOptions`/`AuthOptionsValidator`/`SecretsOptions`, políticas en `AuthModule`.
- [x] `GET /whoami` con `{ id, name, roles[] }`.
- [x] `IKeyProtector` (AES-256-GCM), claim de dominio y `kind` en `whoami`.
- [ ] Rate limit por consumidor (partición por id tras la autenticación).
- [ ] Definir el `code` del 403 de las políticas de rol.

### Paso 3 — Panel
- [x] `SessionService.whoami` en el cliente de la API (`/whoami`).
- [x] En `auth.login`: una key que no sea de Admin se rechaza (403, "Esta cuenta no tiene acceso al panel de administración."; key inválida, 401) y no se guarda la cookie.
- [x] `middleware.ts`: misma comprobación en cada página protegida; sesión caducada → `/?notice=expired`.
- [x] Leer `errors` por campo de los 400 de validación (`parseActionError`).

### Paso 4 — Documentación
- [ ] README y wiki: roles y permisos al día; rotación de la admin key (`make init FORCE=1` y reiniciar la API).

## Criterios de Aceptación

- [ ] Con la admin key, `/whoami` devuelve `Admin` y los roles de `Auth:AdminRoles`; con la key de un consumidor o manager, solo los suyos y su `kind`.
- [ ] Key falsa, alterada, vacía, ausente, rotada o revocada → 401.
- [ ] Una key de otro dominio en una ruta de gestión recibe el mismo 401 que una key inválida.
- [x] El panel entra solo con la admin key; con otra key no crea cookie (401 key inválida; 403 key de consumidor o manager).
- [ ] `dotnet build` sin advertencias y `bun run build` correcto. (Los tests los agrega el usuario.)

## Relacionado

- [[email-service-plan]]
- [[email-service-consumidores-keys-sub-plan]]
