---
tipo: sub-plan
titulo: Slice 7 — Envío 1 a 1 y worker SMTP
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
  - smtp
  - rabbitmq
  - slice-7
---

# Slice 7 — Envío 1 a 1 y worker SMTP

> **Plan padre:** [[email-service-plan]]
> Enviar un correo individual con una plantilla, de punta a punta: petición, cola, SMTP y estado.

## Pendientes

- [ ] Endpoint de envío con `Idempotency-Key`.
- [ ] Worker SMTP real en lugar del handler provisional.
- [ ] Estados, reintentos y cola de fallidos.
- [ ] Consulta interna de mensajes (solo `GroupManager` y Admin) y pantalla del panel.

## Estado Actual

| Área | Estado |
|------|--------|
| Publicador RabbitMQ con confirmaciones y `mandatory` | ✅ |
| Consumidor de `notificacion.email.individual` con ack manual | ✅ |
| `LoggingEmailIndividualHandler` (solo escribe en el log) | ⚠ provisional |
| Tabla `messages` y rol `Sender` (política y `MessagePermissions`) | ✅ |
| Endpoint de envío, render y SMTP | ⏳ |
| Panel | ⏳ |

## Alcance

Cubre el envío 1 a 1 y el seguimiento de su estado hasta `SENT` o `FAILED`. No cubre lotes ni grupos (Slice 8) ni la apertura y las bajas (Slice 9). Solo se guardan metadatos: no se guarda el cuerpo del correo ni las variables.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]] (regla 8, diagrama D6): la API **valida y publica** → 202 con un `MessageId` derivado del consumidor y la `Idempotency-Key`; **el worker es dueño del registro**: lo crea `QUEUED`, envía y deja el estado. El consumidor de RabbitMQ solo llama a `IMessageDispatchService` (Application); `IEmailSender` es un puerto con implementación SMTP. Las aplicaciones consumidoras **no conocen estados internos**; `messages/*` es consulta interna solo para `GroupManager` y Admin.

## Implementación

### Paso 1 — Base de datos
- [ ] Revisar `messages` (estado, `last_error`, reintentos, fechas). El `MessageId` derivado de consumidor + `Idempotency-Key` es la clave: el worker inserta solo si no existe.

### Paso 2 — API
- [ ] `MessageController` y tipo GraphQL en `Api`; `MessageService`, `MessageQueries`, contratos y builder en `Application/Messages`; política `Sender`.
- [ ] `POST` de mensaje (`to`, `template`, `variables`) con `Idempotency-Key`; valida rol, plantilla activa y alcance, y calcula el `MessageId`.
- [ ] Publicar a `email.individual` y devolver 202 con el `MessageId` (la API no escribe el registro).
- [ ] Consulta interna `messages/*` (lista por GraphQL, GET por id por REST) solo para `GroupManager` y Admin; los consumidores no consultan estados.

### Paso 3 — Worker
- [ ] Reemplazar `LoggingEmailIndividualHandler` por el envío real (render + SMTP; credenciales en OpenBao).
- [ ] Crear el registro `QUEUED` al recibir el mensaje (ignora duplicados por `MessageId`).
- [ ] `QUEUED → SENT`; si falla, `RETRYING` con espera creciente y finalmente `FAILED` con el motivo.
- [ ] Cola de fallidos en lugar de descartar el mensaje.

### Paso 4 — Panel
- [ ] Pantalla de envío de prueba e historial de mensajes.

### Paso 5 — Documentación
- [ ] README: contrato del endpoint y estados.

## Criterios de Aceptación

- [ ] Repetir la petición con el mismo `Idempotency-Key` devuelve el mensaje original sin enviar otro correo.
- [ ] Un consumidor sin el rol `Sender` recibe 403 con `code: RoleNotAllowed`.
- [ ] Un correo enviado queda `SENT`; si el SMTP falla se reintenta y termina en `FAILED` con el motivo.
- [ ] Con RabbitMQ caído la API sigue arriba y se recupera sola.
- [ ] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- ¿Qué proveedor SMTP y qué remitente se usan (en `news-service` existe una implementación de referencia)?
- ¿Se permiten adjuntos?
- ¿Cuál es el máximo de reintentos y la espera entre ellos?

## Relacionado

- [[email-service-plan]]
- [[email-service-plantillas-sub-plan]]
- [[email-service-envio-grupal-sub-plan]]
