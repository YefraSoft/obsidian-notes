---
tipo: sub-plan
titulo: Slice 9 — Seguimiento y bajas
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: pendiente
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - seguimiento
  - bajas
  - slice-9
---

# Slice 9 — Seguimiento y bajas

> **Plan padre:** [[email-service-plan]]
> Saber si el destinatario abrió el correo, permitirle darse de baja y consultar el estado de los mensajes.

## Pendientes

- [ ] Píxel de apertura (`SEEN`).
- [ ] Enlace de baja.
- [ ] Consulta interna de mensajes (solo `GroupManager` y Admin).
- [ ] Pantalla de seguimiento.

## Estado Actual

| Área | Estado |
|------|--------|
| Estados de mensaje en la base (`QUEUED`, `SENT`, `RETRYING`, `FAILED`, `SEEN`) | ✅ |
| Campos locales de baja en `contacts` | ✅ |
| Píxel, enlace de baja y consultas | ⏳ |
| Panel | ⏳ |

## Alcance

Cubre apertura, baja y consulta de estado. No cubre webhooks de entrega o rebote ni seguimiento de clics (roadmap). `SEEN` es un indicador aproximado: algunos clientes precargan imágenes y otros las bloquean.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]]: el píxel y la baja son endpoints públicos delgados que llaman a un servicio; la consulta de mensajes es **interna** (solo `GroupManager` y Admin), con la lista por GraphQL y el GET por id por REST.

## Implementación

### Paso 1 — Base de datos
- [ ] Nada nuevo previsto; revisar índices de consulta por consumidor y fecha.

### Paso 2 — API
- [ ] Píxel público con token firmado (`TRACKING_SECRET` en OpenBao); `SEEN` solo se alcanza desde `SENT`.
- [ ] Enlace de baja con token firmado que marca la baja local; las plantillas transaccionales la ignoran.
- [ ] Consulta interna `messages/*` solo para `GroupManager` y Admin (lista por GraphQL, GET por id por REST).
- [ ] Rate limit propio para los endpoints públicos.

### Paso 3 — Panel
- [ ] Estado por mensaje y por lote, y totales de apertura.

### Paso 4 — Documentación
- [ ] README: tracking, bajas y sus límites.

## Criterios de Aceptación

- [ ] Abrir el píxel de un mensaje `SENT` lo pasa a `SEEN`; un token alterado no cambia nada.
- [ ] Un contacto dado de baja deja de recibir los envíos no transaccionales y esa baja sobrevive a la sincronización con el Core.
- [ ] Un consumidor sin el rol `GroupManager` recibe 403 en `messages/*`; `GroupManager` y Admin los consultan.
- [ ] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- (Resuelta) La consulta de mensajes es interna: solo `GroupManager` y Admin.
- ¿Cuánto tiempo se conservan los mensajes consultables?

## Relacionado

- [[email-service-plan]]
- [[email-service-envio-individual-sub-plan]]
- [[email-service-sync-core-sub-plan]]
