---
tipo: wiki
titulo: "Asistencias — Reglas de negocio"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-17
estado: vigente
proyecto: asistencias
tags:
  - wiki
  - asistencias
  - reglas
---

# Asistencias — Reglas de negocio

> Catálogo de reglas vigentes de la v1 del sistema de asistencias. Plan maestro: [[asistencias-plan]].

## Contenido

- [[#1. Checada válida]]
- [[#2. Horarios y bloques]]
- [[#3. Incidencias]]
- [[#4. Rechazos e intentos]]
- [[#5. Roles]]
- [[#6. Fuera de la v1]]

## 1. Checada válida

Una checada es válida solo si se cumplen las tres condiciones:

1. **Ubicación:** el GPS del dispositivo está dentro de la geocerca de la UA del personal (con la tolerancia configurada para esa UA).
2. **Evidencia en vivo:** el video/foto lo toma la cámara del dispositivo; no existe opción de subir archivos. Incluye un reto de gestos (MediaPipe) para descartar una fotografía.
3. **Identidad:** el embedding calculado en el cliente coincide con el embedding de referencia de la persona; la API lo verifica.

El procesamiento pesado ocurre en el cliente y el backend confía en esos datos; solo se verifica la comparación de embeddings, la geocerca y las reglas.

## 2. Horarios y bloques

- Administrativos: horario corrido (un bloque por día).
- Docentes: bloques variables y separados en el día (p. ej. 7:00-9:00 y 22:00-24:00); cada bloque tiene su entrada y su salida.
- Los horarios se migran del Core en el primer ingreso de cada persona. `RH_DIR` administra horarios; si el panel completo entra en la v1 o en la v2 está por confirmar.

## 3. Incidencias

Se evalúan **por bloque**, no por día completo:

| Incidencia | Regla |
|------------|-------|
| Retardo | Entrada a partir de 15 minutos después de la hora del bloque |
| Falta | Entrada a partir de 1 hora después de la hora del bloque |
| Falta de entrada o salida (olvido) | No hay la checada esperada del bloque |
| Salida anticipada | Salida antes de la hora de fin del bloque |
| Ausencia del día | Ninguna checada en el día con horario asignado |

Un docente puede acumular varias faltas el mismo día (p. ej. llegó tarde por la mañana o no llegó, pero asistió en el bloque vespertino). Los días inhábiles, permisos, incapacidades y comisiones no se consideran en la v1.

## 4. Rechazos e intentos

- La persona puede reintentar de inmediato.
- Cada intento rechazado se registra con su motivo (fuera de la geocerca, rostro no coincide, reto no superado, mala foto) como evidencia y auditoría.

## 5. Roles

Cuatro niveles, guardados en `new-core` y ligados por usuario (detalle en [[asistencias-roles-permisos-wiki]]):

- **Nivel 3 — Docente o Administrativo (`CREATE_OWN`):** envía su checada y crea su "registro"; solo ve sus propios datos.
- **Nivel 2 — `RH_ENLACE` (`SEE_OWN`):** ve los datos de su UA.
- **Nivel 1 — `RH_DIR` (`SEE_ALL`, `EDIT_ALL`):** ve todas las checadas, rechaza por inconsistencia y administra horarios.
- **Nivel 0 — `ADMIN` (`ALL`):** control total del sistema.

## 6. Fuera de la v1

Días inhábiles, permisos, incapacidades, comisiones, panel de horarios, checada sin conexión y app móvil: v2.

## Índice / Tabla de contenidos

- [[#1. Checada válida]]
- [[#2. Horarios y bloques]]
- [[#3. Incidencias]]
- [[#4. Rechazos e intentos]]
- [[#5. Roles]]
- [[#6. Fuera de la v1]]

## Relacionado

- [[asistencias-plan]]
- [[asistencias-slice-5-horarios-incidencias-sub-plan]]
- [[asistencias-slice-4-checada-valida-sub-plan]]
