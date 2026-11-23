---
tipo: sub-plan
titulo: "Fase 1 — Fundaciones técnicas y modelo de datos"
autor: Efraín García
creado: 2026-09-11
actualizado: 2026-11-20
estado: completado
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - fase-1
  - backend
  - fundaciones
---

# Fase 1 — Fundaciones técnicas y modelo de datos

> **Plan padre:** [[rh-cotla-plan]]
> Deja listos el entorno Docker, el esqueleto de `cotla-api`, el modelo de datos con migraciones y los seeds que usan las demás fases. Es una fase solo de backend.

## Pendientes

- [x] Infraestructura de desarrollo en `docker-compose.yml`: `api`, `postgres` y `minio` con healthchecks y volúmenes.
- [x] Backend base: `GET /health`, errores RFC 7807, logging JSON, contratos validados.
- [x] Modelo de datos con EF Core y migración inicial versionada.
- [x] Seeds de roles, tienda, departamentos, puestos, horarios y equipos.
- [x] Validación: `dotnet build` sin errores, contenedores healthy, migraciones en base limpia.
- [ ] `HEALTHCHECK` en el Dockerfile del API y fijar el tag de MinIO, hoy `latest` (**FX-BE-4**, ver [[rh-cotla-pendientes-sub-plan]]: registro de toda la deuda con sus ids).
- [ ] CORS cerrado por origen: hoy es `AllowAnyOrigin`, deuda aceptada (A1 en [[rh-cotla-pendientes-sub-plan]]).

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | ✅ Completa. Compose con `api`, `postgres`, `minio`; `/health` responde `Healthy` (Postgres y almacenamiento); build sin errores ni advertencias |
| Panel web | — (esta fase no tiene pantallas) |
| App móvil | — (esta fase no tiene pantallas) |
| Modelo de datos | ✅ Entidades base y de operación, migración inicial y restricciones de unicidad (CURP, teléfono, identificador de usuario) |
| Seeds | ✅ Idempotentes y solo sobre tablas vacías; el RH inicial exige credenciales en Production |

## Alcance

Cubre la infraestructura de desarrollo, el backend base, el modelo de datos y los seeds. No cubre autenticación ni endpoints de negocio ([[rh-cotla-fase-02-autenticacion-colaboradores-sub-plan]] en adelante), ni el despliegue en producción ([[rh-cotla-fase-12-despliegue-sub-plan]]), ni CI/CD y backups, que quedan en el backlog de la versión 2.

## Decisiones

1. **Una sucursal operativa, `sucursal_id` en todo registro.** Se toma siempre del token y nunca del cuerpo de la petición.
2. **Archivos en almacenamiento S3.** MinIO en desarrollo con volúmenes `minio-data` y `minio-certs`; la base guarda solo la clave y la URL, nunca el binario. Credenciales por `MINIO_ROOT_USER` y `MINIO_ROOT_PASSWORD` (mínimo 8 caracteres) en `.env.example`.
3. **Migraciones EF Core como único mecanismo de cambios al esquema** (`dotnet ef database update`); `infra/db.sql` se mantiene sincronizado.
4. **Errores como `ProblemDetails` (RFC 7807)** mediante middleware global y contratos con DataAnnotations.
5. **Seed de la tienda `Ferretería Cotla`:** ubicación desde `SEED_SUCURSAL_LATITUD` y `SEED_SUCURSAL_LONGITUD` (0,0 si faltan); radio 100 m y tolerancia 5 min son valores fijos de la semilla.
6. **Seed de usuario RH** con BCrypt y `PasswordPolicy`; en Production las credenciales `Seed__Rh*` son obligatorias y el arranque falla si faltan.
7. **Tabla `restablecimientos_password`** se creó en la Fase 2 (no estaba en la especificación original).

## Implementación

### Backend (`app-backend/`, `docker-compose.yml`, `.env.example`, `infra/db.sql`)

- [x] **Docker:** `docker compose up --build` reproducible desde cero; `curl http://localhost:8080/health` devuelve 200 `Healthy`.
- [x] **Estructura por módulo:** `Endpoints` + `Contracts`, entidades en `Entities/`.
- [x] **Base:** OpenAPI solo en Development (`/openapi/v1.json`), logging estructurado, CORS para los orígenes de desarrollo.
- [x] **EF Core:** `Npgsql.EntityFrameworkCore.PostgreSQL`, `AppDbContext` con la cadena `Postgres`.
- [x] **Entidades base:** `Sucursal`, `Rol`, `Empleado`, `Departamento`, `Puesto`, `Equipo`, `Horario`, `Usuario`.
- [x] **Entidades de operación:** `Asistencia`, `CorreccionAsistencia`, `SolicitudAusencia`, `SaldoVacaciones`, `MovimientoVacaciones`, `HoraExtra`, `HoraExtraArchivo`, `Notificacion`, `Incidencia`, `Archivo`, `SesionUsuario`, `DispositivoPush`, `EquipoGerente`, `ContactoEmergencia`, `SolicitudAusenciaArchivo`, `Auditoria`.
- [x] **Enums de estado:** cuenta, zona, solicitud, marcación.
- [x] **Seeds (`DbSeeder`):** roles `colaborador`, `gerente`, `rh`; RH inicial; tienda; departamentos, puestos, horarios y equipos.

### Endpoints

- `GET /health` — público; reporta `postgres` y `almacenamiento` (tope de 3 s para el check de MinIO).
- Contratos y rutas de negocio: desde la Fase 2.

### Pantallas

- Web: —. Móvil: —.

## Criterios de Aceptación

- [x] `dotnet build` sin errores ni advertencias.
- [x] Contenedores `api` y `postgres` healthy.
- [x] Migraciones aplicadas sobre una base limpia.
- [x] OpenAPI disponible en desarrollo.
- [x] Seeds solo si no existen (idempotentes).

## Pendientes para otras fases

- [ ] Backups automáticos de PostgreSQL fuera del servidor (**F-INF-1**) y carga de configuración inicial en producción (**F-INF-3**): [[rh-cotla-fase-12-despliegue-sub-plan]].
- [ ] CI/CD, registry con etiquetado por versión y rollback formal (**F-INF-2**): [[rh-cotla-fase-12-despliegue-sub-plan]].
- [ ] Alinear la zona de la tienda entre `.env` y la base (la semilla solo aplica a una tabla vacía): [[rh-cotla-fase-03-configuracion-tienda-sub-plan]].

## Relacionado

- [[rh-cotla-plan]] — plan padre: stack, decisiones generales y orden de las 12 fases.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos implementados, pruebas, configuración (`appsettings.json`) y estado de la pila real.
- [[rh-cotla-integracion-clientes-manual]] — contrato de API para web y móvil, incluida la configuración del servidor.
- [[rh-cotla-pendientes-sub-plan]] — deuda y trabajo futuro con ids F-/FX-.
- [[rh-cotla-fase-02-autenticacion-colaboradores-sub-plan]] — siguiente fase: login, RBAC y colaboradores sobre este modelo.
- [[rh-cotla-fase-12-despliegue-sub-plan]] — despliegue en producción, donde viven los pendientes de infraestructura.
- Repo: `docs/especificaciones.md`, `app-backend/`, `docker-compose.yml`, `infra/db.sql`.
