---
tipo: sub-plan
titulo: "Fase 10 — Alertas y notificaciones"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: en-progreso
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - notificaciones
  - alertas
  - fase-10
---

# Fase 10 — Alertas y notificaciones

> **Plan padre:** [[rh-cotla-plan]] — plan general del proyecto RH Cotla, con stack, roles y estado por fase.
> Bandeja de notificaciones, correo de resoluciones, alertas diarias de cumpleaños y aniversarios (`docs/especificaciones.md` §10) y eventos en tiempo real para el panel. Código completo en las tres apps; falta confirmar en servidor el correo SMTP y el stream por el tunnel. Push móvil descartado.

## Pendientes

- [ ] Verificar en el servidor, tras desplegar: envío SMTP real y stream SSE a través del tunnel de Cloudflare (FX-BE-7).
- [ ] Integrar FCM / push móvil (F-BE-6): descartado por ahora; se conserva `dispositivos_push`.
- [ ] Eventos de horas extra hacia el panel y bus Redis detrás de `PanelBroker` si hay más de una instancia (F-BE-5, FX-BE-8).
- [ ] Confirmar con el cliente el canal definitivo de notificaciones (F-BE-8, §31).
- [ ] Decidir si `POST /users/{id}/password-reset` responde 503 sin SMTP (FX-BE-5).
- [ ] Panel: entrada de menú «Ayuda» deshabilitada; habilitarla con el manual incrustado (F-FE-1).

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | Completo, sin migraciones, sin push: bandeja, correo de resoluciones, alertas diarias y SSE. `NotificationsEndpointsTests` (10), `AlertasTests` (6), `PanelEventsTests` (8); suite 401/401 |
| Panel web | Hecho: campana con contador, `/notificaciones`, eventos en vivo por SSE (`components/layout/panel-eventos-provider.tsx`, `lib/panel-eventos.ts`) |
| App móvil | Hecho: `src/app/notificaciones.tsx`, `src/lib/api-notifications.ts`, badge de no leídas en «Más» |
| Despliegue | Código listo; el SMTP real y el stream por el tunnel no están verificados (FX-BE-7) |

## Alcance

Cubre la bandeja por usuario, el correo de aprobaciones y rechazos, las alertas del día y los eventos SSE del panel. No cubre push (FCM), buzón interno, ni notificaciones por corrección de marcación (§10 no las pide). El SSE es una extensión pedida por el usuario, fuera de §10 y §12.

## Decisiones

1. **Canal:** bandeja por API + correo SMTP solo de resoluciones + SSE para el panel. El móvil se entera por correo y consultando la bandeja; push descartado por ahora.
2. **Tipos:** `solicitud_ausencia_aprobada|rechazada`, `hora_extra_aprobada|rechazada` (el rechazo lleva el motivo), `cumpleanos_propio|equipo`, `aniversario_propio|equipo`. Se crean en la misma transacción que el cambio de estado.
3. **Correo:** solo las resoluciones (`TiposNotificacion.ConCorreo`). El despachador corre cada 60 s (`Notificaciones:IntervaloSegundos`), envía las no enviadas de las últimas 24 h, marca `enviada_at`; sin correo del colaborador se marca atendida y ante fallo SMTP se reintenta. Si SMTP no está configurado, se omite el envío con aviso y se marca enviada. `EnvioCorreoHabilitado=false` lo apaga.
4. **Alertas del día:** `AlertasHostedService` revisa cada 15 min y genera una vez al día desde `Notificaciones:Alertas:HoraLocal` (07:00); notifica al colaborador, a RH y a los gerentes de su equipo (sin duplicar al propio). Idempotente por (usuario, tipo, empleado, fecha). Los «próximos» no notifican: van en `GET /dashboard`.
5. **SSE del panel:** solo `rh` y `gerente`; `Authorization: Bearer` (no se acepta token en la URL, por eso el panel usa `fetch` y no `EventSource`). Evento `latido` cada 25 s (`Notificaciones:PanelLatidoSegundos`), máximo 5 conexiones por usuario, sin repetición de eventos perdidos. Pub/sub en memoria (una sola instancia).
6. **Eventos publicados:** `asistencia`, `solicitud_ausencia`, `correccion_asistencia`. Horas extra no emiten evento.
7. **Dispositivos:** `POST /notifications/devices` guarda el token (upsert); no se envía nada.
8. **Sugerencia móvil:** `GET /notifications/unread-count` al abrir y cada ~60 s en primer plano.

## Implementación

### Backend (`app-backend/cotla-api/`)
- Módulo `Modules/Notifications` (`NotificacionService`, `NotificacionesHostedService`, `AlertasHostedService`, `PanelBroker`).
- `GET /api/v1/notifications?leida=&page=&pageSize=` — autenticado, solo las propias.
- `GET /api/v1/notifications/unread-count` — autenticado; `{ noLeidas }`.
- `POST /api/v1/notifications/{id}/read` y `/unread` — autenticado; 404 si no es del usuario.
- `POST /api/v1/notifications/read-all` — autenticado; `{ marcadas }`.
- `POST /api/v1/notifications/devices` y `DELETE /api/v1/notifications/devices/{id}` — autenticado.
- `POST /api/v1/notifications/alerts/generate` — solo `rh`; 403 a los demás.
- `GET /api/v1/panel/events` — `rh` y `gerente`; 403 colaborador, 401 sin token.

### Panel web (`app-frontend/`)
- Campana del encabezado: `GET /notifications/unread-count` al abrir, cada 60 s, con cada evento y al cambiar la lectura.
- `/notificaciones`: todas o sin leer, marcar leída/no leída, marcar todas, enlace según `datos`; RH tiene «Generar alertas de hoy».
- Eventos en vivo: una conexión SSE por pestaña, reconexión con espera creciente; cada evento recarga listados, dashboard y campana.

### App móvil (`app-movil/`)
- `src/app/notificaciones.tsx` (ruta `/notificaciones`): lista con estado; tocar una notificación alterna leída/no leída.

## Criterios de Aceptación

- [x] El colaborador recibe aviso en la bandeja al aprobar o rechazar una ausencia u hora extra.
- [x] RH y gerentes reciben alertas y eventos de su alcance (el gerente solo de su equipo).
- [x] Una marcación reenviada o rechazada no publica evento; un fallo al publicar no rompe la operación.
- [x] Panel y móvil consumen la bandeja con contador de no leídas.
- [ ] El correo de resolución se envía por SMTP real (no verificado, FX-BE-7).
- [ ] El stream SSE funciona a través del tunnel de Cloudflare (no verificado, FX-BE-7).
- [ ] El cliente confirma el canal definitivo (F-BE-8).

## Pendientes para otras fases

- [ ] FX-BE-7, FX-BE-5 y FX-BE-8 se cierran en [[rh-cotla-fase-12-despliegue-sub-plan]] al verificar el servidor.
- [ ] Rate limit general y revisión de `/diagnostics/*` (FX-BE-3), en la Fase 11.

## Relacionado

- [[rh-cotla-plan]] — plan padre con el estado general.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos del backend; §25 notificaciones y §26 SSE.
- [[rh-cotla-integracion-clientes-manual]] — contrato para web y móvil; secciones «Notificaciones» y «Panel: eventos en tiempo real».
- [[rh-cotla-pendientes-sub-plan]] — deuda y trabajo futuro (F-BE-5, F-BE-6, F-BE-8, FX-BE-5, FX-BE-7, FX-BE-8).
- [[rh-cotla-fase-09-dashboards-calendario-sub-plan]] — fase anterior: «próximos» cumpleaños y aniversarios en el dashboard.
- [[rh-cotla-fase-06-ausencias-vacaciones-sub-plan]] — origen de las resoluciones de ausencias que notifican.
- [[rh-cotla-fase-07-horas-extra-sub-plan]] — origen de las resoluciones de horas extra que notifican.
- [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]] — fase siguiente: seguridad y Redis.
- [[rh-cotla-fase-12-despliegue-sub-plan]] — despliegue y verificación en servidor.
