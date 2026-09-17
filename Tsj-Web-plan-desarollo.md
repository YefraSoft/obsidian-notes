---
tags:
  - tsj-web
  - plan
  - backend
  - seguridad
  - google-auth
  - roles
created: 2026-09-09
status: en-progreso
proyecto: tsj-web
---
# TSJ-Web — Plan de Desarrollo

> Microservicio **tesj-core** (.NET 10 + PostgreSQL + EF Core)
> Objetivos: Mapear BD existente → Seguridad con Google Auth → Permisos y Roles
> Creado: 2026-09-09

---

## Pendientes

- [x] Ajustar los endpoints por dominio
- [x] Agregar oauth (Authorization Code + PKCE + JWT propio)
- [x] Corregir el verifier
- [x] Configurar credenciales reales de Google (Client ID/Secret) + CORS
- [x] Confirmar/registrar en la consola de Google la redirect URI real del frontend + **rotar secrets** (Client Secret / `JWT_SECRET`) — el `.env` ya fue retirado de git
- [ ] Llenar `GoogleAuth:AllowedRedirectUris` con las URIs reales del frontend (hoy vacío = permite todas)
- [ ] Poblar `users.unidad_academica_id` en el login (la columna ya está en el esquema) + tenancy (Fase 3)
- [ ] Terminar dominio `CustomizationUa` (verifier + services + controller)
- [ ] **Ban de UA (Fase 3)**: columna `disabled` ya en el esquema; definir endpoint ban/unban (soft vía `DeleteAll`) + filtro en GET públicos

## Estado Actual (resumen ejecutivo)

| Área | Estado |
|------|--------|
| Tablas en BD | 16 (definidas en `infra/db.sql`) |
| Tablas cubiertas por backend | 16/16 (100%) — 17 DbSets en `AppDbContext` (16 BD + users) |
| Autenticación real | Sí — login Google (Authorization Code + PKCE) + JWT propio (`AuthSchemes.Jwt`) |
| Roles en código | 4 (`Public`, `CampusManager`, `Director`, `Admin`) |
| RBAC | Matriz implementada + `CanDoService` (Fase 3 parcial) |
| Tabla `users` | Creada con `unidad_academica_id` (columna en `db.sql` + FK; poblar pendiente) |
| CORS | Configurado desde `Cors:AllowedOrigins` (fallback `localhost:3000` + `tecmm.mx`); sin `AllowCredentials` (SPA autentica con JWT) |
| Rate limiting | Parcial (GET pública + `POST /auth/google`; en `Common/Config/RateLimiting/`) |
| Sesión | JWT (HS256) emitido por tsj-core; cookie anónimo solo como fallback de lectura |

| Fase | Estado |
|------|--------|
| FASE 1 — Mapeo de la BD | ✔ Completa |
| FASE 2 — Seguridad con Google OAuth2 | ✔ Completa |
| FASE 3 — Permisos y Roles | 🔄 Parcial |

---

## Decisiones de Diseño (vigentes)

### Modelo de roles
- **4 roles finales**: `Public`, `CampusManager`, `Director`, `Admin`.
- `Authenticated` se elimina como rol/policy: ASP.NET no lo exige, era solo un nombre. Se usan únicamente los 4 declarados en `AppRoles`.
- El rol se almacena en la columna `users.role` (no hay tablas pivote `roles/user_roles`).

### Matriz de permisos (`RolePermissions.cs`)

| Rol             | Permisos                                                                                               |
| --------------- | ------------------------------------------------------------------------------------------------------ |
| `Public`        | `Read`                                                                                                 |
| `CampusManager` | `Read`, `ReadOwn`, `WriteOwn`, `EditOwn`, `DeleteOwn`                                                  |
| `Director`      | `Read`, `ReadAll`, `WriteAll`, `EditAll`, `DeleteAll`, `AuthorizeChanges`                              |
| `Admin`         | `Read`, `ReadAll`, `WriteAll`, `EditAll`, `DeleteAll`, `AuthorizeChanges`, `ManageAccounts`, `DropAll` |

**Familias por recurso (Unidad Académica):**

| Familia | Base (sin recurso) | Own (solo `scope` == UA del recurso) | All (cualquier UA)                       |
| ------- | ------------------ | ------------------------------------ | ---------------------------------------- |
| Read    | `Read` (todos)     | `ReadOwn`                            | `ReadAll`                                |
| Write   | —                  | `WriteOwn`                           | `WriteAll`                               |
| Edit    | —                  | `EditOwn`                            | `EditAll`                                |
| Delete  | —                  | `DeleteOwn` (*cambio de estado*)     | `DeleteAll` (*ban*)                      |
| Drop    | —                  | —                                    | `DropAll` (borrado físico, solo `Admin`) |

> Semántica de borrado: `DeleteAll` = BAN (desactivar contenido vía `unidad_academica.disabled`), `DropAll` = Delete (físico, Admin).

### Autorización (`CanDoService`)
- Permisos **sin recurso** (`AuthorizeChanges`, `ManageAccounts`, `DropAll`): verificación de rol directa.
- Permisos **por recurso** (`Read/Write/Edit/Delete` con familias Own/All): resueltos por `ICanDoService` según el rol y la UA del recurso.
- El *scope* del usuario llega como claim `unidad_academica_scope` (`AuthClaims.UnidadAcademicaScope`); si no existe, los permisos `Own` se deniegan (fail-closed). Lo emite `JwtTokenService` en el JWT cuando el usuario tiene `UnidadAcademicaId` (Fase 2).
- Aplicación actual en `UnidadAcademicaController`:
  - GET / GET{id} → `[Authorize(Policy = AuthPolicies.Public)]` + rate limit `GetPublic`
  - POST (crear UA) → `[Authorize(Policy = AuthPolicies.Director)]` + verifier `Can(User, WriteAll)` (Director/Admin)
  - PUT (editar UA) → `[Authorize(Policy = AuthPolicies.CampusManager)]` + verifier `CanEdit(User, entity.Id)` (CampusManager solo su UA; Director/Admin cualquiera)
  - DELETE (borrar UA) → `[Authorize(Policy = AuthPolicies.Admin)]` + verifier `CanDrop(User)` (solo Admin)
- Las policies por rol (`AuthPolicies.Public|CampusManager|Director|Admin`) siguen registradas en `AuthModule` para Endpoints futuros.

### Sesión y autenticación
- Flujo: **Google OAuth2 (Authorization Code + PKCE)** → validación del `id_token` → **JWT propio (HS256)** emitido por tsj-core. Se actualiza la decisión previa (se descartó la cookie firmada / Opción A).
- Schemes: `AuthSchemes.Default` = cookie anónimo (lectura, rol `Public`); `AuthSchemes.Jwt` = usuarios autenticados.
- Esquema `Public` se mantiene como fallback para lectura (rol-neutral).

### Tenancy (control por unidad académica)
- Un usuario está ligado a **una** unidad académica.
- Modelo: columna `users.unidad_academica_id` (pendiente, Fase 3) + claim `unidad_academica_scope` en la sesión.
- No hay tabla pivote `user_campus`.

---

## Arquitectura y Notas Técnicas

- **Tech stack:** .NET 10, ASP.NET Core Minimal Hosting, EF Core 10 + Npgsql
- **BD:** PostgreSQL (puerto 5432 via Docker)
- **Cache:** Redis (puerto 6379 via Docker) — infra cableada, sin consumidores (rate limiting en memoria)
- **Proxy:** Caddy (TLS termination)
- **Auth actual:** Cookie anónimo no firmado (`AuthHandler`) + JWT propio (`AuthSchemes.Jwt`)
- **Auth objetivo:** Google OAuth2 (PKCE) + RBAC

---

## Archivos Clave del Proyecto

| Archivo | Propósito |
|---------|-----------|
| `infra/db.sql` | Esquema completo de BD (321 líneas; incluye `users` con `unidad_academica_id`) |
| `backedn/tesj-core/Common/Config/RateLimiting/` | Rate limiting en memoria (`GetPublic` + `AuthLogin`) |
| `backedn/tesj-core.Tests/` | Tests unitarios (16 verdes) |
| `backedn/tesj-core/Program.cs` | Arranque y configuración |
| `backedn/tesj-core/Data/AppDbContext.cs` | Contexto EF Core (17 DbSets, cobertura 100%) |
| `backedn/tesj-core/Common/Config/Security/AuthModule.cs` | Config de auth (policies, handlers, `CanDoService`) |
| `backedn/tesj-core/Common/Config/Security/Roles/AppRoles.cs` | Roles definidos (4) |
| `backedn/tesj-core/Common/Config/Security/Permissions/Policies.cs` | Enum de permisos |
| `backedn/tesj-core/Common/Config/Security/Permissions/RolePermissions.cs` | Mapa roles→permisos |
| `backedn/tesj-core/Common/Config/Security/Permissions/CanDoService.cs` | Autorización por recurso (UA) |
| `backedn/tesj-core/Common/Config/Security/Models/AuthClaims.cs` | Const de claims (scope) |
| `backedn/tesj-core/Endpoints/UnidadAcademica/UnidadAcademicaController.cs` | Controller de referencia (patrón CanDo) |
| `backedn/tesj-core/Common/Config/Security/GoogleAuth/*` | Config + exchange (PKCE) + validación id_token + `RegisterGoogleAuthentication` (JWT) |
| `backedn/tesj-core/Common/Config/Security/Jwt/*` | `JwtSettings` y `JwtTokenService` (emisión HS256) |
| `backedn/tesj-core/Common/Interfaces/Services/IUserLookupService.cs` | Lookup de usuarios (Google) |
| `backedn/tesj-core/Endpoints/Auth/AuthController.cs` | `POST /auth/google`, `GET /auth/me` |
| `backedn/tesj-core/Endpoints/CustomizeUa/` | Dominio de customización por UA (DTOs; module en curso) |

---

## FASE 1 — Mapeo de la Base de Datos Existente

> **Objetivo:** Documentar completamente el esquema actual, identificar gaps, y preparar la BD para las fases siguientes.
> **Estado: ✔ Completa** — todo lo listado está hecho.

### 1.1 Auditoría del esquema actual

- [x] Revisar `infra/db.sql` y listar las 16 tablas con sus columnas, tipos y constraints
- [x] Verificar que el `AppDbContext.cs` refleja correctamente las tablas mapeadas
- [x] Identificar las 5 tablas NO mapeadas por el backend (hoy todas tienen modelo EF; 17 DbSets, cobertura 100%):
    - `customization_ua` (banners/FAQs por campus) — endpoints en curso (`Endpoints/CustomizeUa`)
    - `workshops_ua` (talleres por campus)
    - `staff_workshops_ua` (personal de talleres)
    - `courses_ua` (cursos en línea)
    - `extra_curricular_activities` (actividades extracurriculares)
- [x] Verificar integridad referencial: todas las FK apuntan a tablas existentes
- [x] Revisar índices existentes vs. consultas frecuentes

### 1.2 Mapeo de tablas no cubiertas

- [x] Decidir si las 5 tablas sin backend se mapean o se eliminan del esquema
- [x] Si se mapean: crear modelos EF Core (`Data/Models/*.cs`) para cada una
- [x] Si se eliminan: crear migración para limpiar el esquema
- [x] Registrar decisión en este documento

### 1.3 Preparación para autenticación (tabla de usuarios)

- [x] Agregar `DbSet<User>` al `AppDbContext`
- [x] Crear modelo `User.cs` en `Data/Models/`
- [x] Tabla `users` (role, google_id, email, ...) — integrada en `infra/db.sql` (en vez del script `create-users-table.sql` aparte)

### 1.4 Auditoría de datos

- [x] Verificar tipos VARCHAR vs. ENUM: `modality`, `type` (courses), `type` (documents)
- [x] Evaluar agregar campos `created_at` / `updated_at` a tablas existentes
- [x] Revisar `AGENTS.md` rules: ¿cumple la BD con las convenciones del repo?
- [x] Documentar desviaciones y decidir si se alinean

### 1.5 Validación

- [x] Ejecutar el backend y verificar que todas las tablas mapeadas funcionan
- [x] Probar CRUD en las 11 tablas existentes
- [x] Verificar que los seeds (`seed.sql`, `seed-final.sql`) son consistentes con el esquema

---

## FASE 2 — Seguridad con Google OAuth2

> **Objetivo:** Reemplazar el cookie anónimo actual con autenticación real vía Google, manteniendo el esquema público como fallback.
> **Estado: ✔ Completa** — flujo PKCE + JWT implementado y compilando; tests verdes (16); pendientes solo externos (redirect URI en consola, rotación de secrets).

### 2.1 Configuración de Google Cloud Console

- [x] Crear proyecto en Google Cloud Console (si no existe)
- [x] Crear credenciales OAuth 2.0 tipo Web (Client ID + Client Secret)
- [x] Registrar la URI de redirección del frontend (la envía el frontend en el body de `POST /auth/google`)
- [x] No requiere People API: los datos vienen del `id_token` de Google
- [x] Guardar Client ID y Client Secret en variables de entorno (nunca en código)

### 2.2 Paquetes NuGet

- [x] `Microsoft.AspNetCore.Authentication.JwtBearer` (validación del JWT propio)
- [x] Exchange manual (Authorization Code + PKCE) vía `HttpClient` en `GoogleTokenExchangeService`
- [x] Verificar compatibilidad con .NET 10

### 2.3 Configurar Google Auth en el backend

- [x] Crear `GoogleAuthConfig.cs` (ClientId, ClientSecret, dominios permitidos) en `Common/Config/Security/GoogleAuth/`
- [x] Crear `JwtSettings.cs` (SigningKey, Issuer, Audience, ExpirationMinutes) en `Common/Config/Security/Jwt/`
- [x] `RegisterGoogleAuthentication()` en `GoogleAuthentication.cs`: configura opciones, servicios de exchange/validación y `AddJwtBearer(AuthSchemes.Jwt)` (validación local, `MapInboundClaims=false`)
- [x] Agregar configuración en `appsettings.json` / `appsettings.Development.json`: secciones `GoogleAuth` y `JwtSettings` (valores placeholder)
- [x] Mantener esquema `Public` (cookie anónimo) como fallback para endpoints de lectura
- [x] `AuthModule.cs` registra ambos schemes (`Default` + `Jwt`) y las policies por rol los aceptan

### 2.4 Crear controller de autenticación

- [x] Crear `AuthController.cs` en `Endpoints/Auth/`
- [x] `POST /auth/google` → intercambia el código (PKCE), valida el `id_token`, sincroniza el usuario y emite el JWT
- [x] `GET /auth/me` → retorna info del usuario autenticado (`[Authorize(Policy = AuthPolicies.GoogleUser)]`)
- [x] (Opcional) `POST /auth/logout` → el frontend descarta el JWT (204 NoContent)

> Nota: la redirección a Google la ejecuta el frontend (flujo PKCE); el backend no expone `/auth/login` ni `/auth/callback`.

### 2.5 Gestión de sesiones (diseño decidido: JWT propio)

- [x] Decidir estrategia: **JWT propio (HS256) emitido por tsj-core** tras validar el `id_token` de Google (Authorization Code + PKCE). *Actualiza la decisión previa de cookie firmada.*
- [x] Configurar `AddJwtBearer(AuthSchemes.Jwt)` con validación local (issuer, audience, signing key, lifetime)
- [x] Configurar expiración de sesión vía `JwtSettings:ExpirationMinutes`
- [x] Emitir claim `unidad_academica_scope` en el JWT cuando el usuario tiene `UnidadAcademicaId` (base para Fase 3)

### 2.6 Modificar handlers existentes

- [x] `AuthHandler.cs` (cookie anónimo) asigna solo rol `Public`
- [x] No se creó handler propio de Google: se usa el `JwtBearerHandler` nativo con el scheme `AuthSchemes.Jwt`
- [x] `AuthSchemes.cs` agregó `Jwt = "jwt"` (usuarios autenticados)
- [x] `AuthModule.cs` maneja ambos esquemas (`Default` cookie + `Jwt`); las policies por rol aceptan ambos
- [x] Endpoints públicos funcionan sin login

### 2.7 Sincronización de usuario con BD

- [x] `IUserLookupService` / `UserLookupService`: `GetOrCreateGoogleUserAsync` crea/actualiza el registro en `users` al hacer login
- [x] Asignar rol por defecto `Public` a nuevos usuarios
- [x] Almacenar `google_id`, `email`, `display_name`, `avatar_url`
- [ ] (Parcial) `User.cs` ya expone `unidad_academica_id` nullable; falta la migración + FK (Fase 3)

### 2.8 Configuración de CORS

- [x] Configurar política CORS en `Program.cs` (orígenes desde `Cors:AllowedOrigins`):
    ```csharp
    builder.Services.AddCors(options =>
    {
        options.AddPolicy("AllowFrontend", policy =>
        {
            policy.WithOrigins("http://localhost:3000", "https://tecmm.mx")
                  .AllowAnyHeader()
                  .AllowAnyMethod();
            // Sin AllowCredentials: el SPA autentica con JWT (header Authorization), no cookies.
        });
    });
    ```
- [x] Aplicar CORS antes de routing (`app.UseCors("AllowFrontend")` en `Program.cs`, antes de `UseAuthentication`)
- [x] ~~Agregar headers de seguridad~~ — **descartado**: `SecurityHeadersMiddleware` eliminado (sin CSP/X-Frame-Options); la seguridad de transporte se delega a Caddy/TLS.

### 2.9 Variables de entorno

- [x] Agregar `GoogleAuth__ClientId` y `GoogleAuth__ClientSecret` a `compose.yml` (+ `.env.example`)
- [x] Agregar `JwtSettings__SigningKey` (>= 32 chars) a `compose.yml` (+ `.env.example`)
- [x] Retirar `.env` del control de versiones: `git rm --cached` + nuevo `.env.example` con placeholders
- [ ] **Rotar credenciales** en Google Cloud Console (Client Secret expuesto) y regenerar `JWT_SECRET`

### 2.10 Validación

- [x] Tests unitarios (16 verdes): `JwtTokenService` (claims/sub/scope), `GoogleTokenExchangeService` (envía `code_verifier`), `GoogleAuthConfig` (dominios + redirect whitelist), `RequiredValue`
- [ ] Probar flujo completo: frontend obtiene el código (PKCE) → `POST /auth/google` → JWT → `GET /auth/me`
- [ ] Verificar que endpoints públicos siguen funcionando sin auth
- [ ] Verificar que endpoints protegidos rechazan requests sin JWT válido
- [ ] Probar en producción con HTTPS

---

## FASE 3 — Permisos y Roles

> **Objetivo:** RBAC por rol + permiso, con control de acceso por recurso (unidad académica).
> **Estado: 🔄 Parcial** — matriz, roles y `CanDoService` implementados; pendiente tenancy y validación.

### 3.1 Diseño del modelo de roles

- [x] Definir roles finales (los 4 declarados en `AppRoles`):
    | Rol | Descripción |
    |-----|-------------|
    | `Public` | Solo lectura, sin login requerido |
    | `CampusManager` | Gestiona los recursos de SU unidad académica (`*Own`) |
    | `Director` | Gestiona cualquier unidad académica (`*All`) + autoriza cambios |
    | `Admin` | Todo lo de Director + gestiona cuentas y borrado físico (`DropAll`) |
- [x] Documentar permisos por rol (matriz implementada)
- [x] Decidir almacenamiento: columna `users.role` (ya existe en la tabla)

### 3.2 Persistencia de roles y tenancy

- [x] Decidir modelo: **columna `role` en `users`** (Opción A simple) — sin tablas pivote `roles/user_roles`
- [x] Decidir tenancy: 1 usuario → 1 unidad académica → columna `users.unidad_academica_id`
- [x] Columna `unidad_academica_id` en `users` — ya en `infra/db.sql` con FK `ON DELETE SET NULL`
- [ ] Poblar `unidad_academica_id` en el login + seed de usuarios base

### 3.3 Actualizar modelo de roles en backend

- [x] `AppRoles.cs` con los 4 roles finales (`Public`, `CampusManager`, `Director`, `Admin`)
- [x] `Policies.cs` con permisos por familia Own/All + `AuthorizeChanges`, `ManageAccounts`, `DropAll`
- [x] `RolePermissions.cs` con el mapa roles → permisos + `HasPermission`

### 3.4 Authorization Policies

- [x] `AuthPolicies.cs` con policies por rol (`Public`, `CampusManager`, `Director`, `Admin`)
- [x] Registrar policies en `AuthModule.cs` (incluye default policy para endpoints no protegidos)
- [x] Scheme `Public` (cookie) registrado; `AuthHandler` asigna rol `Public`

### 3.5 Autorización por recurso

- [x] Crear `ICanDoService` / `CanDoService`:
    - `Can(user, permission)` → permiso sin recurso
    - `CanRead/CanWrite/CanEdit/CanDelete(user, resourceUaId)` → familias Own/All/base
    - `CanDrop(user)` → solo Admin
- [x] Registrar `ICanDoService` (scoped) en `AuthModule`
- [x] El claim `unidad_academica_scope` se emite en el JWT (`JwtTokenService`) cuando el usuario tiene `UnidadAcademicaId` (columna ya en `db.sql`)
- [ ] Poblar `unidad_academica_id` del usuario en el login

### 3.6 Aplicar autorización por endpoint

- [x] Aplicar patrón CanDo en `UnidadAcademicaController` (POST `WriteAll`, PUT `CanEdit(request.Id)`, DELETE `CanDrop`)
- [ ] Aplicar el mismo patrón a futuros controllers (programas, talleres, cursos, etc.)
- [ ] Considerar helper `[Authorize(Policy = AuthPolicies.*)]` para casos simples de rol

### 3.7 Control de acceso por campus (tenancy)

- [x] Decidir modelo: usuario ligado a una `unidad_academica` (columna `users.unidad_academica_id`)
- [x] El claim viaja en el JWT (`JwtTokenService`); la columna `users.unidad_academica_id` ya existe en el esquema
- [ ] Poblar la columna + prueba end-to-end del alcance (CampusManager 403 en otras UAs)
- [ ] Verificar que un `CampusManager` solo accede a SU unidad académica (rechazo en otras con `Forbid()` → 403)

### 3.8 Seeds y gestión de roles

- [ ] Crear servicio o Endpoints admin para cambiar el rol de un usuario (columna `users.role`)
- [ ] Seed de datos iniciales (unidades académicas, usuarios base)

### 3.9 Validación completa

- [ ] Probar flujo: login Google → usuario creado con rol `Public`
- [ ] Probar asignación de rol: admin cambia rol a `CampusManager` / `Director`
- [ ] Probar control de acceso: `CampusManager` accede a su UA, rechazado en otra (403)
- [ ] Probar que `Admin` / `Director` pasan las familias `*All` y `DropAll`
- [ ] Probar que `Public` solo lee y recibe 403 en escritura
- [ ] Probar rate limiting y logs de accesos no autorizados

---

## Dependencias y Orden de Ejecución

```
FASE 1 (Mapeo BD) — ✔ Completa
    │
    ├── 1.1-1.2: Auditoría y mapeo de tablas
    ├── 1.3: Crear tabla users (prepara para Fase 2/3)
    ├── 1.4: Auditoría de datos
    └── 1.5: Validación
         │
         ▼
FASE 2 (Google Auth) — ✔ Completa (PKCE + JWT; tests 16)
    │
    ├── 2.1-2.2: Configuración Google + paquetes (JwtBearer + exchange manual PKCE)
    ├── 2.3-2.4: Configurar auth + controller (RegisterGoogleAuthentication + /auth/google, /auth/me ✔)
    ├── 2.5-2.6: JWT propio + handlers (Public ✔)
    ├── 2.7: Sincronización con BD ✔
    ├── 2.8-2.9: CORS ✔ + secrets retirados de git (pendiente solo rotar)
    └── 2.10: Validación
         │
         ▼
FASE 3 (Permisos y Roles) — 🔄 Parcial
    │
    ├── 3.1-3.4: Roles + matriz + policies (✔ implementado)
    ├── 3.5: CanDoService (✔ implementado; pendiente scope real)
    ├── 3.6: Patrón CanDo en controllers (UnidadAcademica ✔; resto pendiente)
    ├── 3.7: Tenancy — scope por UA (columna en esquema; falta poblar/validar)
    ├── 3.8: Seeds y gestión de roles
    └── 3.9: Validación completa
```
