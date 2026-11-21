---
tipo: sub-plan
titulo: "Fase 5 — Incidencias y correcciones"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: en-progreso
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - fase-5
  - incidencias
  - correcciones
---

# Fase 5 — Incidencias y correcciones

> **Plan padre:** [[rh-cotla-plan]] — plan general de RH Cotla con el stack y el orden de las fases.
> Detectar marcas faltantes y marcaciones fuera de zona como incidencias, y gestionar las solicitudes de corrección de marcación sin alterar nunca la marcación original (`docs/especificaciones.md` §6).

## Pendientes

- [ ] **F-BE-4** «Detectar ahora» debe traer también las incidencias fuera de zona (BE+FE). Ver [[rh-cotla-pendientes-sub-plan]].
- [ ] Confirmar con el cliente las decisiones aplicadas por vacío de la especificación: RH no edita marcaciones directamente, un día sin marcaciones no es incidencia y la comida no genera incidencia (**F-BE-8**).
- [ ] Confirmar en dispositivo la pantalla `app-movil/src/app/correcciones.tsx`: el plan móvil la daba por pendiente y el código ya la contiene (**FX-DOC-1**).
- [ ] Sin cobertura de pruebas: no hay caso para el origen `correccion_pendiente` del reporte (vive en la Fase 8), y queda abierta la falla previa `AttendanceCorrectionsTests.Aprobar_conserva_la_marcacion_original…` por precisión de microsegundos al comparar fechas.

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | ✅ completo (12-nov-2026): `IncidenciaDetector`, `IncidenciasHostedService` (cada 30 min), `/incidents`, `/attendance-corrections` y migración `Fase5IncidenciasIdempotentes` (dos índices únicos parciales). Pruebas: `IncidentsEndpointsTests` y `AttendanceCorrectionsTests` |
| Panel web | ✅ `/incidencias` (filtros, resolver, descartar, «Detectar ahora») y `/correcciones` (aprobar o rechazar); el detalle de asistencia muestra `correccionAprobada` junto a la marcación original |
| App móvil | ✅ solicitar corrección (`correccion-nueva.tsx`, botón desde `asistencia-detalle.tsx`); el listado `correcciones.tsx` existe en el código pero el plan móvil aún lo marca pendiente |
| Despliegue | ✅ el backend de esta fase corre en `api.yefrasoft.com` desde el 13-nov-2026 |

## Alcance

**Cubre:** detección automática y manual de incidencias, su resolución o descarte por RH, solicitud de corrección por el dueño de la marcación y su revisión por RH, con auditoría de cada decisión.

**No cubre:** corregir una marca inexistente (`correcciones_asistencia.asistencia_id` es obligatorio; ese caso lo atiende RH sobre la incidencia), edición directa de marcaciones por RH, notificación al colaborador al resolver una corrección (la especificación §6 no la define), días laborales por horario ni marcación sin conexión. Las marcaciones viven en [[rh-cotla-fase-04-checador-sub-plan]]; el reporte de incidencias, en [[rh-cotla-fase-08-reportes-sub-plan]].

## Decisiones

1. **Solo jornadas cerradas.** El detector revisa `fecha_local < hoy` de los últimos `Incidencias:DiasRevision` = 7 días. `entrada_faltante` y `salida_faltante` solo si la jornada tiene al menos una marcación; un día sin marcaciones no genera incidencia.
2. **Tipos y estados cerrados.** Tipos `entrada_faltante`, `salida_faltante`, `fuera_de_zona` (una por marcación); estados `pendiente`, `resuelta`, `descartada`. La comida no genera incidencia (§6.7).
3. **Idempotencia.** No se repite una incidencia aunque ya esté resuelta: índices únicos parciales `UX_incidencias_dia_tipo` y `UX_incidencias_asistencia_tipo`, y `pg_advisory_xact_lock` para serializar corridas.
4. **Ejecución.** `IncidenciasHostedService` cada `IntervaloMinutos` = 30 y bajo demanda con `POST /incidents/detect` (responde `{ "creadas": n }`); configurable en `Incidencias:{Habilitado=true, IntervaloMinutos=30, DiasRevision=7}`.
5. **Atender es solo de RH.** Resolver o descartar con `descripcion` opcional; `409` si ya fue atendida. El gerente solo lee su equipo.
6. **Una corrección pendiente por marcación** (`409`), con candado por marcación. El dueño solicita con motivo y hora propuesta (ISO 8601 con zona, mismo día, no futura) o tipo propuesto; al menos uno.
7. **Aprobar no modifica `asistencias`.** `GET /attendance/{id}` agrega `correccionAprobada { correccionId, fechaHoraPropuesta, tipoPropuesto, revisadoAt }`. Decisión de §31 aplicada y por confirmar con el cliente.
8. **Rechazar exige `motivoRechazo`.**
9. **Auditoría:** `solicitar_`, `aprobar_` y `rechazar_correccion_asistencia`, `resolver_incidencia`, `descartar_incidencia`.

## Implementación

### Backend (`app-backend/cotla-api`)
- [x] `Modules/Incidents/Services/IncidenciaDetector` e `IncidenciasHostedService`.
- [x] Migración `Fase5IncidenciasIdempotentes` y `infra/db.sql` sincronizado.
- [x] Endpoints (todos requieren `Bearer`; listados paginados sin filtros vacíos):
  - `GET /api/v1/incidents` y `GET /api/v1/incidents/{id}` — `rh` todo, `gerente` sus equipos, `colaborador` lo suyo. Filtros `fechaDesde`, `fechaHasta`, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estado`.
  - `POST /api/v1/incidents/{id}/resolve` y `/dismiss` — solo `rh`.
  - `POST /api/v1/incidents/detect` — solo `rh`.
  - `POST /api/v1/attendance-corrections` — el dueño de la marcación (`403` si es ajena).
  - `GET /api/v1/attendance-corrections` y `/{id}` — `rh` todo, `gerente` sus equipos, `colaborador` lo suyo (filtros `empleadoId`, `asistenciaId`, `estado`).
  - `POST /api/v1/attendance-corrections/{id}/approve` y `/reject` — solo `rh`.

### Panel web (`app-frontend`)
- [x] `/incidencias`: listado con filtros, resolver o descartar con nota opcional y «Detectar ahora» (RH); el gerente solo lee.
- [x] `/correcciones`: listado con filtros (colaborador, estado); aprobar o rechazar con motivo obligatorio (RH).
- [x] `/asistencia`: el detalle muestra `correccionAprobada` junto a la marcación original.
- [x] Menú «Incidencias» y «Correcciones» para `rh` y `gerente`.

### App móvil (`app-movil`)
- [x] `src/lib/api-corrections.ts` (`createCorrection`, `listCorrections`, `getCorrection`).
- [x] `/correccion-nueva` (`correccion-nueva.tsx`): motivo obligatorio, hora y tipo propuestos opcionales; entrada desde `asistencia-detalle.tsx` cuando no hay corrección aprobada.
- [ ] `/correcciones` (`correcciones.tsx`, enlazada desde `(tabs)/explore.tsx`): existe en el código; el plan móvil no la marca como hecha hasta confirmarla en dispositivo.

## Criterios de Aceptación

- [x] Se detectan marcas obligatorias faltantes.
- [x] La corrección no sobreescribe el histórico original.
- [x] La auditoría registra quién, acción y fecha.
- [x] RH resuelve o descarta incidencias y el gerente solo consulta.
- [x] Solo existe una corrección pendiente por marcación.
- [ ] «Detectar ahora» trae también las incidencias fuera de zona (F-BE-4).

## Pendientes para otras fases

- [ ] Las incidencias y correcciones pendientes alimentan el reporte unificado: ver [[rh-cotla-fase-08-reportes-sub-plan]].
- [ ] Alertas de marcas faltantes y campana: ver [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]].
- [ ] Regla de días laborales y de ausentes (**FX-BE-1**, afecta el dashboard): ver [[rh-cotla-fase-09-dashboards-calendario-sub-plan]].
- [ ] Marcación sin conexión (D7): ver [[rh-cotla-fase-04-checador-sub-plan]].

## Relacionado

- [[rh-cotla-plan]] — plan general del proyecto, stack y orden de las fases.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos implementados del backend; §21 documenta esta fase.
- [[rh-cotla-integracion-clientes-manual]] — contrato de API para web y móvil, sección «Incidencias, correcciones y ausencias».
- [[rh-cotla-pendientes-sub-plan]] — deuda y trabajo futuro, incluidos F-BE-4, F-BE-8 y FX-DOC-1.
- [[rh-cotla-fase-04-checador-sub-plan]] — marcaciones de las que nacen las incidencias y las correcciones.
- [[rh-cotla-fase-06-ausencias-vacaciones-sub-plan]] — fase siguiente: solicitudes de ausencia y saldos.
