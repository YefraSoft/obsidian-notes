---
tipo: plan
titulo: "Asistencias Tecnológico Superior de Jalisco — Plan de Desarrollo"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-17
estado: en-progreso
proyecto: asistencias
tags:
  - plan
  - asistencias
  - reconocimiento-facial
---

# Asistencias Tecnológico Superior de Jalisco — Plan de Desarrollo

> App de checadas para el personal: web ahora (migrará a móvil), backend en C# y modelos de reconocimiento facial que validan cada checada. Plan maestro; las tareas y entregables viven en Plane (proyecto "App Asistencias", workspace `direccion-de-sistemas`).

## Pendientes

- [ ] Definir stack del frontend y base de datos.
- [ ] Definir el modelo de reconocimiento de producción (SFace vs variantes YOLO) y su umbral.
- [ ] Definir el esquema de versionado SemVer y la rama base del flujo (QA → staging → main).
- [ ] Definir cómo se enrolan los rostros de referencia de cada persona.
- [ ] Confirmar rutas KV de OpenBao y volumen de modelos en Dokploy.

## Estado Actual (resumen ejecutivo)

| Área | Estado |
|------|--------|
| Repositorio (`checadores-app`) | Inicializado: `.gitignore`, `CLAUDE.md`, `AGENTS.md`, Dockerfile y CI base |
| API (C#) | ⏳ Sin código |
| App web | ⏳ Sin código |
| Modelos ONNX | Disponibles en `modelos/` (fuera de git); modelo de producción por decidir |
| CI/CD | Base creada (`.gitea/workflows/ci.yml`); versionado SemVer pendiente |

| Fase | Estado |
|------|--------|
| FASE 1 — Fundamentos (repo, CI/CD, secretos, docs) | 🔄 En progreso |
| FASE 2 — Personal, centros de trabajo y geocerca | ⏳ Pendiente |
| FASE 3 — Checada con evidencia y reconocimiento facial | ⏳ Pendiente |
| FASE 4 — App web y pruebas | ⏳ Pendiente |
| FASE 5 — Producción y migración a móvil | ⏳ Pendiente |

## Equipo

- Efraín García — Desarrollo
- Gabriel Cisneros — Líder de equipo
- Ana Espinoza — Scrum Master

## Decisiones de Diseño (vigentes)

- **Checada válida** = (1) ubicación GPS dentro de la geocerca del centro de trabajo, (2) fotografía de evidencia tomada con la cámara del dispositivo (no existe opción de subir archivos) y (3) el reconocimiento facial confirma que el rostro es de la persona que checa.
- La geocerca y el reconocimiento facial se validan **en el servidor**; el cliente solo captura y envía.
- La app web captura la foto en vivo con la cámara; nunca con un selector de archivos.
- Backend en C# con arquitectura Clean adaptada de un solo proyecto (`Core`, `Application`, `Infrastructure`, `Api`, `Platform`), igual que `email-service`.
- **Reparto de herramientas:** Obsidian = plan maestro, sub-planes, decisiones de diseño y documentación general (desarrollo y producción); Plane = tareas y entregables.
- Los modelos ONNX no se versionan en git ni van dentro de la imagen; se montan como volumen (huellas SHA-256 en `modelos/README.md` del repo).

## Arquitectura y Notas Técnicas

- **Stack:** API en C# / .NET con EF Core code-first; ONNX Runtime para detección (YuNet) y reconocimiento (SFace / YOLO); frontend por definir.
- **Despliegue:** la app se despliega en **Dokploy**. Cada servicio necesita su `Dockerfile`; la imagen se publica en **Harbor** y Dokploy la descarga (redespliegue por webhook desde el CI).
- **Secretos:** se versionan en **OpenBao** como KV (`secret/asistencias` en desarrollo; la ruta de producción se confirma). Aquí solo se documentan rutas y nombres de clave, nunca valores.
- **CI/CD (Gitea Actions):** push/PR compila API y app web; `QA` publica y despliega testing; `main` publica y despliega producción. Flujo de ramas y versionado según [[flujo-ci-cd-manual]] y [[versionado-semver-manual]].
- Referencia de implementación: [[email-service-plan]].

## Archivos Clave del Proyecto

| Archivo | Propósito |
|---------|-----------|
| `CLAUDE.md` / `AGENTS.md` | Objetivo y reglas de trabajo del repo |
| `.gitignore` | Excluye `modelos/`, `.env*`, binarios |
| `modelos/README.md` | Registro y huellas de los modelos ONNX |
| `api/Dockerfile` | Imagen de la API para Harbor/Dokploy |
| `.gitea/workflows/ci.yml` | CI/CD |

## FASE 1 — Fundamentos

> **Objetivo:** repo, CI/CD, secretos y documentación listos para desarrollar.
> **Estado:** 🔄 En progreso

### 1.1 Repositorio y despliegue
- [x] `.gitignore` (modelos fuera de git) y documentación del repo.
- [x] Dockerfile de la API y CI base.
- [ ] Secretos KV en OpenBao.
- [ ] Remoto de git y secretos del CI en Gitea.

### 1.2 Gestión
- [x] Sub-planes en Obsidian.
- [ ] Proyecto y tareas en Plane.

Detalle: [[asistencias-cicd-sub-plan]].

## FASE 2 — Personal, centros de trabajo y geocerca

> **Objetivo:** catálogo de personas y centros con su geocerca. Detalle: [[asistencias-geocerca-sub-plan]].
> **Estado:** ⏳ Pendiente

## FASE 3 — Checada con evidencia y reconocimiento facial

> **Objetivo:** registrar checadas válidas. Detalle: [[asistencias-checada-valida-sub-plan]] y [[asistencias-reconocimiento-facial-sub-plan]].
> **Estado:** ⏳ Pendiente

## FASE 4 — App web y pruebas

> **Objetivo:** pantallas de checada y administración; pruebas de QA. Detalle: [[asistencias-app-web-sub-plan]].
> **Estado:** ⏳ Pendiente

## FASE 5 — Producción y migración a móvil

> **Objetivo:** despliegue productivo y plan de la versión móvil.
> **Estado:** ⏳ Pendiente

## Dependencias y Orden de Ejecución

```
FASE 1 — Fundamentos
    │
    └── FASE 2 — Personal, centros y geocerca
         │
         └── FASE 3 — Checada y reconocimiento facial
              │
              └── FASE 4 — App web y pruebas
                   │
                   └── FASE 5 — Producción y móvil
```

## Relacionado

- [[asistencias-checada-valida-sub-plan]]
- [[asistencias-reconocimiento-facial-sub-plan]]
- [[asistencias-geocerca-sub-plan]]
- [[asistencias-cicd-sub-plan]]
- [[asistencias-app-web-sub-plan]]
- [[email-service-plan]]
