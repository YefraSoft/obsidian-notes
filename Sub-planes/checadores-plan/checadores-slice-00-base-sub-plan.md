---
tipo: sub-plan
titulo: Slice 0 — Base y plataforma
autor: Efraín García
creado: 2026-11-23
actualizado: 2026-11-23
estado: pendiente
proyecto: checadores
padre: "[[checadores-plan]]"
tags:
  - checadores
  - sub-plan
  - base
  - slice-0
---

# Slice 0 — Base y plataforma

> **Plan padre:** [[checadores-plan]] · **Rama:** `feat/slice-00-base` · Plane: CHECADORES-1 (tareas 14–19)
> Cimientos técnicos sobre los que se montan los demás slices: API .NET 10, front Astro, Postgres, Docker, ramas y caché de configuración. Sin lógica de negocio.

## Pendientes

- [ ] Entrevista previa: versiones y paquetes NuGet/npm a aprobar (requiere confirmación).
- [ ] Confirmar el esquema Postgres (`docs/esquema-postgres.sql`) antes de generar migraciones.
- [ ] Decidir entre `BackgroundService` y Quartz para los procesos periódicos (auditor).
- [ ] Definir si la migración de EF corre al arrancar la API (como en email-service) o la lanza el responsable.

## Estado Actual

| Área | Estado |
|------|--------|
| Requisitos | ✅ Definidos |
| Proyectos en blanco | ⏳ |
| Compose, CI e Insomnia | ⏳ |

## Legacy → nuevo

| Legacy | Nuevo |
|--------|-------|
| Node/pkg con código y estrategias dentro de SQLite; secretos en archivos `.json` | .NET 10 compilado; secretos solo por entorno |
| Sin entorno reproducible | `compose.yaml` con `api`, `worker`, `front`, `postgres` |
| Sin CI ni ramas | `feat/*` → `Qa` → `Staging` → `main` |

## Decisiones

1. Un solo proyecto .NET con capas por carpeta; `api` y `worker` salen del mismo código.
2. Astro `output: 'server'` con adapter Node, React, Tailwind vía Vite y Actions; Bun como gestor.
3. Configuración editable en Postgres servida desde **caché en memoria**; invalida al guardar y el worker recarga por versión (sin SQLite).
4. Convenciones de BD: `snake_case` en español, `timestamptz`, claves únicas de idempotencia.
5. Secretos únicamente por variables de entorno; `.env.example` lista solo nombres.

## Alcance

Cubre esqueleto, infraestructura local y de despliegue, convenciones y caché de configuración. No cubre ninguna regla de negocio ni pantalla funcional.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[checadores-plan]]. Ciclo CREAR → MODIFICAR → ENLAZAR con un sub-agente por paso.

## Implementación

### Paso 1 — API (00.1)
- [ ] `checadores-back/` con `Src/{Core,Application,Infrastructure,Api,Platform}`, `Program.cs` mínimo y `GET /health`.
- [ ] Jerarquía `DomainException` con nivel de log en la base, `GlobalExceptionHandler` y `ProblemDetails` (`code`, `traceId`).
- [ ] Logs estructurados (JSON) y `TimeProvider` inyectado.

### Paso 2 — Front (00.2)
- [ ] `checadores-front/` con Astro server, `@astrojs/node`, `@astrojs/react`, Tailwind, Actions y `Dockerfile`; cliente de la API solo en el servidor.

### Paso 3 — Postgres (00.3)
- [ ] `ChecadoresDbContext`, convenciones, primera migración y `IDataTransaction` con traducción de unicidad a `InConflictException`.

### Paso 4 — Configuración en caché (00.6)
- [ ] Tabla `configuracion` con `version`; puerto `IConfiguracionCache` en `Application`; implementación en `Infrastructure` con invalidación al guardar y recarga por versión en el worker.

### Paso 5 — Docker, ramas e Insomnia (00.4 y 00.5)
- [ ] `compose.yaml` con healthchecks; `.env.example` sin valores; `Dockerfile` de api y front.
- [ ] Compilación automática por rama; colección de Insomnia con `baseUrl` y `apiKey` vacía.

## Criterios de Aceptación

- [ ] `dotnet build` con 0 errores y 0 advertencias; `bun run build` correcto.
- [ ] `docker compose up` levanta los cuatro servicios y `curl /health` responde.
- [ ] Cambiar una preferencia se refleja en `api` y `worker` sin reiniciar.
- [ ] Ninguna credencial queda en el repo.

## Relacionado

- [[checadores-plan]]
- [[checadores-slice-01-identidad-sub-plan]]
