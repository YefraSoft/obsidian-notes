---
tipo: sub-plan
titulo: API CRUD Aspirantes + EDCORE (Pase Directo v1)
autor: Efraín García
creado: 2026-09-24
actualizado: 2026-09-24
estado: en-progreso
proyecto: pase-directo
padre: "[[Tsj-Web-plan-desarollo]]"
tags:
  - pase-directo
  - api-aspirantes
  - edcore
  - renapo
---
> Plan padre: [[Tsj-Web-plan-desarollo]] · Épica Plane: **SITEM-9** · Tarea: **SITEM-12** (deadline 2026-09-30)

# API CRUD Aspirantes + EDCORE (SITEM-12)

> API CRUD de aspirantes con conexión a EDCORE (MariaDB CORE, consulta directa). Endpoints v1: GET consulta de aspirante (por CURP exacta) y POST registro. Sin eliminación/update en esta versión.

## Alcance

Implementar en `tesj-core` (base `/tsj-api`) los endpoints v1:

- `GET /aspirantes/{curp}` — consulta exacta por CURP contra EDCORE: existe como aspirante en `core.CapAspirantes` y/o como alumno (`core.Users` JOIN `core.UserStudents`). Protegido por JWT.
- `POST /aspirantes/registro` — registro público de aspirante. Orquesta el flujo EDCORE primero, luego RENAPO. Devuelve aspirante + JWT + redirect al sistema externo.

## Estado Actual (resumen ejecutivo)

- ✅ Setup .NET 10 + Docker (SITEM-10, done) — base del backend.
- ✅ Integración RENAPO (SITEM-13, done) — REST con credenciales reales.
- 🟡 Modelo de datos EDCORE (SITEM-11, entrega Gabriel) — **parcial**: ya se definieron `core.CapAspirantes` (DDL) y la consulta de alumnos `Users JOIN UserStudents`; resta confirmar el detalle final (columnas exactas del JOIN y `status` inicial al insertar).
- Referencia estructural: dominio `Endpoints/UnidadAcademica/` (Controller + Module + Services + Dto + Verifier).

## Decisiones de Diseño

1. **Conector MariaDB vía `MySqlConnector`** (NuGet) con SQL directo. No EF Core para EDCORE (consulta directa sobre BD externa).
2. **`contrasena` = SHA1 hex** (cabe en `varchar(40)`, legado EDCORE).
3. **Auth por JWT**: en el POST se crea un `User` en Postgres `tesj` (GoogleId sintético `"asp:{id}"`, Email=correo, DisplayName=nombre, Role=`Public`) y se emite JWT con el `JwtTokenService` ya existente. El GET va con `[Authorize(Policy = AuthPolicies.Public)]`.
4. **RENAPO**: cliente REST con `HttpClient`, `GET {BaseUrl}/porCurp/{curp}`, headers `x-renapo-host` / `x-renapo-key` desde config (secretos en `.env`), timeout 10 s; fallo mapeado a `BusinessException`.
5. **Redirect** al portal de admisiones vía `ExternalServices:AdmisionesUrl` (config; valor pendiente de Gabriel).

## Arquitectura y Notas Técnicas

Nuevo dominio `Endpoints/Aspirantes/`:

- `AspirantesController.cs` — `[Route("/aspirantes")]`, `[EnableRateLimiting(...)]` + auth.
- `AspirantesModule.cs : IModule` — registro DI por reflection (no tocar `Program.cs`).
- `Services/EdcoreConnectionFactory.cs` — `MySqlConnection` desde `ConnectionStrings:Edcore`.
- `Services/IEdcoreGateway` + `AspiranteEdcoreService.cs` — consultas/insert directos:
  - `SELECT ... FROM core.CapAspirantes WHERE curp = @curp`.
  - `SELECT ... FROM core.Users U JOIN core.UserStudents US ON U.user_IdUser = US.user_IdUser WHERE U.curp = @curp`.
  - `INSERT` en `core.CapAspirantes` (proceso/procedimiento según request; `status` = `Interesado`; `contrasena` = SHA1).
- `Services/IRenapoClient` + `RenapoClient.cs` — `GET {BaseUrl}/porCurp/{curp}` con headers, timeout y manejo de errores HTTP.
- `Services/RegistroAspiranteService.cs` — orquestador del POST.
- `Dto/` — `AspiranteCreateRequest`, `AspiranteResponse`, `ConsultaAspiranteResponse`, `RegistroAspiranteResponse`.

### Flujo del POST (EDCORE primero, luego RENAPO)

1. Consultar EDCORE por CURP (aspirante en `CapAspirantes` y/o alumno en `Users`).
2. **Si existe** → validar CURP contra RENAPO.
   - Válido → `200 { existe: true, tipo: "aspirante"|"alumno", redirectUrl, login: true }` (el front regresa al Login; no se crea usuario ni se registra).
   - Inválido → 422.
3. **Si no existe** → `INSERT` en `CapAspirantes` → crear `User` en Postgres `tesj` → emitir JWT.
4. `201 { aspirante, jwt, redirectUrl }`.

### Tabla `core.CapAspirantes` (MariaDB)

```sql
CREATE TABLE `CapAspirantes` (
  `cap_IdAspirante` int(11) NOT NULL AUTO_INCREMENT,
  `curp` char(18) NOT NULL,
  `nombre` varchar(100) NOT NULL,
  `primer_apellido` varchar(25) NOT NULL,
  `segundo_apellido` varchar(25) DEFAULT NULL,
  `fecha_nacimiento` date DEFAULT NULL,
  `lugar_nacimiento` varchar(100) DEFAULT NULL,
  `sexo` enum('H','M') DEFAULT NULL,
  `correo` varchar(200) NOT NULL,
  `celular` varchar(15) NOT NULL,
  `proceso` enum('Licenciaturas','Posgrados','Pase Directo') NOT NULL DEFAULT 'Licenciaturas',
  `procedimiento` enum('Nuevo Ingreso','Pase Directo') NOT NULL DEFAULT 'Nuevo Ingreso',
  `contrasena` varchar(40) DEFAULT NULL,
  `status` enum('Interesado','Aspirante','Alumno') NOT NULL DEFAULT 'Interesado',
  PRIMARY KEY (`cap_IdAspirante`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### Config nueva

`appsettings*.json` + `.env`/compose (secretos nunca en código):

```jsonc
"ConnectionStrings": {
  "Edcore": "Server={EDCORE_HOST};Port={EDCORE_PORT};Database=core;User={EDCORE_USER};Password={EDCORE_PASSWORD}"
},
"Renapo": {
  "BaseUrl": "https://curp-mexico1.p.rapidapi.com",
  "Path": "/porCurp/{curp}",
  "Host": "",
  "ApiKey": "",
  "TimeoutSeconds": 10
},
"ExternalServices": {
  "AdmisionesUrl": ""
}
```

Env vars a añadir en `tsj-web/.env.example` y `compose.yml`: `EDCORE_HOST`, `EDCORE_PORT`, `EDCORE_DB`, `EDCORE_USER`, `EDCORE_PASSWORD`, `RENAPO_HOST`, `RENAPO_KEY`, `ADMISIONES_URL`.

## Archivos Clave

- `tesj-core/Endpoints/UnidadAcademica/` — patrón de referencia (Controller, Module, Services, Dto, Verifier).
- `tesj-core/Common/Config/Security/Jwt/JwtTokenService.cs` — emisión de JWT ya implementada.
- `tesj-core/Data/Models/User.cs` — entidad `User` (Postgres `tesj`) a crear en el registro.
- `tesj-core/Common/Config/Security/AuthPolicies.cs` + `RateLimitPolicies.cs` — auth y rate limiting.
- `tsj-web/compose.yml` + `tsj-web/.env.example` — inyección de variables (se agregarán las de EDCORE/RENAPO).
- `tsj-web/infra/edcore_db/schema.sql` — avance del esquema EDCORE (parcial, Gabriel).

## FASE 1 — Conector + configuración

- [x] Agregar NuGet `MySqlConnector` a `tesj-core/tesj-core.csproj`.
- [x] Crear `Endpoints/Aspirantes/Services/EdcoreConnectionFactory.cs` (lee `ConnectionStrings:Edcore`).
- [x] Agregar secciones `Renapo` y `ExternalServices:AdmisionesUrl` en `appsettings.json` / `appsettings.Development.json`.
- [x] Añadir variables a `tsj-web/.env.example` y `compose.yml`.
- [x] Definir política de rate limit para el POST de registro (reusar `RateLimitPolicies.Write` u otra según convención).

## FASE 2 — Capa EDCORE (MariaDB)

- [x] Crear `IEdcoreGateway` + `AspiranteEdcoreService`:
  - [x] Consultar aspirante por CURP en `core.CapAspirantes`.
  - [x] Consultar alumno por CURP en `core.Users` JOIN `core.UserStudents`.
  - [x] Insertar aspirante en `core.CapAspirantes` (status `Interesado`, `contrasena` SHA1).
- [x] Helper de hashing SHA1 en `Common/Utils/` (opcional).

## FASE 3 — Cliente RENAPO (REST)

- [x] Crear `IRenapoClient` + `RenapoClient`:
  - [x] `GET {BaseUrl}/porCurp/{curp}` con headers `x-renapo-host` / `x-renapo-key`.
  - [x] Timeout configurable (10 s), manejo de errores HTTP → `BusinessException`/`GoogleAuthException` según aplique.

## FASE 4 — GET consulta por CURP

- [ ] DTOs: `ConsultaAspiranteResponse` (existe, tipo, datos del aspirante/alumno), `AspiranteResponse`.
- [ ] Controller `GET /aspirantes/{curp}` con `[Authorize(Policy = AuthPolicies.Public)]` + rate limit.
- [ ] Servicio de consulta sobre `IEdcoreGateway`. Cache Redis opcional (clave `aspirante:curp:{curp}`, TTL corto).

## FASE 5 — POST registro

- [ ] DTO request: `curp`, `nombre`, `primer_apellido`, `segundo_apellido`, `fecha_nacimiento`, `lugar_nacimiento`, `sexo`, `correo`, `celular`, `contrasena`, `proceso`, `procedimiento`.
- [ ] `RegistroAspiranteService` orquesta: EDCORE → RENAPO → insert → crear `User` (Postgres) + JWT → `RegistroAspiranteResponse { aspirante, jwt, redirectUrl }`.
- [ ] Controller `POST /aspirantes/registro` (público + rate limit). Validación de CURP (regex en `RegexDictionaries`).

## FASE 6 — Tests xUnit (`tesj-core.Tests`)

- [ ] Dobles de `IEdcoreGateway` y `IRenapoClient`.
- [ ] Casos: existe como aspirante/alumno → valida RENAPO y responde `login:true`; no existe → insert + JWT; RENAPO rechaza → 422; validación de CURP; SHA1.

## FASE 7 — Validación

- [ ] `dotnet build tesj-core.slnx`.
- [ ] `dotnet test tesj-core.Tests` (EF InMemory, sin BD).
- [ ] Prueba manual con curl contra la MariaDB externa usando `.env` (GET por CURP existente y POST de nuevo aspirante).

## Dependencias y Orden de Ejecución

1. ✅ **Setup .NET 10 + Docker** (SITEM-10) — base.
2. ✅ **Integración RENAPO** (SITEM-13) — contrato REST + credenciales.
3. 🟡 **Modelo de datos EDCORE** (SITEM-11, entrega Gabriel) — `CapAspirantes` + `Users`/`UserStudents` conocidos; confirmar `status` inicial y detalle del JOIN.
4. 🔗 Frontend: **SITEM-28 Formulario de Registro** consume el POST; **SITEM-27 Login** consumirá el JWT posteriormente (SITEM-14 Auth JWT).

## Relacionado

- [[Tsj-Web-plan-desarollo]]
- [[backend-manual]]
