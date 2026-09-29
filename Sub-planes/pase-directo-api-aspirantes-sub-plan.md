---
tipo: sub-plan
titulo: "Registro de Aspirantes + EDCORE (Pase Directo v1)"
autor: "Efraín García"
creado: 2026-09-24
actualizado: 2026-09-25
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

# Registro de Aspirantes + EDCORE (SITEM-12)

> API de registro de aspirantes con EDCORE (MariaDB CORE). El flujo vigente separa prevalidación, alta, sesión interna y cambio de procedimiento. Contrato funcional: [[pase-directo-reglas-negocio-registro-aspirantes-manual]].

## Alcance

Implementar en `tesj-core` (base `/tsj-api`) los siguientes endpoints:

- `POST /aspirantes/validar` — prevalidación pública: consulta alumno y aspirante en EDCORE; solo si puede continuar consulta RENAPO y emite token temporal de Redis.
- `POST /aspirantes/registro` — alta pública final con token de prevalidación, sin repetir consultas a EDCORE ni RENAPO.
- `POST /aspirantes/login` — sesión interna de aspirante con CURP y contraseña EDCORE.
- `PATCH /aspirantes/procedimiento` — cambio autenticado entre Nuevo Ingreso y Pase Directo, identificado exclusivamente por el JWT interno.

No forman parte del alcance: `GET /aspirantes/{curp}`, creación de `User` en Postgres para aspirantes, autenticación Google ni `redirectUrl` a un sistema externo.

## Estado Actual (resumen ejecutivo)

- ✅ Setup .NET 10 + Docker (SITEM-10, done) — base del backend.
- ✅ Conector MariaDB, gateway inicial EDCORE y hashing SHA-1 — requieren alineación funcional en Fase 3.5.
- ✅ Integración RENAPO (SITEM-13, done) — el cliente existe; su invocación queda limitada a la prevalidación permitida.
- 🟡 Modelo de datos EDCORE (SITEM-11) — `CapAspirantes` y las consultas de alumno están definidas; falta incorporar `Programs.prog_IdLevel` y cerrar el contrato de reglas.
- Referencia estructural: dominio `Endpoints/UnidadAcademica/` (Controller + Module + Services + Dto + Verifier).

## Decisiones de Diseño

1. **EDCORE como identidad de aspirante.** El aspirante usa CURP + contraseña de `core.CapAspirantes`; no se crea ni sincroniza `User` en Postgres y no interviene Google Auth.
2. **Conector MariaDB vía `MySqlConnector`.** EDCORE se consulta e inserta con SQL directo; no se usa EF Core para esa base externa.
3. **Valores de negocio.** La API acepta `proceso` = `Licenciaturas` o `Posgrados`, y `procedimiento` = `Nuevo Ingreso` o `Pase Directo`. El valor legado `Pase Directo` que existe en el enum de la tabla no es un proceso aceptado por la API.
4. **Clasificación académica.** `Programs.prog_IdLevel = 7` es maestría; cualquier otro nivel corresponde a licenciatura. No se considera estado académico adicional.
5. **Prevalidación antes de RENAPO.** `POST /validar` revisa alumno y aspirante en EDCORE. Solo los casos permitidos consultan RENAPO y reciben un token opaco Redis de un uso, con TTL de 12 minutos; el frontend muestra 10 minutos.
6. **Alta sin reconsulta.** `POST /registro` valida CURP, proceso y procedimiento contra el token, lo consume y registra el aspirante. No vuelve a consultar EDCORE ni RENAPO.
7. **Sesión de aspirante.** La contraseña se compara como SHA-1 hexadecimal en tiempo constante. El JWT interno contiene `aspiranteId` y no reutiliza la identidad de usuarios Google.
8. **Redirección pendiente.** `redirectUrl` al portal de admisiones queda fuera del contrato hasta confirmación de Gabriel.

## Arquitectura y Notas Técnicas

Nuevo dominio `Endpoints/Aspirantes/`:

- `AspirantesController.cs` — validar, registro y login públicos; PATCH protegido por JWT interno y rate limits específicos.
- `AspirantesModule.cs : IModule` — registro DI por reflection (no tocar `Program.cs`).
- `Services/EdcoreConnectionFactory.cs` — `MySqlConnection` desde `ConnectionStrings:Edcore`.
- `Services/IEdcoreGateway` + `AspiranteEdcoreService.cs` — consultas/insert directos:
  - `SELECT ... FROM core.CapAspirantes WHERE curp = @curp`.
  - `SELECT ... FROM core.Users U JOIN core.UserStudents US ON U.user_IdUser = US.user_IdUser JOIN core.Programs P ON US.prog_IdProgram = P.prog_IdProgram WHERE U.curp = @curp`.
  - `INSERT` en `core.CapAspirantes` (proceso/procedimiento según request; `status` = `Interesado`; `contrasena` = SHA1).
- `Services/IRenapoClient` + `RenapoClient.cs` — `GET {BaseUrl}/porCurp/{curp}` únicamente tras una decisión permitida.
- Servicios de prevalidación, registro y sesión — orquestan matriz EDCORE, token Redis, alta y cambio de procedimiento.
- `Dto/` — solicitudes/respuestas de validar, registro, login, procedimiento y aspirante; errores con Problem Details.

### Matriz EDCORE

| Hallazgo EDCORE | Solicitud | Resultado API |
| --- | :---: | --- |
| Alumno y aspirante | Cualquiera | Registrar evento de regla 422; exponer `409` para control escolar. |
| Aspirante con mismo procedimiento | Cualquiera | `200` con `usuarioRegistrado`; dirigir a login o recuperación. |
| Aspirante con otro procedimiento | Nuevo Ingreso ↔ Pase Directo | `422` con procedimiento actual y solicitud de cambio mediante login. |
| Alumno de licenciatura | Posgrados y procedimiento omitido | Permitir prevalidación; el alta asigna Nuevo Ingreso. |
| Cualquier otro alumno | Cualquiera | `200` con `usuarioRegistrado`. |
| Sin alumno ni aspirante | Solicitud válida | Permitir prevalidación. |

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

> La base conserva `Pase Directo` como valor legado de `proceso`; la validación API lo rechaza.

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
}
```

Env vars requeridas en `tsj-web/.env.example` y `compose.yml`: `EDCORE_HOST`, `EDCORE_PORT`, `EDCORE_DB`, `EDCORE_USER`, `EDCORE_PASSWORD`, `RENAPO_HOST`, `RENAPO_KEY`. El flujo de aspirantes no depende de `ADMISIONES_URL`.

## Archivos Clave

- `tesj-core/Endpoints/UnidadAcademica/` — patrón de referencia (Controller, Module, Services, Dto, Verifier).
- `tesj-core/Common/Config/Security/Jwt/JwtTokenService.cs` — emisión de JWT ya implementada.
- `tesj-core/Data/RedisService.cs` — almacenamiento del token de prevalidación.
- `tesj-core/Common/Config/Security/AuthPolicies.cs` + `RateLimitPolicies.cs` — auth y rate limiting.
- `tsj-web/compose.yml` + `tsj-web/.env.example` — inyección de variables (se agregarán las de EDCORE/RENAPO).
- `tsj-web/infra/edcore_db/schema.sql` — avance del esquema EDCORE (parcial, Gabriel).

## FASE 1 — Conector + configuración

- [x] Agregar NuGet `MySqlConnector` a `tesj-core/tesj-core.csproj`.
- [x] Crear `Endpoints/Aspirantes/Services/EdcoreConnectionFactory.cs` (lee `ConnectionStrings:Edcore`).
- [x] Agregar la configuración `Renapo` en `appsettings.json` / `appsettings.Development.json`.
- [x] Añadir variables EDCORE y RENAPO a `tsj-web/.env.example` y `compose.yml`.
- [x] Definir políticas de rate limit para operaciones públicas de aspirantes.
- [x] Retirar `ExternalServices:AdmisionesUrl` y `ADMISIONES_URL` del flujo y contratos de aspirantes; no bloquear el registro por la falta de redirect.

## FASE 2 — Capa EDCORE (MariaDB)

- [x] Crear `IEdcoreGateway` + `AspiranteEdcoreService`:
  - [x] Consultar aspirante por CURP en `core.CapAspirantes`.
  - [x] Consultar alumno por CURP en `core.Users` JOIN `core.UserStudents`.
  - [x] Insertar aspirante en `core.CapAspirantes` (status `Interesado`, `contrasena` SHA1).
- [x] Helper de hashing SHA1 en `Common/Utils/` (opcional).
- [x] Extender la consulta de alumno con `core.Programs` y devolver `prog_IdLevel` para aplicar la clasificación licenciatura/maestría.
- [x] Unificar los mapeos de enums EDCORE en el `EnumMapper` reutilizable y validar los valores permitidos por la API.
- [x] Exponer en el gateway la consulta conjunta alumno/aspirante necesaria para detectar la inconsistencia de una misma CURP en ambos registros.

## FASE 3 — Cliente RENAPO (REST)

- [x] Crear `IRenapoClient` + `RenapoClient`:
  - [x] `GET {BaseUrl}/porCurp/{curp}` con headers `x-renapo-host` / `x-renapo-key`.
  - [x] Timeout configurable (10 s), manejo de errores HTTP → `BusinessException`/`GoogleAuthException` según aplique.
- [x] Limitar su invocación a las decisiones permitidas de la prevalidación; no consultar RENAPO en usuario existente, conflicto o cambio de procedimiento.

## FASE 3.5 — Alineación de fases previas

- [ ] Reemplazar el flujo funcional de alta directa por prevalidación → token Redis → alta final.
- [ ] Retirar del flujo de aspirantes la creación de `User` en Postgres, GoogleId sintético y JWT asociado a Google.
- [ ] Definir el token Redis opaco, de un solo uso y TTL de 12 minutos, ligado a CURP, proceso y procedimiento; contemplar la omisión permitida de procedimiento para Posgrados.
- [ ] Definir el JWT interno con `aspiranteId`, separado de los claims de usuarios Google.
- [ ] Alinear DTOs, servicios y mensajes con la matriz EDCORE y Problem Details (`409` / `422`).
- [ ] Mantener la excepción: alumno de licenciatura (`prog_IdLevel != 7`) que solicita Posgrados sin procedimiento continúa; el alta usa `Nuevo Ingreso`.

## FASE 4 — Prevalidación de CURP

- [ ] Crear DTO `AspiranteValidarRequest`: CURP, proceso y procedimiento; admitir procedimiento omitido solo en la excepción de Posgrados.
- [ ] Crear DTO de respuesta permitida: `puedeRegistrarse`, `tokenValidacion`, `expiraEnSegundos` y datos RENAPO.
- [ ] Implementar `POST /aspirantes/validar` público con rate limit y evaluación EDCORE antes de RENAPO.
- [ ] Responder `409` para alumno + aspirante y registrar internamente el evento de regla 422.
- [ ] Responder `200 usuarioRegistrado` para mismo procedimiento u otro alumno existente.
- [ ] Responder `422` para procedimiento distinto con el texto que pide cambio autenticado.
- [ ] Emitir token Redis únicamente en los casos permitidos y devolver datos RENAPO.

## FASE 5 — Registro final

- [ ] Crear DTO `AspiranteRegistroRequest`: token, datos personales, contacto, contraseña, proceso y procedimiento.
- [ ] Implementar `POST /aspirantes/registro` público con rate limit.
- [ ] Validar token existente, vigente, sin consumir y coincidente con CURP, proceso y procedimiento; rechazar cualquier discrepancia.
- [ ] Resolver `Nuevo Ingreso` por defecto para la excepción Posgrados permitida.
- [ ] Consumir el token e insertar en `CapAspirantes`; no consultar EDCORE, RENAPO ni crear `User` local durante esta operación.
- [ ] Devolver `201` con aspirante y JWT interno.

## FASE 6 — Sesión y cambio de procedimiento

- [ ] Crear `POST /aspirantes/login` con CURP y contraseña.
- [ ] Comparar SHA-1 en tiempo constante y responder `401` genérico ante credenciales inválidas.
- [ ] Emitir JWT interno que contenga únicamente la identidad de aspirante necesaria para autorizar el cambio.
- [ ] Crear `PATCH /aspirantes/procedimiento` protegido por ese JWT.
- [ ] Obtener `aspiranteId` exclusivamente del JWT, validar ambos procedimientos y actualizar solo esa columna.

## FASE 7 — Tests xUnit (`tesj-core.Tests`)

- [ ] Crear dobles de `IEdcoreGateway`, `IRenapoClient` y almacenamiento de token Redis.
- [ ] Cubrir la matriz completa: inconsistencia, aspirante con mismo/diferente procedimiento, alumno de licenciatura a Posgrados, otro alumno y CURP sin registros.
- [ ] Cubrir token vencido, consumido o no coincidente; confirmar que el registro final no reconsulta EDCORE ni RENAPO.
- [ ] Cubrir SHA-1, login, JWT de aspirante y PATCH que no permite modificar a otra CURP.
- [ ] No realizar consultas directas a EDCORE ni consultas CURP/RENAPO pagadas: usar únicamente dobles de prueba.

## FASE 8 — Validación

- [ ] Ejecutar `dotnet build tesj-core.slnx`.
- [ ] Ejecutar `dotnet test tesj-core.Tests` sin EDCORE, RENAPO ni servicios externos.
- [ ] Verificar contratos HTTP, Problem Details y rate limits mediante pruebas locales con dobles.
- [ ] Confirmar con frontend que el contador usa 10 minutos aunque el token expire a los 12 minutos.

## Dependencias y Orden de Ejecución

1. ✅ **Setup .NET 10 + Docker** (SITEM-10) — base.
2. ✅ **Conector EDCORE y cliente RENAPO** (SITEM-11 / SITEM-13) — capacidades técnicas iniciales.
3. 🔄 **Fase 3.5** — alinear esas capacidades con el contrato vigente antes de exponer nuevos endpoints.
4. **Fase 4 y 5** — prevalidación y registro final; SITEM-12.
5. **Fase 6** — sesión interna y cambio de procedimiento; SITEM-14 y SITEM-27.
6. **Frontend** — SITEM-28 consume prevalidación, token, formulario y respuestas; SITEM-29 puede dirigir a recuperación, cuyo backend sigue fuera de este ajuste.
7. **Fases 7 y 8** — pruebas con dobles y validación local; sin llamadas directas a EDCORE o RENAPO.

## Relacionado

- [[Tsj-Web-plan-desarollo]]
- [[pase-directo-reglas-negocio-registro-aspirantes-manual]]
- [[backend-manual]]
