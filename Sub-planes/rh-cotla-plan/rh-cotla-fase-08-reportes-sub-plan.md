---
tipo: sub-plan
titulo: "Fase 8 — Reportes"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: completado
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - fase-8
  - reportes
---

# Fase 8 — Reportes

> **Plan padre:** [[rh-cotla-plan]] — plan general de RH Cotla con el stack y el orden de las fases.
> Cuatro reportes de consulta en pantalla, paginados y con el alcance del rol, sin exportación (`docs/especificaciones.md` §9).

## Pendientes

- [ ] Prueba del origen `correccion_pendiente` del reporte de incidencias: la consulta existe y compila, pero no tiene caso en `ReportsEndpointsTests`.
- [ ] Recorrido de `/reportes` con el backend real (**FX-FE-11**). Ver [[rh-cotla-pendientes-sub-plan]].
- [ ] Reglas que afectan los datos de origen: días laborales y ausentes (**FX-BE-1**) y detección de fuera de zona en «Detectar ahora» (**F-BE-4**).

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | ✅ completo, sin migraciones: `Modules/Reports` (`ReportService`), `/reports/attendance`, `/incidents`, `/overtime` y `/vacations`. Pruebas: `ReportsEndpointsTests` (7) |
| Panel web | ✅ `/reportes` con una pestaña por reporte y filtros alineados con el backend |
| App móvil | ➖ sin pantalla: los reportes son del panel (RH y gerente); el móvil no tiene fase 8 en su plan |
| Despliegue | ✅ backend en `api.yefrasoft.com` desde el 13-nov-2026 |

## Alcance

**Cubre:** consulta paginada con filtros de marcaciones, incidencias, horas extra y saldos con solicitudes, con control de alcance por rol (`rh` todo, `gerente` sus equipos, `colaborador` lo suyo).

**No cubre:** exportación a archivo (PDF, Excel o CSV), pantallas móviles de reportes, ni tableros y gráficas (ver [[rh-cotla-fase-09-dashboards-calendario-sub-plan]]). Los resultados pueden tener hasta 15 s de retraso por caché (Fase 11).

## Decisiones

1. **Solo consulta, sin exportación** (§11 original). Todos son `GET` autenticados; paginación estándar (`page`, `pageSize`), fechas `yyyy-MM-dd`.
2. **Alcance en el servicio**, no en el cliente: `rh` todo, `gerente` sus equipos, `colaborador` lo suyo. En marcaciones el gerente no ve selfie ni coordenadas (§3.2).
3. **Validación:** `400` si `fechaDesde > fechaHasta` o un `tipo` o `estado` no existe.
4. **Incidencias unificadas:** filas `{origen, referenciaId, empleado, fecha, tipo, estado, descripcion}` con origen incidencia (`entrada_faltante`, `salida_faltante`, `fuera_de_zona`), `solicitud_pendiente` y `correccion_pendiente`. Sin `estado`, solo pendientes; el `estado` aplica igual a los tres orígenes. Se une en memoria (20–50 colaboradores, una tienda).
5. **Horas extra:** `{ registros, totalHorasFiltradas }`; la suma cubre todo el filtro, no solo la página.
6. **Vacaciones:** por colaborador, saldo (iniciales, generados, utilizados, disponibles) y `solicitudes[]`; a diferencia de `/vacation-balances`, el colaborador sí consulta su propio reporte.
7. **Marcaciones** delegan en el listado de asistencia (`AsistenciaResponse`: coordenadas, zona, `fotoUrl`).
8. **`departamentoId`** se añadió a `GET /overtime` y `/reports/overtime` sin romper clientes.

## Implementación

### Backend (`app-backend/cotla-api`)
- [x] Endpoints (todos `GET`, requieren `Bearer`):
  - `GET /api/v1/reports/attendance` — filtros `fechaDesde`, `fechaHasta`, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estadoZona`.
  - `GET /api/v1/reports/incidents` — fechas, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estado`.
  - `GET /api/v1/reports/overtime` — fechas, `empleadoId`, `departamentoId`, `equipoId`, `estado`.
  - `GET /api/v1/reports/vacations` — fechas, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estado`.
- [x] Fotos: `fotoUrl` y `fotos[].url` se consumen con `Bearer`.

### Panel web (`app-frontend`)
- [x] `/reportes` con pestañas: marcaciones, incidencias, horas extra y saldos y solicitudes; solo pantalla.
- [x] El gerente no ve selfie ni coordenadas; incidencias sin `estado` muestran solo pendientes y la pantalla lo avisa; horas extra con el total de todo el filtro.
- [x] El `400` por rango de fechas inválido se muestra como error del listado.
- [x] Menú «Reportes» para `rh` y `gerente`.

### App móvil (`app-movil`)
- Sin cambios: no hay pantalla de reportes en el plan móvil.

## Criterios de Aceptación

- [x] Los reportes devuelven la información indicada con filtros funcionales.
- [x] Alcances por rol verificados en backend (colaborador, gerente y RH) en cada reporte.
- [x] Escalamiento para 20–50 colaboradores y 1 tienda.
- [x] Sin exportación, según la especificación.
- [x] Pantallas de reportes en el panel (`/reportes`).
- [ ] Cobertura de prueba del origen `correccion_pendiente`.

## Pendientes para otras fases

- [ ] Caché de reportes (hasta 15 s) y compresión de fotos: ver [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]].
- [ ] Exportación a archivo: fuera de 1.0.0; sin id en [[rh-cotla-pendientes-sub-plan]] (decidir con el cliente si se incluye).
- [ ] Indicadores agregados y gráficas: ver [[rh-cotla-fase-09-dashboards-calendario-sub-plan]].

## Relacionado

- [[rh-cotla-plan]] — plan general del proyecto, stack y orden de las fases.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos implementados del backend; §23 documenta los reportes.
- [[rh-cotla-integracion-clientes-manual]] — contrato de API para web y móvil, sección «Reportes».
- [[rh-cotla-pendientes-sub-plan]] — deuda y trabajo futuro, incluidos F-BE-4, FX-BE-1 y FX-FE-11.
- [[rh-cotla-fase-07-horas-extra-sub-plan]] — fase anterior: origen del reporte de horas extra.
- [[rh-cotla-fase-09-dashboards-calendario-sub-plan]] — fase siguiente: dashboards y calendario.
