---
tipo: sub-plan
titulo: "Asistencias — Slice 6: Notificaciones"
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
  - notificaciones
---

# Asistencias — Slice 6: Notificaciones

> **Plan padre:** [[asistencias-plan]] · **Rama:** `feat/slice-6-notificaciones`
> Correos a la persona mediante `email-service`, desde una tabla outbox y un servicio en segundo plano (sin RabbitMQ). Referencia: [[email-service-plan]].

## Pendientes

- [ ] Detallar el slice con el responsable y crear sus sub-tareas en Plane.
- [ ] Definir plantillas y variables de cada notificación.
- [ ] Definir el consumidor (key y roles) de Asistencias en email-service y guardar su credencial en OpenBao.
- [ ] Definir de dónde se toma el correo de la persona (se migra del Core en el primer ingreso).
- [ ] Definir envío por gRPC o REST según el volumen y los reintentos de la outbox.

## Estado Actual

| Área | Estado |
|------|--------|
| Requisitos | ✅ Definidos |
| Diseño técnico | ✅ Outbox + servicio en segundo plano |
| Implementación | ⏳ Pendiente |

## Alcance (vertical)

- **BD:** tabla `outbox` en `new-core` (tipo, estado, intentos, destino).
- **API:** escritura en la outbox en la misma transacción que la checada, el intento o la incidencia; servicio en segundo plano con reintentos, `pg_advisory_lock` entre réplicas y marca de enviado; puerto `INotificationSender` y cliente de email-service.
- **Notificaciones (v1):** confirmación de checada, checada rechazada o incidencia, y aviso de olvido de checada. Sin resúmenes a RH.
- **Pantalla:** estado de envíos para `ADMIN` (opcional) y configuración de plantillas en email-service.
- **Docs:** nota de testing.

## Implementación

### Bloque CREAR
- [ ] Tabla outbox y migración; `INotificationSender`; servicio en segundo plano.

### Bloque MODIFICAR
- [ ] Escribir en la outbox desde los servicios de checada (slice 4) e incidencias (slice 5).

### Bloque ENLAZAR
- [ ] Alta del consumidor y plantillas en email-service; verificación con Mailpit en local.

## Entrega

1. `feat/slice-6-notificaciones` desde `QA` → PR a `QA` → **testing**.
2. `Testing/asistencias-slice-6-testing`; ajustes `fix/*`.
3. PR a `staging` → PR a `main`.

## Criterios de Aceptación

- [ ] Una falla de email-service no impide registrar la checada.
- [ ] Cada aviso se envía una sola vez (idempotencia con la clave de la fila).
- [ ] Un aviso fallido se reintenta y deja el motivo registrado.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-arquitectura-wiki]]
- [[asistencias-slice-4-checada-valida-sub-plan]]
- [[asistencias-slice-5-horarios-incidencias-sub-plan]]
- [[email-service-plan]]
