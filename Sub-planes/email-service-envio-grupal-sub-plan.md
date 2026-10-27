---
tipo: sub-plan
titulo: Slice 8 — Envío a grupos y lotes
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: pendiente
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - envio
  - grupos
  - lotes
  - slice-8
---

# Slice 8 — Envío a grupos y lotes

> **Plan padre:** [[email-service-plan]]
> Notificar a todos los contactos suscritos de un grupo, con un lote que lleva el progreso y cada mensaje con su propio estado.

## Pendientes

- [ ] Endpoint de envío a grupo.
- [ ] Lote con foto de los miembros.
- [ ] Cola `notificacion.email.grupal` y su consumidor.
- [ ] Cierre del lote y control de duplicados.
- [ ] Pantalla de progreso.

## Estado Actual

| Área | Estado |
|------|--------|
| Tablas `batches` y `messages_batch` | ✅ |
| `GroupPermissions.RequireSendAsync` y `TemplatePermissions.RequireUseForGroupAsync` | ✅ |
| Rol `GroupSender` con su política | ✅ |
| Endpoint, cola grupal, cierre de lote y reconciliador | ⏳ |
| Panel | ⏳ |

## Alcance

Cubre el envío a un grupo y el seguimiento de su lote. Reutiliza el worker del Slice 7. No cubre envíos periódicos ni grupos dinámicos (roadmap).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]] y los mismos puertos del Slice 7. La API valida y publica un único mensaje grupal; **el worker reparte el lote** (foto de miembros, mensajes y cierre) y descarta duplicados por `MessageId`. La consulta de lotes es interna: solo `GroupManager` y Admin.

## Implementación

### Paso 1 — Base de datos
- [ ] Revisar `batches` (`total`, `status`, `sent_count`, `failed_count`, `finished_at`) y la restricción de un contacto por lote.

### Paso 2 — API
- [ ] `POST /groups/{id}/messages` (`GroupSender`): exige el rol, el grupo asignado y la plantilla asignada.
- [ ] El worker toma la foto de miembros activos y suscritos (las plantillas transaccionales ignoran la baja) y fija el total del lote al recibir el mensaje grupal.
- [ ] Rechazar con 409 el envío idéntico evidente; el worker descarta los duplicados por `MessageId`.
- [ ] Consulta interna de lotes y su progreso (lista por GraphQL, GET por id por REST) solo para `GroupManager` y Admin.

### Paso 3 — Worker
- [ ] Declarar `notificacion.email.grupal` y consumirla con el mismo envío.
- [ ] Cerrar el lote como `FINISH`, `FAILED` o `PARTIAL` con sus conteos.
- [ ] Reconciliador de lotes que quedaron abiertos.

### Paso 4 — Panel
- [ ] Enviar a un grupo y ver el avance del lote.

### Paso 5 — Documentación
- [ ] README: contrato del envío a grupos y estados del lote.

## Criterios de Aceptación

- [ ] Un grupo no asignado al consumidor da 403 con `code: ResourceNotGranted`; una plantilla no asignada también.
- [ ] El total del lote queda fijo aunque cambien los miembros después.
- [ ] Un envío duplicado en curso da 409.
- [ ] El lote termina con el estado correcto y sus conteos.
- [ ] `dotnet build` y `bun run build` correctos.

## Relacionado

- [[email-service-plan]]
- [[email-service-grupos-sub-plan]]
- [[email-service-envio-individual-sub-plan]]
