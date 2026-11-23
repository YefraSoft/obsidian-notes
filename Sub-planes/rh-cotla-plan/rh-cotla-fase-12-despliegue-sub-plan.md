---
tipo: sub-plan
titulo: "Fase 12 — Despliegue"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: en-progreso
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - despliegue
  - docker
  - cloudflare
  - fase-12
---

# Fase 12 — Despliegue

> **Plan padre:** [[rh-cotla-plan]] — plan general del proyecto RH Cotla, con stack, roles y estado por fase.
> Despliegue de la versión 1.0.0 desde el equipo del proyecto con Docker Compose y Cloudflare Tunnel (`docs/especificaciones.md` §29 y §30). Cerrado con alcance acotado en el backend; CI/CD, backups y la publicación móvil siguen abiertos.

## Pendientes

- [ ] Backups automáticos de PostgreSQL con destino fuera del servidor y restauración probada (F-INF-1).
- [ ] CI/CD, registry de imágenes con etiquetado por versión, smoke tests, rollback formal, hosting definitivo y TLS propio (F-INF-2).
- [ ] Documentar la carga de configuración inicial en producción: tienda, zona, horarios, saldos y usuarios (F-INF-3).
- [ ] Rate limit general y proteger o retirar `/diagnostics/*` (FX-BE-3).
- [ ] `HEALTHCHECK` en Dockerfile y compose, y fijar el tag de MinIO (hoy `latest`) (FX-BE-4).
- [ ] Verificar tras desplegar las fases 10 y 11: compresión real contra MinIO, caché por el túnel y SMTP real (FX-BE-7).
- [ ] Exportar OpenAPI como artefacto del release (F-BE-7); en Production no se publica.
- [ ] Decisiones del cliente que condicionan el cierre (F-BE-8, §31).
- [ ] Una sola instancia del API: pub/sub del panel en memoria (FX-BE-8).
- [ ] Si el servidor tiene colaboradores demo 201–203 sembrados antes del cambio, desactivarlos a mano (FX-BE-9).
- [ ] Panel: puerto 3000 publicado en todas las interfaces (FX-FE-3) e imagen sin `HEALTHCHECK` que copia todo `node_modules` (FX-FE-9).
- [ ] Móvil: `eas init`, `EXPO_PUBLIC_API_URL` en `production`, heap de Gradle y ficha de tiendas (FX-MOV-2, FX-MOV-14); publicación en Google Play y App Store.
- [ ] Deuda aceptada A1 y A2 (no se corregirá en 1.0.0, ver Decisiones).

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | Cerrado con alcance acotado: Dockerfile reproducible, variables por entorno, endurecimiento previo a 1.0.0, rollback con `docker tag` documentado; suite 401/401 |
| Panel web | Parcial: servicio `frontend` dentro del `up` normal; imagen construida con bun 1.4.4 y `--frozen-lockfile` y arrancada el 20-nov-2026 (`/` y `/privacidad` responden 200); smoke test contra la API desplegada pendiente |
| App móvil | Parcial: identidad de tienda, versión 1.0.0, aviso de privacidad y build Android release local; falta EAS, publicación y smoke test |
| Infraestructura | Compose con `api`, `postgres` 17, `redis`, `minio` y `frontend`; Cloudflare Tunnel activo; base limpia verificada el 2026-11-20 |

Base limpia verificada el 2026-11-20: volumen `rh-cotla_postgres-data` recreado, una sucursal, RH y gerente sembrados. Con esa base el dashboard muestra 2 ausentes (111 y 112), ver FX-BE-1 en [[rh-cotla-fase-09-dashboards-calendario-sub-plan]].

## Alcance

Cubre el stack de contenedores, la exposición pública, las variables de entorno, el rollback y el endurecimiento previo a 1.0.0. No cubre CI/CD, hosting definitivo ni backups automatizados (v2). La publicación móvil en tiendas se rastrea aquí pero la ejecuta el equipo móvil.

## Decisiones

1. **Despliegue desde el equipo del proyecto** por Cloudflare Tunnel; el tunnel se gestiona en Cloudflare, no en el repositorio.
2. **Nombres públicos (solo estos dos):** `api.yefrasoft.com` → `http://127.0.0.1:8088` (puerto 8088 del compose → 8080 del contenedor) y `by.yefrasoft.com` → `http://127.0.0.1:3000` (panel). Caddy ya no interviene.
3. **Servicios del `docker-compose.yml`:** `api`, `postgres` (17), `redis` (`redis:7-alpine`), `minio` y `frontend`; todos con `restart: unless-stopped`. Volúmenes `postgres-data`, `minio-data` y `redis-data`. El `frontend` entra en el `docker compose up -d --build` normal (sin `--profile web`).
4. **Variables en `.env`** (fuera de git; `.env.example` sin valores sensibles, el compose falla si faltan): `POSTGRES_PASSWORD`, `REDIS_PASSWORD`, `SEED_RH_NUMERO_EMPLEADO`, `SEED_RH_PASSWORD`, `SEED_GERENTE_*`, `SEED_CREAR_COLABORADORES_DEMO` (`false` por defecto), `PASSWORD_RESET_BASE_URL` (obligatoria; apunta a `by.yefrasoft.com/reset-password`), `FRONTEND_API_URL` (por defecto `https://api.yefrasoft.com`; se incrusta al compilar el panel, cambiarla exige reconstruir), más JWT, SMTP y almacenamiento (`JwtSettings__SigningKey` de mínimo 32 bytes).
5. **Endurecimiento aplicado antes de 1.0.0:** contraseña de Postgres desde `.env`; puertos `5432`, `9000` y `9001` solo en `127.0.0.1`; colaboradores demo apagados; versión `1.0.0` en `cotla-api.csproj`; `.dockerignore` del API. En el panel: bun 1.4.4 única, `NEXT_PUBLIC_API_URL` obligatoria, sesión no persistente por defecto y cabeceras de seguridad.
6. **Rollback con `docker tag`:** `docker tag rh-cotla-api rh-cotla-api:previous` antes de actualizar; para volver, `docker tag rh-cotla-api:previous rh-cotla-api:latest` y `docker compose up -d --no-build api`. Es seguro mientras no haya migraciones nuevas (ninguna posterior a la Fase 6); con migraciones, hacer antes `docker exec cotla-postgres pg_dump -U cotla cotla_db > respaldo.sql`.
7. **Publicación de fases 10 y 11:** `docker compose up -d --build` (corte breve de 1–2 min), luego `curl -s https://api.yefrasoft.com/health` esperando postgres, almacenamiento y redis en `Healthy`. El API aplica migraciones al arrancar.
8. **Deuda aceptada (decisión del usuario):** **A1** CORS abierto (`AllowAnyOrigin`; `Cors:AllowedOrigins` no se lee) y **A2** puerto 8080/8088 publicado con `X-Forwarded-For` confiado desde redes privadas, que permite falsear IP y saltarse el límite de 10 intentos/min desde la LAN. Cerrar el 8080 o limitar `TrustedNetworks` la eliminaría.
9. **Panel y móvil:** el `Dockerfile` del panel pertenece al equipo web (contrato: `ARG NEXT_PUBLIC_API_URL` antes de `next build`, arranque `bun --bun next start -p 3000 -H 0.0.0.0`).

## Implementación

### Backend e infraestructura (`app-backend/`, `docker-compose.yml`, `.env.example`, `infra/db.sql`)
- [x] Dockerfile reproducible, imagen verificada con la librería nativa de SkiaSharp y usuario sin privilegios.
- [x] Secretos y cadenas de conexión por variables de entorno.
- [x] Tunnel de Cloudflare con los dos nombres públicos.
- [x] Fases 1–9 desplegadas y verificadas por `curl` con tokens de RH y gerente el 13-nov-2026 (sin escrituras sobre datos reales).
- [x] Base limpia verificada el 2026-11-20 con RH y gerente sembrados.
- [ ] Fases 10 y 11 en producción y verificadas (FX-BE-7).
- [ ] No verificado: reinicio del equipo con la política `restart`.

### Panel web (`app-frontend/`)
- [x] `Dockerfile` y `.dockerignore`; `NEXT_PUBLIC_API_URL` documentada en `README.md`.
- [x] Servicio `frontend` en el compose (puerto 3000 para `by.yefrasoft.com`).
- [ ] Smoke test del panel contra la API desplegada y recorrido en DevTools (FX-FE-11).
- [ ] `/privacidad` pública y `/api/v1/public/company` dependen de redesplegar panel y API.

### App móvil (`app-movil/`)
- [x] Identidad de tienda, versión 1.0.0, aviso de privacidad y build Android release local (`prebuild` + `./gradlew assembleRelease`).
- [ ] Builds con EAS (`eas.json`; requiere `eas init`) y `EXPO_PUBLIC_API_URL` en `production` (FX-MOV-2).
- [ ] Resolver los tres ⚠ antes de las tiendas: eliminación de cuenta (FX-MOV-1), `EXPO_PUBLIC_API_URL` (FX-MOV-2) y galería (FX-MOV-3).
- [ ] Publicación en Google Play y App Store; smoke test contra la API desplegada.

## Criterios de Aceptación

- [x] `docker compose up -d --build` levanta `api`, `postgres`, `redis`, `minio` y `frontend`.
- [x] `https://api.yefrasoft.com` y `https://by.yefrasoft.com` responden por el tunnel.
- [x] Sin secretos en el repositorio; el compose falla si faltan variables obligatorias.
- [x] Procedimiento de rollback de imagen documentado.
- [x] Base limpia con una sucursal, RH y gerente verificada el 2026-11-20.
- [ ] `/health` con postgres, almacenamiento y redis en `Healthy` tras desplegar las fases 10 y 11.
- [ ] Backups automáticos con restauración probada.
- [ ] CI/CD y smoke tests automatizados.
- [ ] Aplicación móvil publicada en las tiendas.

## Pendientes para otras fases

- [ ] FX-BE-2 (revocación de sesiones) y F-INF-1 (backups) encabezan la versión 1.0.1: ver [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]].
- [ ] Coordinación entre equipos: cualquier cambio de compose o puertos del panel lo pide el equipo web al backend.

## Relacionado

- [[rh-cotla-plan]] — plan padre con el estado general.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos del backend; §32 cierre de 1.0.0, rollback y registro de deuda A1–A2 y P1–P8.
- [[rh-cotla-integracion-clientes-manual]] — contrato para web y móvil; «Configuración del servidor» con tunnel y variables.
- [[rh-cotla-pendientes-sub-plan]] — deuda y trabajo futuro (F-INF-1 a F-INF-3, FX-BE-3/4/7/8/9, FX-FE-3/9, FX-MOV-14).
- [[rh-cotla-fase-01-fundaciones-sub-plan]] — fase inicial: Docker de desarrollo y modelo de datos.
- [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]] — SMTP y SSE que quedan por verificar en servidor.
- [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]] — fase anterior: Redis, caché y seguridad.
