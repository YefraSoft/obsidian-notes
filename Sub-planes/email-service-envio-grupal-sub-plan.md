---
tipo: sub-plan
titulo: Slice 8 — Envío a grupos y lotes
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-11-09
estado: implementado
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
> El slice envía una notificación a todos los contactos elegibles de un grupo. Cada envío forma un lote que lleva el progreso, y cada mensaje del lote conserva su propio estado.

## Pendientes

- [x] Endpoint de envío a grupo, por REST y por gRPC (`SendToGroup`).
- [x] Lote con foto de los miembros (total fijo).
- [x] Cola `notificacion.email.grupal` y su consumidor (expansión del lote).
- [x] Cierre del lote, reconciliador y control de duplicados.
- [x] Avance en vivo (SSE y pub/sub de Valkey) y pantallas del panel.
- [x] Documentación: AGENTS.md, wiki y README.
- [ ] Deploy a staging: aplicar el `ALTER` de `batches` y publicar. Lo hace el responsable.
- [ ] Fase de dudas y refinamiento.

## Estado Actual

| Área | Estado |
|------|--------|
| Tablas `batches` (`id_consumer` nulo para el Admin, índice `ix_batches_open`) y `messages_batch` | ✅ |
| `GroupPermissions.RequireSendAsync` y `TemplatePermissions.RequireUseAsync(..., GROUP)` | ✅ |
| `POST groups/{id}/messages`, `GET groups/assigned`, `GET batches/{id}`, GraphQL `batches`, gRPC `SendToGroup` | ✅ |
| Cola grupal, expansión, cierre atómico y `BatchReconciler` | ✅ |
| Avance en vivo `GET batches/{id}/progress` (SSE) con el proxy del panel | ✅ |
| Panel: "Enviar a grupo" del consumidor con avance en vivo y pestaña "Lotes" del Admin | ✅ |
| Verificado en local con Mailpit (lotes de 2036 contactos) | ✅ |
| Deploy a staging | ⏳ |

## Alcance

Cubre el envío a un grupo y el seguimiento de su lote. Reutiliza el worker del Slice 7.

No cubre:
- los envíos periódicos;
- los grupos dinámicos (roadmap);
- el consumo concurrente (siguiente versión, ver "Mejoras").

## Decisiones (2026-11-06)

- **Detalle y lista de lotes:** solo los ve el Admin. El GroupManager no los ve.
- **Avance del consumidor:** ve en vivo el avance de sus propios lotes, como filas `[correo][estado]`, sin ids ni texto de error. Esto cambia la regla anterior de que "el consumidor no ve estados".
- **Transporte del avance:** SSE con pub/sub de Valkey (canal `batch:<id>`). El servidor de Astro reenvía el flujo al navegador.
- **Envíos del Admin:** el Admin puede enviar a cualquier grupo; sus lotes tienen `id_consumer` NULL.
- **Duplicados:** `Idempotency-Key` con ámbito propio, más una guarda "en curso" por huella mientras el lote siga abierto (409 `BatchInProgress`).
- **Cierre:** lo hace el worker tras cada entrega, y el reconciliador cada 60 s. Un lote que pasa de 60 min vence: sus pendientes pasan a `FAILED` con `batch_timeout`.

## Implementación (hecha)

Seguimos los pasos CREAR → MODIFICAR → ENLAZAR con un commit por paso:

1. **BD y `BatchStatus`:** el valor guardado es `IN-PROCESS`, con guion, así que la traducción al enum es explícita.
2. **Core:** ruta `email.grupal`, `SendGroupMessage`, `MessageIdentity.ForBatch` y `ForBatchMember`, huella `OfGroup` e `InBatchInProgressException`.
3. **Application:** puertos de lotes, `BatchPermissions`, `GroupSendService`, `BatchExpansionService`, `BatchCloser` y `BatchQueries`.
4. **Infrastructure:**
   - `BatchRepository`, con el cierre atómico en un solo UPDATE;
   - `RedisBatchGuard` y `RedisBatchProgress`;
   - `EmailGroupConsumer` y `BatchReconciler`.
5. **Ajustes:**
   - resolutor de variables compartido;
   - `BatchId` en `SendEmailMessage`;
   - en gRPC, la política va en cada método.
6. **API:** DI, rutas REST, SSE, GraphQL y `SendToGroup`. El avance funciona en cuanto se acepta el envío, aunque el worker todavía no haya creado el lote.
7. **Worker:** `BatchDeliveryTracker` publica el avance y cierra el lote tras cada entrega.
8. **Panel:**
   - `SendGroupModal` y `BatchProgressPanel`;
   - el proxy `/envios/lotes/[id]/progreso`;
   - la pestaña "Lotes";
   - después, un refactor en hooks y subcomponentes.
9. **Documentación.**

## Errores encontrados

- **ER-033:** las direcciones con caracteres no ASCII antes de la arroba (`maria.muñoz@…`) no se enviaban y se reintentaban cuatro veces. Se corrigió con SMTPUTF8 (`DeliveryFormat = International`). Si el servidor no lo admite, el correo se trata como rechazo permanente. Lo certifican los tests MSU-001 a MSU-003 y MRS-012.

## Mejoras (siguiente versión)

- **Consumo concurrente:**
  - Hoy la cola individual procesa un mensaje a la vez y los lotes salen en orden FIFO.
  - Más adelante el servidor SMTP será **Stalwart**. Antes de subir la concurrencia hay que medir la capacidad del servicio y la de Stalwart, para no saturarlos.
- **Grupos grandes:** expandirlos por tandas en lugar de en una sola transacción.

## Criterios de Aceptación

- [x] Un grupo no asignado al consumidor da 403 con `code: ResourceNotGranted`. Una plantilla no asignada con ámbito `GROUP` también.
- [x] El total del lote queda fijo aunque cambien los miembros después.
- [x] Un envío duplicado en curso da 409 `BatchInProgress`.
- [x] El lote termina con el estado correcto y sus conteos (`FINISH`, `PARTIAL` y `FAILED` por vencimiento, verificados en local).
- [x] `dotnet build` correcto con `-warnaserror`; `bun run build`, `check` y `lint` correctos.

## Relacionado

- [[email-service-plan]]
- [[email-service-grupos-sub-plan]]
- [[email-service-envio-individual-sub-plan]]
