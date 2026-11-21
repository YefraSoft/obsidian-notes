---
tipo: sub-plan
titulo: "Fase 7 — Horas extra"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: en-progreso
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - fase-7
  - horas-extra
---

# Fase 7 — Horas extra

> **Plan padre:** [[rh-cotla-plan]] — plan general de RH Cotla con el stack y el orden de las fases.
> Registro desde la app móvil con hasta 5 fotografías de evidencia, cálculo automático de horas en el servidor y aprobación por RH (`docs/especificaciones.md` §8).

## Pendientes

- [ ] **FX-MOV-3** `horas-extra-nueva.tsx` promete «seleccionar de la galería» pero solo existe la cámara y `expo-image-picker` no está en `package.json`; quitar el texto o implementar el selector (⚠ antes de las tiendas). Ver [[rh-cotla-pendientes-sub-plan]].
- [ ] Confirmar que `horas-extra-detalle.tsx` muestra las fotos: el plan móvil lo daba por pendiente y el código ya incluye la galería de evidencias (**FX-DOC-1**).
- [ ] **FX-FE-7** Accesibilidad en `horas-extra-detalle.tsx` del panel: un `alt` que es un nombre de archivo y `eslint-disable no-img-element`.
- [ ] Decisiones sin definir en la especificación: cancelar o editar tras enviar y horas que cruzan medianoche.

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | ✅ completo, sin migraciones (usa `horas_extra` y `horas_extra_archivos` de la Fase 1): `Modules/Overtime` (`HoraExtraService`). Pruebas: `OvertimeEndpointsTests` (9) |
| Panel web | ✅ `/horas-extra`: listado con filtros, detalle con evidencias, aprobar o rechazar (RH), consulta del gerente |
| App móvil | 🟡 `horas-extra-nueva.tsx` y `horas-extra.tsx` hechos; el detalle (`horas-extra-detalle.tsx`) existe en el código pero el plan móvil lo mantiene pendiente de confirmar |
| Despliegue | ✅ backend en `api.yefrasoft.com` desde el 13-nov-2026 |

## Alcance

**Cubre:** alta con evidencias, cálculo de horas, consulta con alcance por rol, descarga de fotos con sesión, aprobación o rechazo por RH.

**No cubre:** cancelar o editar tras enviar, horas que cruzan medianoche (ambos por vacío de la especificación), el correo y la bandeja al aprobar o rechazar (ver [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]]) y eventos en vivo hacia el panel (**F-BE-5**, v2).

## Decisiones

1. **Horas calculadas por el servidor.** `totalHoras` es la columna generada de PostgreSQL (`hora_final − hora_inicial`); un campo `totalHoras` enviado en el formulario se ignora (§9.2 de la especificación original).
2. **Campos.** `fecha` (`yyyy-MM-dd`, no futura), `horaInicio` y `horaFin` (`HH:mm`, mismo día, fin > inicio), `concepto` obligatorio, `comentarios` opcional (≤255).
3. **Fotos: 0 a 5**, JPEG o PNG verificados por firma, 5 MB c/u (`HorasExtra:{FotoMaxBytes, MaxFotos}`); se guardan referencias en `horas_extra_archivos`, no binarios, y se recomprimen a WebP (cola de la Fase 11). Si falla el guardado se borran las fotos ya subidas.
4. **Estados:** `pendiente` → `aprobada` | `rechazada`; `motivoRechazo` obligatorio (3–255); `409` si ya no está pendiente, con `pg_advisory_xact_lock`.
5. **Alcance:** `rh` todo, `gerente` sus equipos (solo lectura, §21.2), `colaborador` lo suyo. Cualquier usuario registra para sí mismo.
6. **Filtro `departamentoId`** añadido a `GET /overtime` de forma compatible hacia atrás.
7. **Auditoría:** `crear_`, `aprobar_` y `rechazar_hora_extra`.
8. **Las aprobadas** aparecen en el reporte de horas extra y en el calendario.

## Implementación

### Backend (`app-backend/cotla-api`)
- [x] Endpoints (todos requieren `Bearer`):
  - `POST /api/v1/overtime` — cualquier usuario; `multipart/form-data` (`fecha`, `horaInicio`, `horaFin`, `concepto`, `comentarios?`, `fotos`); `201` en `pendiente`; `400` por validación, `503` si falla el almacenamiento.
  - `GET /api/v1/overtime` — `rh` todo, `gerente` sus equipos, `colaborador` lo suyo; filtros `estado`, `fechaDesde`, `fechaHasta`, `empleadoId`, `departamentoId`, `equipoId`; paginado.
  - `GET /api/v1/overtime/{id}` — mismo alcance; incluye `motivoRechazo` y `fotos[]`.
  - `GET /api/v1/overtime/{id}/photos/{archivoId}` — mismo alcance; requiere `Bearer`.
  - `POST /api/v1/overtime/{id}/approve` y `/reject` — solo `rh`.

### Panel web (`app-frontend`)
- [x] `/horas-extra`: filtros (fechas, colaborador, departamento, equipo, estado); RH aprueba o rechaza con motivo; el gerente solo consulta.
- [x] Detalle con evidencias (descarga con `fetch` + `blob:`); `totalHoras` se muestra tal cual lo calcula el servidor.
- [x] Menú «Horas extra» para `rh` y `gerente`.

### App móvil (`app-movil`)
- [x] `/horas-extra-nueva` (`horas-extra-nueva.tsx`): fecha, horas, concepto y hasta 5 fotos con la cámara; el total lo calcula el backend.
- [x] `/horas-extra` (`horas-extra.tsx`): listado con estado (`GET /overtime`).
- [ ] `/horas-extra-detalle` (`horas-extra-detalle.tsx`): existe; falta confirmar que muestre las fotos en dispositivo.
- [ ] Selector de galería: no existe (FX-MOV-3).

## Criterios de Aceptación

- [x] Las horas se calculan automáticamente y no son editables.
- [x] Hasta 5 fotografías consultables por RH.
- [x] Aprobadas y rechazadas con motivo; las aprobadas aparecen en reportes.
- [x] El gerente solo consulta (sin aprobar).
- [ ] La app móvil permite elegir fotos de la galería (hoy solo cámara, FX-MOV-3).
- [ ] Suite completa sin fallas: al cerrar la fase quedaba una previa ajena (`AttendanceCorrectionsTests.Aprobar_conserva…`).

## Pendientes para otras fases

- [ ] Notificación y correo al aprobar o rechazar: ver [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]].
- [ ] Horas extra aprobadas en el calendario: ver [[rh-cotla-fase-09-dashboards-calendario-sub-plan]].
- [ ] Compresión a WebP y descarga protegida de fotos: ver [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]].
- [ ] Reporte de horas extra con total filtrado: ver [[rh-cotla-fase-08-reportes-sub-plan]].

## Relacionado

- [[rh-cotla-plan]] — plan general del proyecto, stack y orden de las fases.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos implementados del backend; §22 documenta esta fase.
- [[rh-cotla-integracion-clientes-manual]] — contrato de API para web y móvil, sección «Horas extra».
- [[rh-cotla-pendientes-sub-plan]] — deuda y trabajo futuro, incluidos FX-MOV-3, FX-FE-7, F-BE-5 y FX-DOC-1.
- [[rh-cotla-fase-06-ausencias-vacaciones-sub-plan]] — fase anterior: ausencias y vacaciones.
- [[rh-cotla-fase-08-reportes-sub-plan]] — fase siguiente: reportes, incluido el de horas extra.
