---
tipo: sub-plan
titulo: "Fase 6 — Ausencias y vacaciones"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: en-progreso
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - fase-6
  - ausencias
  - vacaciones
---

# Fase 6 — Ausencias y vacaciones

> **Plan padre:** [[rh-cotla-plan]] — plan general de RH Cotla con el stack y el orden de las fases.
> Solicitudes de vacaciones, permisos e incapacidades con validación de saldo, aprobación de RH y libro mayor de vacaciones (`docs/especificaciones.md` §7).

## Pendientes

- [ ] **F-BE-1** Manejo de colisiones en solicitudes de permisos (BE+FE). Hoy solo existe el `409` por traslape propio; ver [[rh-cotla-pendientes-sub-plan]].
- [ ] Devengo automático de vacaciones (D1): diferido; decidir con el cliente (**F-BE-8**).
- [ ] Confirmar en dispositivo el flujo de incapacidad con documento (cámara, 5 MB, 3 archivos).
- [ ] Documentación móvil: «cámara o galería» solo es cámara (**FX-DOC-1**).

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | ✅ completo (12-nov-2026), sin migraciones: `/leave-requests` y `/vacation-balances`. Pruebas: `LeaveRequestsTests`. Notificación in-app al aprobar o rechazar |
| Panel web | ✅ `/solicitudes` (lista, detalle con documentos, aprobar o rechazar) y `/vacaciones` (saldos y movimientos, de la Fase 3) |
| App móvil | ✅ `solicitud-nueva.tsx`, `solicitudes.tsx` (con motivo de rechazo y cancelación) y `vacaciones.tsx` (saldo y movimientos) |
| Despliegue | ✅ backend en `api.yefrasoft.com` desde el 13-nov-2026 |

## Alcance

**Cubre:** crear, listar, cancelar, aprobar y rechazar solicitudes; documentos de incapacidad; descuento de saldo al aprobar con movimiento `uso`; consulta de saldos.

**No cubre:** devengo automático por antigüedad, medio día, cancelar una solicitud aprobada, ni el canal push o correo (ver [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]]). El saldo inicial y los ajustes de RH se configuran en [[rh-cotla-fase-03-configuracion-tienda-sub-plan]].

## Decisiones

1. **Cuatro tipos.** `vacaciones` (con saldo, sin documento), `permiso_con_goce` y `permiso_sin_goce` (sin saldo ni documento), `incapacidad` (documento obligatorio, confirmado).
2. **Fechas.** Enteras, no pasadas, `fin >= inicio`; días naturales (`fin − inicio + 1`) calculados por el servidor (`diasSolicitados`). Motivo opcional, máx. 255.
3. **El saldo no se reserva al crear.** Vacaciones exige saldo y que alcance (`400`); al aprobar se revalida y descuenta en transacción con `pg_advisory_xact_lock` por colaborador (`409` si ya no alcanza o no está pendiente).
4. **Movimiento `uso` en días negativos** ligado a la solicitud; `/vacation-balances` se conserva como consulta de saldos y movimientos (no se crea `/vacations`).
5. **Documentos:** JPEG, PNG o PDF verificados por firma, 5 MB y 3 archivos (`Ausencias:{DocumentoMaxBytes=5242880, MaxDocumentos=3}`); si la transacción falla se borran los ya subidos.
6. **Traslape:** `409` si hay otra solicitud propia `pendiente` o `aprobada` que se cruza (defensa que no está en la especificación).
7. **Estados:** `pendiente` → `aprobada` | `rechazada` (motivo obligatorio) | `cancelada`; cancelar solo el dueño y solo `pendiente`.
8. **Notificación in-app** al aprobar o rechazar (`solicitud_ausencia_aprobada` / `solicitud_ausencia_rechazada`); la consulta y el canal son de la Fase 10.
9. **Auditoría:** `crear_`, `cancelar_`, `aprobar_` y `rechazar_solicitud_ausencia`, con el saldo antes y después al aprobar vacaciones.

## Implementación

### Backend (`app-backend/cotla-api`)
- [x] Endpoints (todos requieren `Bearer`; filtros sin valores vacíos):
  - `POST /api/v1/leave-requests` — cualquier usuario, para sí mismo; `multipart/form-data` (`tipo`, `fechaInicio`, `fechaFin`, `motivo?`, `documentos`); `201` en `pendiente`.
  - `GET /api/v1/leave-requests` y `/{id}` — `rh` todo, `gerente` sus equipos, `colaborador` lo suyo; filtros `estado`, `tipo`, `fechaDesde`, `fechaHasta`, `empleadoId`, `equipoId`.
  - `GET /api/v1/leave-requests/{id}/documents/{archivoId}` — mismo alcance; requiere `Bearer`.
  - `POST /api/v1/leave-requests/{id}/cancel` — solo el dueño.
  - `POST /api/v1/leave-requests/{id}/approve` y `/reject` — solo `rh`.
  - `GET /api/v1/vacation-balances` y `/{id}/movements` — saldos y libro mayor (rh y gerente; el colaborador, el propio).
- [ ] Devengo automático (D1) — diferido.

### Panel web (`app-frontend`)
- [x] `/solicitudes`: lista con filtros por estado, tipo, fechas, colaborador y equipo.
- [x] Detalle con documentos (descarga con `fetch` + `blob:`); aprobar y rechazar con motivo solo RH y solo `pendiente`; los `409` se muestran como error de la acción.
- [x] `/vacaciones`: saldos y movimientos, incluido el tipo `uso`.

### App móvil (`app-movil`)
- [x] `/solicitud-nueva` (`solicitud-nueva.tsx`, `src/lib/api-leave.ts`): multipart con `File` de `expo-file-system`; documento obligatorio en incapacidad tomado con la cámara; fechas validadas en el cliente, la autoridad es el servidor.
- [x] `/solicitudes` (`solicitudes.tsx`): lista con estado y motivo de rechazo (no hay pantalla de detalle aparte).
- [x] Cancelar solicitud pendiente con confirmación (`409` si ya fue revisada).
- [x] `/vacaciones` (`vacaciones.tsx`): saldo y movimientos.

## Criterios de Aceptación

- [x] Vacaciones sin saldo no se envían.
- [x] RH aprueba o rechaza con motivo y trazabilidad (auditoría).
- [x] El saldo refleja ajustes y consumos.
- [x] La cancelación solo procede en `Pendiente`.
- [x] `dotnet test cotla-api.slnx` sin fallas de esta fase (277 casos en verde al cerrarla).
- [ ] Colisiones de permisos tratadas más allá del traslape (F-BE-1).

## Pendientes para otras fases

- [ ] Canal push/correo y bandeja de notificaciones de la aprobación: ver [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]].
- [ ] Ausencias aprobadas en el calendario y colaboradores de licencia en el dashboard: ver [[rh-cotla-fase-09-dashboards-calendario-sub-plan]].
- [ ] Reporte de saldos y solicitudes: ver [[rh-cotla-fase-08-reportes-sub-plan]].

## Relacionado

- [[rh-cotla-plan]] — plan general del proyecto, stack y orden de las fases.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos implementados del backend; §14 (saldos) y §21 (esta fase).
- [[rh-cotla-integracion-clientes-manual]] — contrato de API para web y móvil, secciones «Solicitudes de ausencia» y «Saldos de vacaciones».
- [[rh-cotla-pendientes-sub-plan]] — deuda y trabajo futuro, incluidos F-BE-1, F-BE-8 y FX-DOC-1.
- [[rh-cotla-fase-05-incidencias-correcciones-sub-plan]] — fase anterior: incidencias y correcciones.
- [[rh-cotla-fase-07-horas-extra-sub-plan]] — fase siguiente: registro y aprobación de horas extra.
