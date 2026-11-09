---
tipo: sub-plan
titulo: Slice 9 — Seguimiento y bajas
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-11-09
estado: pendiente
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - seguimiento
  - bajas
  - migraciones
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
- [ ] Migraciones automáticas con EF Core (decisión del 2026-11-09): reemplazan los `ALTER` a mano.

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

### Paso 1 — Base de datos y migraciones con EF
- [ ] Revisar índices de consulta por consumidor y fecha.
- [ ] **Migraciones automáticas con EF Core.** Cambia la regla 9 del plan ("sin migraciones").
  - Crear una migración base (`dotnet ef migrations add Inicial`) que reproduzca exactamente el esquema actual de `infra/db.sql`, incluidos los cambios del slice 8 (`batches.id_consumer` nulo e `ix_batches_open`). Comparar el script generado (`dotnet ef migrations script`) con `db.sql` antes de darla por buena.
  - Marcar esa migración como aplicada, sin ejecutarla, en las bases que ya existen (local, staging y producción): insertar su fila en `__EFMigrationsHistory` con un script idempotente que entrega Efraín.
  - Decidir quién aplica las migraciones siguientes. Opciones:
    1. La API al arrancar (`Database.MigrateAsync`), con un candado para varias réplicas.
    2. Un paso del pipeline de CI antes del deploy.
    3. Un script `migrations script --idempotent` que aplica Efraín.

    Ver la pregunta abierta.
  - Conciliar con el scaffold. Hoy las entidades salen de la base por scaffold (database-first) y hay que restaurar el constructor de `EmailDbContext`. Con migraciones, el modelo de EF pasa a ser la fuente del esquema (code-first) y el scaffold deja de usarse.
  - `db.sql` deja de ser la fuente de verdad: se conserva como referencia generada o se elimina. Hay que actualizar `AGENTS.md`, la wiki (Despliegue y configuración, Modelo de datos) y el `Makefile` (`make down-volumes` y el `initdb` de Postgres).
  - Verificar: base local nueva creada solo con migraciones, y una base existente que, tras marcar la base, aplica una migración de prueba sin perder datos.

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
- [ ] Una base vacía queda con el esquema completo solo con las migraciones de EF, y las bases existentes las adoptan sin perder datos.
- [ ] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- (Resuelta) La consulta de mensajes es interna: solo `GroupManager` y Admin.
- ¿Cuánto tiempo se conservan los mensajes consultables?
- ¿Quién aplica las migraciones de EF: la API al arrancar, el pipeline o Efraín con un script idempotente? Producción la administra Efraín.

## Relacionado

- [[email-service-plan]]
- [[email-service-envio-individual-sub-plan]]
- [[email-service-sync-core-sub-plan]]
