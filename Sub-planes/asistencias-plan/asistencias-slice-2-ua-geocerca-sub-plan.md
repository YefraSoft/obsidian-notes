---
tipo: sub-plan
titulo: "Asistencias — Slice 2: UA y geocerca"
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
  - geocerca
---

# Asistencias — Slice 2: UA y geocerca

> **Plan padre:** [[asistencias-plan]] · **Rama:** `feat/slice-2-ua-geocerca`
> Catálogo de las 17 UA con su geocerca y verificación de que una coordenada GPS cae dentro de ella; se administra desde el panel.

## Pendientes

- [ ] Detallar el slice con el responsable y crear sus sub-tareas en Plane.
- [ ] Definir la forma de la geocerca (círculo con radio o polígono).
- [ ] Definir la precisión de GPS máxima aceptada y la tolerancia por UA.
- [ ] Levantar coordenadas y radios de las 17 UA (se capturan en el panel).
- [ ] Definir si una persona puede estar adscrita a más de una UA.

## Estado Actual

| Área | Estado |
|------|--------|
| Requisitos | ✅ Definidos: se configuran en el panel de administración |
| Implementación | ⏳ Pendiente |

## Alcance (vertical)

- **BD:** UA y geocerca (coordenadas, forma o radio, tolerancia) en `new-core`; adscripción del personal a su UA.
- **API:** CRUD de UA y geocercas (solo `ADMIN`) y servicio de verificación de ubicación en `Application` (coordenada + precisión).
- **Pantalla:** panel de administración de UA y geocercas (mapa o captura de coordenadas y radio).
- **Docs:** nota de testing del slice y actualización de las reglas de negocio.

No cubre la captura de GPS en el cliente (slice 4).

## Implementación

### Bloque CREAR
- [ ] Entidades y migración; repositorios; servicio de verificación de ubicación.

### Bloque MODIFICAR
- [ ] Ajustes del modelo de usuario para la adscripción a UA (del slice 1).

### Bloque ENLAZAR
- [ ] Endpoints REST, Actions y pantalla del panel; verificación con curl.

## Entrega

1. `feat/slice-2-ua-geocerca` desde `QA` → PR a `QA` → **testing**.
2. `Testing/asistencias-slice-2-testing`; ajustes `fix/*`.
3. PR a `staging` → PR a `main`.

## Criterios de Aceptación

- [ ] Una coordenada dentro de la geocerca se acepta; fuera, se rechaza con motivo.
- [ ] Una precisión peor que el límite configurado se rechaza.
- [ ] Solo `ADMIN` crea o edita UA y geocercas, sin recompilar.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-roles-permisos-wiki]]
- [[asistencias-slice-1-ingreso-roles-sub-plan]]
- [[asistencias-slice-4-checada-valida-sub-plan]]
