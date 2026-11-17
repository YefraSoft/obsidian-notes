---
tipo: sub-plan
titulo: "Asistencias — Reconocimiento facial"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-17
estado: pendiente
proyecto: asistencias
padre: "[[asistencias-plan]]"
tags:
  - plan
  - sub-plan
  - asistencias
  - reconocimiento-facial
---

# Asistencias — Reconocimiento facial

> **Plan padre:** [[asistencias-plan]]
> Detección y comparación de rostros con modelos ONNX para validar la checada.

## Pendientes

- [ ] Elegir el modelo de reconocimiento de producción (SFace o variantes YOLO) y confirmar el significado de los sufijos `_8`, `_quant`, `_16`, `_gpu`.
- [ ] Definir el umbral de similitud y la tasa de falsos aceptados / rechazados aceptable.
- [ ] Definir el enrolamiento de rostros de referencia y su protección (datos biométricos).

## Estado Actual

| Área | Estado |
|------|--------|
| Modelos disponibles | Detección YuNet; reconocimiento SFace y variantes YOLO (en `modelos/`, fuera de git) |
| Integración con ONNX Runtime | ⏳ Pendiente |

## Alcance

Puerto de reconocimiento en `Application` e implementación en `Infrastructure` con ONNX Runtime: detectar un rostro, obtener su embedding y compararlo con el de referencia de la persona. No cubre la regla de checada ([[asistencias-checada-valida-sub-plan]]).

## Implementación

### Paso 1 — Puerto y contrato
- [ ] Puerto `IFaceRecognizer` (detectar, extraer embedding, comparar).

### Paso 2 — Implementación ONNX
- [ ] Carga de modelos desde la ruta montada (volumen), con verificación de huella contra `modelos/README.md`.
- [ ] Preprocesado, inferencia y comparación.

### Paso 3 — Enrolamiento
- [ ] Alta y almacenamiento seguro del embedding de referencia.

## Criterios de Aceptación

- [ ] Sin rostro o con más de uno en la foto, la validación falla con motivo claro.
- [ ] El modelo y el umbral son configurables sin recompilar.
- [ ] Los embeddings y fotos se tratan como datos sensibles (acceso y retención definidos).

## Relacionado

- [[asistencias-plan]]
- [[asistencias-checada-valida-sub-plan]]
