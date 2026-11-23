---
tipo: sub-plan
titulo: "Asistencias — Slice 4: Checada válida"
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
  - checada
---

# Asistencias — Slice 4: Checada válida

> **Plan padre:** [[asistencias-plan]] · **Rama:** `feat/slice-4-checada`
> Caso de uso central: registrar entrada o salida y decidir si es válida. Reglas: [[asistencias-reglas-de-negocio-wiki]]. Secuencia: [[asistencias-arquitectura-wiki]].

## Pendientes

- [ ] Detallar el slice con el responsable y crear sus sub-tareas en Plane.
- [ ] Definir qué valida el backend con el procesamiento en el cliente (forma/dimensión del embedding, hora del servidor, geocerca, reto no reutilizado, bloque de horario).
- [ ] Definir el reto (emisión por el servidor, un solo uso, vigencia) con el resultado de la PoC del slice 0.
- [ ] Definir formato y tamaño de la evidencia (clip corto con fotograma JPEG o GIF) y su ruta en el CDN (colección e identificador; archivo con UUID).
- [ ] Subir el límite de tasa del CDN y resolver cómo llegan las keys temporales de subida al navegador.
- [ ] Definir el comportamiento sin internet (se avisa; no se checa).

## Estado Actual

| Área | Estado |
|------|--------|
| Reglas | ✅ Definidas |
| Dependencias | Requiere slices 2 (geocerca) y 3 (embeddings de referencia) |
| Implementación | ⏳ Pendiente |

## Alcance (vertical)

- **BD:** `reto`, `checada`, `intento` (persona, UA, bloque, tipo entrada/salida, fecha UTC, coordenadas, precisión, resultado, motivo, referencia a la evidencia) en `new-core`.
- **API:** emisión del reto, registro de la checada con verificación (sesión, reto vigente y no usado, geocerca, bloque de horario, comparación de embedding con el umbral), registro de intentos rechazados con motivo, un solo `CommitAsync` por caso de uso. Sin ruta para subir archivos arbitrarios.
- **Pantalla (cliente):** captura de GPS, cámara en vivo, reto de gestos (MediaPipe), cálculo del embedding en el navegador, **subida directa del clip al CDN** con las keys temporales que entrega el servidor de Astro (solo en sesión con reto activo), y resultado con motivo y reintento inmediato. Sin selector de archivos.
- **Docs:** nota de testing y actualización de la arquitectura.

El procesamiento pesado ocurre en el frontend y el backend confía en esos datos; verifica la comparación contra el embedding de referencia, la geocerca y las reglas.

## Implementación

### Bloque CREAR
- [ ] Entidades y migración (`reto`, `checada`, `intento`); servicio del reto; servicio de checada.
- [ ] Componentes del cliente: captura, reto de gestos, embedding y subida al CDN.

### Bloque MODIFICAR
- [ ] Ajustes al modelo de usuario/horarios para localizar el bloque aplicable.

### Bloque ENLAZAR
- [ ] Astro Actions (`iniciarChecada`, `registrarChecada`), pantalla de checada y entrega de keys temporales; verificación con curl y pruebas en celulares.

## Entrega

1. `feat/slice-4-checada` desde `QA` → PR a `QA` → **testing**.
2. `Testing/asistencias-slice-4-testing`; ajustes `fix/*` con pruebas en dispositivos reales.
3. PR a `staging` → PR a `main`.

## Criterios de Aceptación

- [ ] Una checada fuera de la geocerca o con rostro que no coincide se rechaza con motivo y queda como intento.
- [ ] La persona puede reintentar de inmediato con un reto nuevo.
- [ ] No existe forma de subir un archivo como evidencia.
- [ ] El clip se sube directo al CDN y a la API solo llega su `fileId`.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-arquitectura-wiki]]
- [[asistencias-reglas-de-negocio-wiki]]
- [[asistencias-slice-2-ua-geocerca-sub-plan]]
- [[asistencias-slice-3-enrolamiento-sub-plan]]
- [[asistencias-slice-5-horarios-incidencias-sub-plan]]
