---
tipo: sub-plan
titulo: "Asistencias — Slice 0: Esqueleto, despliegue y prueba de concepto del modelo"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-18
estado: en-progreso
proyecto: asistencias
padre: "[[asistencias-plan]]"
tags:
  - plan
  - sub-plan
  - asistencias
  - slice
  - cicd
  - poc-modelo
---

# Asistencias — Slice 0: Esqueleto, despliegue y prueba de concepto del modelo

> **Plan padre:** [[asistencias-plan]] · **Rama:** `feat/PoC` (fase 0a)
> Dos fases: **0a** PoC descartable en `poc/` para elegir modelo, umbral y formato de evidencia; **0b** esqueleto real (`api/`, `app-web/`) desplegable de punta a punta (CI → Harbor → Dokploy).

## Pendientes

- [ ] Probar la PoC con la cámara en la laptop y registrar resultados (3 a 5 personas, condiciones básicas).
- [ ] Probar en la red local (celulares con HTTPS: `mkcert` o la bandera de Chrome) y después en QA con HTTPS.
- [ ] Afinar los umbrales de gestos y el mapeo izquierda/derecha de MediaPipe con personas reales.
- [ ] Verificar que los vectores cliente y servidor coinciden (preprocesado idéntico, canales RGB/BGR) y decidir el esquema: cliente, promedio, consistencia o servidor.
- [ ] Decidir modelo (SFace fp32, `_8` o `_quant`), umbral y formato de evidencia; documentarlo en [[asistencias-plan]].
- [ ] Fase 0b: solución .NET y app Astro reales, `Dockerfile` ajustado, CI, remoto de git, secretos del CI y webhooks, stacks de Dokploy y volumen de modelos.
- [ ] Completar `POSTGRES_CONNECTION` en `apps/asistencias` y `apps/asistencias-prod` cuando exista `new-core`.
- [ ] Subir el límite de tasa del CDN (se necesita desde el slice 4).
- [ ] SemVer y flujo de ramas. Nota: el remoto tiene una rama `Staginng` (con typo) y los documentos dicen `staging`.

## Estado Actual

| Área | Estado |
|------|--------|
| Repositorio, `.gitignore`, `CLAUDE.md`, `AGENTS.md` | ✅ |
| `api/Dockerfile` y `.gitea/workflows/ci.yml` base | ✅ (nombre del proyecto .NET por ajustar en 0b) |
| KV de OpenBao (`apps/asistencias`, `apps/asistencias-prod`) | ✅ (falta `POSTGRES_CONNECTION`) |
| Rama `feat/PoC` | ✅ creada en el remoto |
| PoC: base PostgreSQL (compose + `referencia.sql`) | ✅ |
| PoC: backend (`poc/api`) | ✅ compila (0 advertencias); retos, historial de gestos y escenarios probados con curl |
| PoC: frontend (`poc/web`) | ✅ compila y sirve modelos y wasm; **falta probarlo con cámara** |
| PoC: medición con personas | ⏳ Pendiente |
| Esqueleto real (0b) y despliegue a QA | ⏳ Pendiente |

## Fase 0a — PoC (detalle)

**Objetivo:** elegir modelo, umbral, formato de evidencia y esquema de verificación midiendo **tiempos y fidelidad**.

**Estructura** (descartable, en `poc/`): `docker-compose.yml` (PostgreSQL `postgres:17-alpine`, `127.0.0.1:5433`), `db/referencia.sql` (esquema `poc_*`), `api/` (.NET minimal API con ONNX Runtime y Npgsql), `web/` (Vite + React + TypeScript con MediaPipe y ONNX Runtime Web), `storage/` (imágenes anotadas, fuera de git) y `README.md`.

**Pipeline idéntico en navegador y servidor:** YuNet (640x640, BGR 0-255) → alineación por 5 puntos (similitud, plantilla ArcFace de 112) → SFace (112x112 → 128 d) → normalización L2 → coseno (umbral inicial 0.363, el de referencia de SFace).

**Tres escenarios (un endpoint por escenario):**

| Escenario | Qué se compara | Endpoint |
|-----------|----------------|----------|
| 1 | Vector del **cliente** → backend | `POST /poc/escenarios/1/verificar` |
| 2 | **Promedio** normalizado cliente+backend, y **consistencia** cliente↔backend como control | `POST /poc/escenarios/2/verificar` |
| 3 | Vector del **backend** (desde la imagen) | `POST /poc/escenarios/3/verificar` |

La pantalla «Resultados» muestra los tres lado a lado con similitud, decisión y tiempos (cliente, servidor, comparación, total) y un resumen con similitud de genuinos e impostores, FRR y FAR.

**Retos de gestos:** 10 gestos con MediaPipe FaceLandmarker (parpadear, guiñar ojo izquierdo/derecho, sonreír, abrir la boca, levantar cejas, girar la cabeza izquierda/derecha, asentir, mirar arriba); el backend emite 2 al azar con nonce y vigencia de 60 s; cada gesto realizado se guarda en `poc_gesto_realizado` (historial por persona).

**Referencias temporales:** vectores y miniaturas en `localStorage`; el backend guarda solo imágenes anotadas (marco de la cara y texto rojo con fecha y nombre) en `poc/storage/`.

## Hallazgos de la PoC (hasta ahora)

- **Los modelos `fr_yolo*` no son de reconocimiento:** son detectores de cara YOLOv8 de una clase (salida `[1,5,8400]`, Ultralytics; `fr_yolo_16` en fp16 y `fr_yolo_gpu` con entradas dinámicas). **El único modelo que genera embeddings es SFace** (`fr_sface`, `_8`, `_quant`: entrada `[1,3,112,112]`, salida `[1,128]`). YuNet entrega caja y 5 puntos faciales (salidas por stride 8/16/32), entrada fija 640x640.
- **Licencias:** ImageSharp 4.x exige licencia de Six Labors incluso para compilar en Release; la PoC usa **SkiaSharp (MIT)**. Tenerlo presente para el backend real. Los modelos YOLO de Ultralytics son AGPL-3.0.
- El historial de gestos y los intentos viven en PostgreSQL; en el slice 4 migran a `new-core`.

## Fase 0b — Esqueleto y despliegue (pendiente de detallar al terminar la PoC)

- **BD:** ninguna tabla de negocio; conexión y chequeo de salud contra `new-core`.
- **API:** solución .NET de un proyecto con capas por carpeta, `health/ready`, `ProblemDetails` y configuración desde OpenBao/variables.
- **Pantalla:** app Astro + React mínima.
- **Despliegue:** Dockerfile de API y app web, imágenes en Harbor, stacks en Dokploy (testing sigue `QA`, producción sigue `main`), secretos en OpenBao, modelos por volumen.

## Entrega

1. Ramas `feat/PoC` (0a) y la rama del esqueleto (0b) desde `QA` → PR a `QA` (lo aprueba QA) → CI publica a Harbor y despliega **testing**.
2. Nota de entrega: `Testing/asistencias-slice-0-testing`; ajustes `fix/*` a `QA`.
3. PR a `staging` → PR a `main`.

## Criterios de Aceptación

- [ ] La PoC deja resultados medidos (tiempos y fidelidad por escenario) y un modelo, umbral, formato de evidencia y esquema de verificación elegidos y documentados.
- [ ] Un push a `QA` compila, publica la imagen en Harbor y redespliega testing.
- [ ] `health/ready` responde y refleja la conexión a `new-core`.
- [ ] Ningún secreto, modelo ni imagen de personas está en git ni en la imagen.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-arquitectura-wiki]]
- [[flujo-ci-cd-manual]]
- [[versionado-semver-manual]]
- [[asistencias-slice-3-enrolamiento-sub-plan]]
