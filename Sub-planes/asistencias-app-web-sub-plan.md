---
tipo: sub-plan
titulo: "Asistencias — App web (repartido en slices)"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-18
estado: completado
proyecto: asistencias
padre: "[[asistencias-plan]]"
tags:
  - plan
  - sub-plan
  - asistencias
  - obsoleto
---

# Asistencias — App web (repartido en slices)

> **Plan padre:** [[asistencias-plan]]
> Esta nota quedó **obsoleta** con el enfoque de vertical slices: cada slice incluye su propia pantalla. Se conserva hasta que el responsable autorice eliminarla.

## Dónde quedó cada parte

| Parte de la app web | Slice |
|---------------------|-------|
| Esqueleto Astro + React, página de la PoC | [[asistencias-slice-0-esqueleto-sub-plan]] |
| Login, aviso de privacidad y roles | [[asistencias-slice-1-ingreso-roles-sub-plan]] |
| Panel de UA y geocercas | [[asistencias-slice-2-ua-geocerca-sub-plan]] |
| Consentimiento y enrolamiento | [[asistencias-slice-3-enrolamiento-sub-plan]] |
| Pantalla de checada, captura, subida directa al CDN | [[asistencias-slice-4-checada-valida-sub-plan]] |
| Pantallas de horarios e incidencias | [[asistencias-slice-5-horarios-incidencias-sub-plan]] |
| Consulta, visor de evidencia (proxy de Astro) y reportes | [[asistencias-slice-7-consulta-reportes-sub-plan]] |

La comunicación sigue vigente: lo ligero por Astro Actions; el clip de evidencia directo del navegador al CDN; la evidencia se lee por un proxy en el servidor de Astro. Ver [[asistencias-arquitectura-wiki]].

## Relacionado

- [[asistencias-plan]]
- [[asistencias-arquitectura-wiki]]
