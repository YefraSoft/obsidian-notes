---
tipo: sub-plan
titulo: CI/CD — Gitea Actions, Harbor y Dokploy
autor: Efraín García
creado: 2026-11-03
actualizado: 2026-11-09
estado: en-progreso
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - cicd
  - gitea
  - harbor
  - dokploy
---

# CI/CD — Gitea Actions, Harbor y Dokploy

> **Plan padre:** [[email-service-plan]]
> Que CI corra en cada push y que la entrega a testing sea un `git push` a `staging`: imagen en Harbor, fila del historial y redeploy en Dokploy automáticos. Los secretos de la app siguen en OpenBao y los `ALTER` siguen siendo manuales.

## Pendientes

- [x] Dialogar el diseño con Efraín (2026-11-03).
- [x] `push-harbor.sh`: publica también el panel, modo `CI=1`, salida `VERSION=`.
- [x] `compose.staging.yml`: servicio `panel` y healthcheck de la API.
- [x] `.gitea/workflows/ci.yml` (con el job `qa-testing`: Playwright y k6 contra testing tras cada entrega) e `infra/ci/act-runner/*`.
- [x] Alta en infraestructura (Efraín, con lista de pasos): Actions en Gitea, runner en el servidor de Dokploy, secretos del repo, rama `staging` y webhook en Dokploy.
- [ ] Primera entrega por pipeline (slices 3 a 6 a testing) y E2E (las imágenes desde `0.0.0.04` ya las publica `gitea-actions`, ver `docs/versions/version-history.md`; falta el E2E).
- [ ] Documentación (wiki, AGENTS, README).

## Estado Actual

| Área | Estado |
|------|--------|
| `push-harbor.sh` manual (solo API), historial `X.Y.Z.TT` | ✅ |
| `compose.staging.yml` con `pull_policy: always`, secretos desde Dokploy/OpenBao | ✅ |
| Dockerfile del panel | ✅ (servicio `panel` en `compose.staging.yml`) |
| Runner de Gitea, workflow, secretos de CI | ✅ (entregas de `gitea-actions` desde `0.0.0.04`) |

## Decisiones (2026-11-03)

1. **Mapa de ramas y etiquetas.** `feature/*` y `main`: solo CI (build .NET sin advertencias; `check`, `lint`, `format` y `build` del panel; `dotnet test` cuando exista el proyecto). `staging`: CI + imágenes + historial + redeploy en testing; se entrega con `git merge main` en `staging` y push. Etiquetas `vX.Y.Z`: producción, se conecta después. Motivo: el desarrollo va a `main` con un commit por paso y no debe desplegar a mitad de un slice; testing por etiqueta generaría decenas de `T` sin valor; las etiquetas inmutables se reservan para producción, que calcula sobre `X.Y.Z`.
2. **Runner** `gitea/act_runner` en Docker en el servidor de Dokploy con el socket del host (build amd64 nativo, red local a Dokploy).
3. **Secretos del pipeline** en *Secrets* del repositorio de Gitea (`HARBOR_USER`, `HARBOR_PASSWORD`, `DOKPLOY_WEBHOOK_URL`); el push del historial con el token automático de Actions. OpenBao no cambia.
4. **El panel entra en testing** como imagen `email-service-panel` y servicio `panel` del compose (`API_URL` al servicio de la API, puerto 5038).
5. **Historial:** lo escribe el job con la lógica del script, solo si las dos imágenes subieron; el bot lo commitea en `staging` con `[skip ci]` y después se trae a `main` con `git merge staging`.
6. **`ALTER` manuales:** el job avisa en su resumen si `infra/db.sql` cambió desde la entrega anterior.
7. **Forma de trabajo:** los pasos de código los hace un sub-agente; los de infraestructura los ejecuta Efraín con una lista de pasos (sin secretos) y se comprueban con medios de solo lectura.

## Alcance

Cubre CI en Gitea y la entrega a testing (API y panel). No cubre producción (etiquetas `v*`), migraciones automáticas, tests ni un AppRole de CI en OpenBao.

## Relacionado

- [[email-service-plan]]
- [[email-service-grupos-sub-plan]]
