---
tipo: sub-plan
titulo: "Asistencias — Slice 3: Enrolamiento (embeddings de referencia)"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-18
estado: pendiente
proyecto: asistencias
padre: "[[asistencias-plan]]"
tags:
  - plan
  - sub-plan
  - asistencias
  - slice
  - reconocimiento-facial
---

# Asistencias — Slice 3: Enrolamiento (embeddings de referencia)

> **Plan padre:** [[asistencias-plan]] · **Rama:** `feat/slice-3-enrolamiento`
> Genera y guarda el embedding de referencia de cada persona a partir de su foto en el CDN, con el modelo elegido en el slice 0, y deja listo el cliente del CDN y la gestión de sus keys.

## Pendientes

- [ ] Detallar el slice con el responsable y crear sus sub-tareas en Plane.
- [ ] Definir cómo y cuándo se generan los embeddings (por lotes desde la URL de foto del Core, bajo demanda en el primer ingreso o con el registro existente).
- [ ] Definir la gestión de keys del CDN: `X-Api-Key` rotatoria (cómo llega), `private_route_key` y `X-User-Key` en OpenBao.
- [ ] Definir el flujo del aviso de privacidad y consentimiento antes de enrolar.
- [ ] Definir qué pasa si la persona no tiene foto o la foto no es utilizable.

## Estado Actual

| Área | Estado |
|------|--------|
| Modelos disponibles | YuNet (detección); SFace y variantes YOLO (en `modelos/`, fuera de git) |
| Elección de modelo | Se decide en el slice 0 (PoC) |
| Implementación | ⏳ Pendiente |

## Alcance (vertical)

- **BD:** tabla de embeddings de referencia por usuario (con versión de modelo) y registro de consentimiento.
- **API:** cliente del CDN (lectura de la foto de referencia), generación del embedding en el servidor con ONNX Runtime y su almacenamiento; puerto `IFaceMatcher` (comparación y umbral configurable) reutilizable por la checada.
- **Pantalla:** aviso de privacidad y consentimiento y, para `ADMIN`, estado del enrolamiento por persona.
- **Docs:** nota de testing y datos sensibles (acceso y retención de embeddings).

## Implementación

### Bloque CREAR
- [ ] Entidades y migración; puertos `IFaceMatcher` y cliente del CDN; generación del embedding.

### Bloque MODIFICAR
- [ ] Ajustes del flujo de ingreso del slice 1 para disparar el enrolamiento.

### Bloque ENLAZAR
- [ ] Servicio de enrolamiento, endpoints y pantalla de consentimiento; verificación con fotos de prueba.

## Entrega

1. `feat/slice-3-enrolamiento` desde `QA` → PR a `QA` → **testing**.
2. `Testing/asistencias-slice-3-testing`; ajustes `fix/*`.
3. PR a `staging` → PR a `main`.

## Criterios de Aceptación

- [ ] Todo usuario con foto de referencia utilizable tiene su embedding en `new-core`.
- [ ] El modelo y el umbral son configurables sin recompilar y el modelo es el mismo en cliente y servidor.
- [ ] Los embeddings y las fotos se tratan como datos sensibles (acceso definido y consentimiento registrado).

## Relacionado

- [[asistencias-plan]]
- [[asistencias-slice-0-esqueleto-sub-plan]]
- [[asistencias-slice-1-ingreso-roles-sub-plan]]
- [[asistencias-slice-4-checada-valida-sub-plan]]
- [[cdn-service-manual]]
