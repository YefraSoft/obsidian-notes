---
tipo: sub-plan
titulo: "Asistencias — Checada válida"
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
---

# Asistencias — Checada válida

> **Plan padre:** [[asistencias-plan]]
> Caso de uso central: registrar una checada y decidir si es válida.

## Pendientes

- [ ] Definir estados de la checada (p. ej. válida, rechazada por ubicación, rechazada por rostro).
- [ ] Definir política de reintentos y de checadas rechazadas.

## Estado Actual

| Área | Estado |
|------|--------|
| Diseño del contrato | ⏳ Pendiente |
| Implementación | ⏳ Pendiente |

## Alcance

Cubre el endpoint y el servicio que reciben ubicación GPS y fotografía, aplican las tres condiciones y guardan el resultado. No cubre el catálogo de centros ([[asistencias-geocerca-sub-plan]]) ni el motor de reconocimiento ([[asistencias-reconocimiento-facial-sub-plan]]).

Una checada es **válida** cuando: (1) el GPS está dentro de la geocerca del centro de trabajo, (2) trae una fotografía tomada con la cámara del dispositivo (no se admiten archivos subidos) y (3) el reconocimiento facial confirma a la persona.

## Implementación

### Paso 1 — Contrato y entidades
- [ ] Entidad checada (persona, centro, fecha UTC, coordenadas, precisión, resultado, motivo).
- [ ] Contrato del request (ubicación + imagen) y respuesta.

### Paso 2 — Servicio de checada
- [ ] Validar geocerca, ejecutar reconocimiento, persistir con un solo `CommitAsync`.

### Paso 3 — Evidencia
- [ ] Almacenar la foto de evidencia y definir su retención.

## Criterios de Aceptación

- [ ] Una checada fuera de la geocerca se rechaza con motivo.
- [ ] Una checada con rostro distinto o sin rostro se rechaza con motivo.
- [ ] No existe ruta para subir archivos arbitrarios como evidencia.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-geocerca-sub-plan]]
- [[asistencias-reconocimiento-facial-sub-plan]]
