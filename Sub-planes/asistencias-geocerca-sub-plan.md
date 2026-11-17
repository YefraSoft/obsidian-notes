---
tipo: sub-plan
titulo: "Asistencias — Centros de trabajo y geocerca"
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
  - geocerca
---

# Asistencias — Centros de trabajo y geocerca

> **Plan padre:** [[asistencias-plan]]
> Catálogo de centros de trabajo y verificación de que una coordenada GPS cae dentro de su ubicación establecida.

## Pendientes

- [ ] Definir la forma de la geocerca (círculo con radio o polígono).
- [ ] Definir el manejo de la precisión del GPS reportada por el dispositivo.

## Estado Actual

| Área | Estado |
|------|--------|
| Diseño | ⏳ Pendiente |
| Implementación | ⏳ Pendiente |

## Alcance

CRUD de centros de trabajo con su geocerca, asignación de personal a centros y puerto de verificación de ubicación en `Application`. No cubre la captura de GPS en el cliente ([[asistencias-app-web-sub-plan]]).

## Implementación

### Paso 1 — Entidades y migración
- [ ] Centro de trabajo y geocerca; relación con personal.

### Paso 2 — Verificación
- [ ] Servicio que decide si una coordenada (con su precisión) está dentro de la geocerca.

### Paso 3 — Administración
- [ ] Endpoints REST de gestión.

## Criterios de Aceptación

- [ ] Una coordenada dentro de la geocerca se acepta; fuera, se rechaza con motivo.
- [ ] Una precisión peor que el límite configurado se rechaza.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-checada-valida-sub-plan]]
