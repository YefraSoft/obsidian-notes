---
tipo: sub-plan
titulo: Slice 4 — Sincronización con el Core
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: pendiente
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - core
  - sincronizacion
  - slice-4
---

# Slice 4 — Sincronización con el Core

> **Plan padre:** [[email-service-plan]]
> Mantener en PostgreSQL una copia local de los contactos del Core (MySQL, solo lectura) como base para grupos y envíos.

## Pendientes

- [ ] Definir tablas, campos y poblaciones del Core a sincronizar.
- [ ] Lector del Core con credencial en OpenBao.
- [ ] Job periódico y disparo manual.
- [ ] *Upsert* respetando los campos locales.
- [ ] Pantalla de estado en el panel.

## Estado Actual

| Área | Estado |
|------|--------|
| Tabla `contacts` con `core_id`, `source`, `subscribed`, `unsubscribed_*`, `deleted_at` | ✅ |
| `contacts.source` (`CORE` / `LOCAL`) y `core_id` nullable | ✅ |
| Conexión al Core, job de sincronización y endpoints | ⏳ |
| Panel | ⏳ |

## Alcance

Cubre la lectura del Core y la copia local, con su estado y su ejecución manual. No cubre la gestión de contactos (Slice 5) ni los grupos (Slice 6). La sincronización **nunca** sobrescribe los campos locales (`subscribed`, `unsubscribed_at`, `unsubscribed_reason`).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: un puerto para leer el Core (`ICoreContactSource`) con su implementación MySQL en `Infrastructure`; el job periódico es un adaptador que llama a un servicio de sincronización. El estado de la última ejecución se consulta por REST.

## Implementación

### Paso 1 — Base de datos
- [ ] Registro de la última ejecución (fecha, cantidades, error) si se quiere mostrar en el panel.

### Paso 2 — API
- [ ] Conector MySQL de solo lectura; cadena en OpenBao (`CORE_DB__CONNECTION`) y verificación en `/health/ready`.
- [ ] `BackgroundService` con intervalo configurable en `appsettings`.
- [ ] *Upsert* por `core_id`; baja lógica (`deleted_at`) de los que desaparecen del Core.
- [ ] `POST` para disparar la sincronización y `GET` de estado (Admin).

### Paso 3 — Panel
- [ ] Estado de la última sincronización y botón "Sincronizar ahora".

### Paso 4 — Documentación
- [ ] README: variables del Core y comportamiento de la sincronización.

## Criterios de Aceptación

- [ ] Una segunda sincronización sin cambios en el Core no modifica filas.
- [ ] Una baja local de un contacto sobrevive a una sincronización.
- [ ] Un contacto que desaparece del Core queda con `deleted_at` y no se pierde su historial.
- [ ] Si el Core no responde, la API sigue funcionando y el estado lo refleja.
- [ ] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- ¿Qué tablas y campos del Core se usan (nombre, correo, tipo de persona)?
- ¿Con qué frecuencia se sincroniza?
- ¿Qué poblaciones entran (alumnos, docentes, administrativos) y cómo se distinguen para armar grupos después?

## Relacionado

- [[email-service-plan]]
- [[email-service-contactos-sub-plan]]
