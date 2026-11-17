---
tipo: sub-plan
titulo: "Asistencias — CI/CD y despliegue"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-17
estado: en-progreso
proyecto: asistencias
padre: "[[asistencias-plan]]"
tags:
  - plan
  - sub-plan
  - asistencias
  - cicd
  - dokploy
  - harbor
  - openbao
---

# Asistencias — CI/CD y despliegue

> **Plan padre:** [[asistencias-plan]]
> Compilación, publicación de imágenes y despliegue. Flujo general del equipo: [[flujo-ci-cd-manual]]; versionado: [[versionado-semver-manual]].

## Pendientes

- [ ] Secretos KV en OpenBao (ruta de desarrollo y de producción).
- [ ] Secretos del repositorio en Gitea (nombres en `.gitea/workflows/ci.yml`).
- [ ] Crear los stacks de testing y producción en Dokploy y sus webhooks.
- [ ] Definir volumen de modelos ONNX en Dokploy.
- [ ] Versionado SemVer y fila de historial (adaptar `push.sh` de `email-service`).
- [ ] Dockerfile de la app web cuando se defina su stack.

## Estado Actual

| Área | Estado |
|------|--------|
| `api/Dockerfile` y `.dockerignore` | ✅ Creados (nombre del proyecto .NET por ajustar al crear el `.csproj`) |
| `.gitea/workflows/ci.yml` | ✅ Base: build, publicación a Harbor y webhook de Dokploy por rama |
| OpenBao KV | ⏳ Pendiente de confirmar ruta |
| Dokploy | ⏳ Pendiente |

## Alcance

- **Despliegue en Dokploy**: cada servicio con su `Dockerfile`; Dokploy descarga la imagen de Harbor y se redespliega por webhook.
- **Imágenes en Harbor**: `asistencias-api` (y la de la app web); etiquetas por commit y `latest` hasta que exista SemVer.
- **Secretos en OpenBao**: KV `secret/asistencias` (desarrollo); la ruta de producción se confirma. Se documentan rutas y nombres de clave, nunca valores.
- Los modelos ONNX no van en la imagen: volumen montado en `/models`.

## Implementación

### Paso 1 — Imágenes y CI
- [x] Dockerfile de la API y CI base.

### Paso 2 — Secretos
- [ ] Crear los KV (solo los KV; sin AppRole ni políticas) y documentar claves.

### Paso 3 — Dokploy
- [ ] Stack de testing (sigue `QA`) y de producción (sigue `main`), con las variables que reflejan los KV.

## Criterios de Aceptación

- [ ] Un push a `QA` publica la imagen en Harbor y redespliega testing.
- [ ] Un push a `main` hace lo mismo hacia producción.
- [ ] Ningún secreto ni modelo está en git ni en la imagen.

## Relacionado

- [[asistencias-plan]]
- [[flujo-ci-cd-manual]]
- [[versionado-semver-manual]]
- [[email-service-cicd-sub-plan]]
