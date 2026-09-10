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

## Estado Actual (resumen ejecutivo)

| Área | Estado |
|------|--------|
| Tablas en BD | 16 (definidas en `infra/db.sql`) |
| Tablas cubiertas por backend | 11 (68.75%) |
| Autenticación real | No existe — solo cookie anónimo falsificable (`PublicAuthHandler`) |
| Roles en código | 4 (`Public`, `CampusManager`, `Director`, `Admin`) |
| RBAC | Matriz implementada + `CanDoService` (Fase 3 parcial) |
| Tabla `users` | Creada (sin `unidad_academica_id` aún) |
| CORS | No configurado |
| Rate limiting | Parcial (solo GET) |
| Sesión | Cookie sin firmar (pendiente Google + firma) |

| Fase | Estado |
|------|--------|
| FASE 1 — Mapeo de la BD | ✔ Completa |
| FASE 2 — Seguridad con Google OAuth2 | ⏳ Pendiente |
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

| Familia | Base (sin recurso) | Own (solo `scope` == UA del recurso) | All (cualquier UA) |
|---------|--------------------|--------------------------------------|--------------------|
| Read | `Read` (todos) | `ReadOwn` | `ReadAll` |
| Write | — | `WriteOwn` | `WriteAll` |
| Edit | — | `EditOwn` | `EditAll` |
| Delete | — | `DeleteOwn` (*cambio de estado*) | `DeleteAll` (*ban*) |
| Drop | — | — | `DropAll` (borrado físico, solo `Admin`) |

> Semántica de borrado: `DeleteAll` = BAN (desactivar contenido), `DropAll` = Delete (físico, Admin).

### Autorización (`CanDoService`)
- Permisos **sin recurso** (`AuthorizeChanges`, `ManageAccounts`, `DropAll`): verificación de rol directa.
- Permisos **por recurso** (`Read/Write/Edit/Delete` con familias Own/All): resueltos por `ICanDoService` según el rol y la UA del recurso.
- El *scope* del usuario llega como claim `unidad_academica_scope` (`AuthClaims.UnidadAcademicaScope`); si no existe, los permisos `Own` se deniegan (fail-closed). Lo poblará el handler de sesión de la Fase 2.
- Aplicación actual en `UnidadAcademicaController`:
  - GET / GET{id} → `[Authorize(Policy = AuthPolicies.Public)]`
  - POST (crear UA) → `Can(User, Policies.WriteAll)` (Director/Admin)
  - PUT (editar UA) → `CanEdit(User, request.Id)` (CampusManager solo su UA; Director/Admin cualquiera)
  - DELETE (borrar UA) → `CanDrop(User)` (solo Admin)
- Las policies por rol (`AuthPolicies.Public|CampusManager|Director|Admin`) siguen registradas en `AuthModule` para Endpoints futuros.

### Sesión y autenticación
- Flujo: **Google OAuth2** con cookie firmada ASP.NET (**Opción A**). Se descarta JWT/refresh.
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
- **Auth actual:** Cookie anónimo no firmado (`PublicAuthHandler`)
- **Auth objetivo:** Google OAuth2 + RBAC

---

## Archivos Clave del Proyecto

| Archivo | Propósito |
|---------|-----------|
| `infra/db.sql` | Esquema completo de BD (276 líneas) |
| `infra/create-users-table.sql` | Tabla `users` (role, google_id, email) |
| `backedn/tesj-core/Program.cs` | Arranque y configuración |
| `backedn/tesj-core/Data/AppDbContext.cs` | Contexto EF Core (11 DbSets) |
| `backedn/tesj-core/Common/Config/Security/AuthModule.cs` | Config de auth (policies, handlers, `CanDoService`) |
| `backedn/tesj-core/Common/Config/Security/Roles/AppRoles.cs` | Roles definidos (4) |
| `backedn/tesj-core/Common/Config/Security/Roles/Policies.cs` | Enum de permisos |
| `backedn/tesj-core/Common/Config/Security/Roles/RolePermissions.cs` | Mapa roles→permisos |
| `backedn/tesj-core/Common/Config/Security/Roles/CanDoService.cs` | Autorización por recurso (UA) |
| `backedn/tesj-core/Common/Config/Security/Models/AuthClaims.cs` | Const de claims (scope) |
| `backedn/tesj-core/Endpoints/UnidadAcademica/UnidadAcademicaController.cs` | Controller de referencia (patrón CanDo) |

---

## FASE 1 — Mapeo de la Base de Datos Existente

> **Objetivo:** Documentar completamente el esquema actual, identificar gaps, y preparar la BD para las fases siguientes.
> **Estado: ✔ Completa** — todo lo listado está hecho.

### 1.1 Auditoría del esquema actual

- [x] Revisar `infra/db.sql` y listar las 16 tablas con sus columnas, tipos y constraints
- [x] Verificar que el `AppDbContext.cs` refleja correctamente las 11 tablas mapeadas
- [x] Identificar las 5 tablas NO mapeadas por el backend:
    - `customization_ua` (banners/FAQs por campus)
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
- [x] Crear script `infra/create-users-table.sql` (role, google_id, email)

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
> **Estado: ⏳ Pendiente.**

### 2.1 Configuración de Google Cloud Console

- [ ] Crear proyecto en Google Cloud Console (si no existe)
- [ ] Habilitar People API
- [ ] Crear credenciales OAuth 2.0 (Client ID + Client Secret)
- [ ] Configurar URIs de redirección autorizados:
    - `https://localhost:5001/signin-google` (desarrollo)
    - `https://<dominio-produccion>/signin-google` (producción)
- [ ] Guardar Client ID y Client Secret en variables de entorno (nunca en código)

### 2.2 Paquetes NuGet

- [ ] Instalar `Microsoft.AspNetCore.Authentication.Google`
- [ ] Verificar compatibilidad con .NET 10

### 2.3 Configurar Google Auth en el backend

- [ ] Agregar esquema Google en `AuthModule.cs`:
    ```csharp
    .AddGoogle(options =>
    {
        options.ClientId = config["GoogleAuth:ClientId"];
        options.ClientSecret = config["GoogleAuth:ClientSecret"];
        options.CallbackPath = "/signin-google";
    });
    ```
- [ ] Crear `GoogleAuthOptions.cs` en `Common/Config/Security/Models/`
- [ ] Agregar configuración en `appsettings.json`:
    ```json
    "GoogleAuth": {
        "ClientId": "",
        "ClientSecret": ""
    }
    ```
- [ ] Mantener esquema `Public` como fallback para endpoints de lectura
- [ ] Configurar esquema Google como scheme de sesión (`trust`/`session`)

### 2.4 Crear controller de autenticación

- [ ] Crear `AuthController.cs` en `Endpoints/Auth/`
- [ ] Implementar endpoint `GET /auth/login` → redirige a Google
- [ ] Implementar endpoint `GET /auth/callback` → procesa respuesta de Google
- [ ] Implementar endpoint `GET /auth/me` → retorna info del usuario autenticado
- [ ] Implementar endpoint `POST /auth/logout` → cierra sesión
- [ ] Implementar endpoint `GET /auth/external-login` → Challenge con Google

### 2.5 Gestión de sesiones (diseño decidido: cookie firmada)

- [x] Decidir estrategia: **Cookie firmada ASP.NET (Opción A)** — se descarta JWT/refresh
- [ ] Configurar cookie de autenticación firmada en `AuthModule.cs` (CookieAuthenticationOptions)
- [ ] Configurar expiración de sesión
- [ ] Emitir claim `unidad_academica_scope` en el principal (base para Fase 3)

### 2.6 Modificar handlers existentes

- [x] `PublicAuthHandler.cs` ya asigna solo rol `Public`
- [ ] Crear `GoogleAuthHandler.cs` o usar handler nativo de ASP.NET
- [ ] Modificar `AuthSchemes.cs` para agregar scheme Google: `public const string Google = "Google";`
- [ ] Actualizar `AuthModule.cs` para manejar ambos esquemas (trust/session/google)
- [ ] Asegurar que endpoints públicos funcionan sin login

### 2.7 Sincronización de usuario con BD

- [ ] Al hacer login con Google, crear/actualizar registro en tabla `users`
- [ ] Asignar rol por defecto `Public` a nuevos usuarios
- [ ] Almacenar `google_id`, `email`, `display_name`, `avatar_url`
- [ ] Implementar servicio `UserService.cs` para operaciones CRUD de usuarios

### 2.8 Configuración de CORS

- [ ] Configurar política CORS en `Program.cs`:
    ```csharp
    builder.Services.AddCors(options =>
    {
        options.AddPolicy("AllowFrontend", policy =>
        {
            policy.WithOrigins("http://localhost:3000", "https://tu-dominio.com")
                  .AllowAnyHeader()
                  .AllowAnyMethod()
                  .AllowCredentials();
        });
    });
    ```
- [ ] Aplicar CORS antes de routing
- [ ] Agregar headers de seguridad (X-Content-Type-Options, X-Frame-Options, etc.)

### 2.9 Variables de entorno

- [ ] Agregar `GoogleAuth__ClientId` y `GoogleAuth__ClientSecret` a `compose.yml`
- [ ] Agregar a `.env.production`
- [ ] Verificar que no se commitean secrets

### 2.10 Validación

- [ ] Probar flujo completo: login → callback → sesión activa → logout
- [ ] Verificar que endpoints públicos siguen funcionando sin auth
- [ ] Verificar que endpoints protegidos rechazan requests sin sesión
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
- [ ] Agregar columna `unidad_academica_id` a `users` (migración + FK)
- [ ] Seed inicial: creación de unidades académicas y usuarios base

### 3.3 Actualizar modelo de roles en backend

- [x] `AppRoles.cs` con los 4 roles finales (`Public`, `CampusManager`, `Director`, `Admin`)
- [x] `Policies.cs` con permisos por familia Own/All + `AuthorizeChanges`, `ManageAccounts`, `DropAll`
- [x] `RolePermissions.cs` con el mapa roles → permisos + `HasPermission`

### 3.4 Authorization Policies

- [x] `AuthPolicies.cs` con policies por rol (`Public`, `CampusManager`, `Director`, `Admin`)
- [x] Registrar policies en `AuthModule.cs` (incluye default policy para endpoints no protegidos)
- [x] Scheme `Public` registrado; `PublicAuthHandler` asigna rol `Public`

### 3.5 Autorización por recurso

- [x] Crear `ICanDoService` / `CanDoService`:
    - `Can(user, permission)` → permiso sin recurso
    - `CanRead/CanWrite/CanEdit/CanDelete(user, resourceUaId)` → familias Own/All/base
    - `CanDrop(user)` → solo Admin
- [x] Registrar `ICanDoService` (scoped) en `AuthModule`
- [ ] Resolver el scope desde `users.unidad_academica_id` en la sesión (claim `unidad_academica_scope`)

### 3.6 Aplicar autorización por endpoint

- [x] Aplicar patrón CanDo en `UnidadAcademicaController` (POST `WriteAll`, PUT `CanEdit(request.Id)`, DELETE `CanDrop`)
- [ ] Aplicar el mismo patrón a futuros controllers (programas, talleres, cursos, etc.)
- [ ] Considerar helper `[Authorize(Policy = AuthPolicies.*)]` para casos simples de rol

### 3.7 Control de acceso por campus (tenancy)

- [x] Decidir modelo: usuario ligado a una `unidad_academica` (columna `users.unidad_academica_id`)
- [ ] Implementar el claim `unidad_academica_scope` en el handler de sesión
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
FASE 2 (Google Auth) — ⏳ Pendiente
    │
    ├── 2.1-2.2: Configuración Google + paquetes
    ├── 2.3-2.4: Configurar auth + controller
    ├── 2.5-2.6: Configurar cookie firmada + handlers (PublicAuthHandler ✔)
    ├── 2.7: Sincronización con BD
    ├── 2.8-2.9: CORS + variables
    └── 2.10: Validación
         │
         ▼
FASE 3 (Permisos y Roles) — 🔄 Parcial
    │
    ├── 3.1-3.4: Roles + matriz + policies (✔ implementado)
    ├── 3.5: CanDoService (✔ implementado; pendiente scope real)
    ├── 3.6: Patrón CanDo en controllers (UnidadAcademica ✔; resto pendiente)
    ├── 3.7: Tenancy — scope por UA (decidido; pendiente implementación)
    ├── 3.8: Seeds y gestión de roles
    └── 3.9: Validación completa
```
