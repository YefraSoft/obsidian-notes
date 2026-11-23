---
tipo: sub-plan
titulo: "Fase 9 — Dashboards y calendario"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: en-progreso
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - dashboards
  - calendario
  - fase-9
---

# Fase 9 — Dashboards y calendario

> **Plan padre:** [[rh-cotla-plan]] — plan general del proyecto RH Cotla, con stack, roles y estado por fase.
> Indicadores operativos para RH y gerente (`docs/especificaciones.md` §12) y calendario personal con consulta de RH (§13). Backend, panel y móvil están construidos; queda abierta la regla de «ausentes» y el pulido móvil del calendario web.

## Pendientes

- [ ] Definir con el cliente la regla de ausentes del dashboard (FX-BE-1): hoy cuenta a RH y gerente (sin horario) y no considera día laboral ni hora de entrada.
- [ ] Calendario web: ancho fijo del nombre del mes y cabecera de tabla sin saltos al cambiar de mes (F-FE-2, F-FE-3).
- [ ] Mejorar la vista móvil del calendario del panel (F-FE-2, F-FE-5).
- [ ] Decidir si el gerente consulta el calendario de su equipo: §13 no lo autoriza y no se implementó.

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | Completo, sin migraciones: `GET /dashboard` (RH y gerente) y `GET /calendar`; 315/315 pruebas al cerrar la fase (`DashboardEndpointsTests` 15, `CalendarEndpointsTests` 7) y suite actual 401/401 |
| Panel web | Hecho: `/dashboard` con indicadores reales, «Mi jornada de hoy», cumpleaños y aniversarios (solo RH) y `/calendario` mensual; se retiró `dashboard-demo.ts` |
| App móvil | Hecho: `src/app/calendario.tsx` con rejilla mensual, indicadores por categoría, leyenda y eventos del día; enlace «Calendario» en la pestaña «Más» |
| Despliegue | `api.yefrasoft.com` ya corre esta fase (fases 1–9 verificadas por `curl` el 13-nov-2026) |

## Alcance

Cubre indicadores del día por rol, próximos cumpleaños y aniversarios, y el calendario con ausencias, asistencia, incidencias y horas extra aprobadas. No cubre alertas ni notificaciones (ver [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]]), exportación de reportes (ver [[rh-cotla-fase-08-reportes-sub-plan]]) ni calendario de equipo para el gerente.

## Decisiones

1. **Ausente** = activo sin entrada hoy y sin ausencia aprobada que cubra hoy; **de licencia** = activo sin entrada con ausencia aprobada hoy. Invariante: `activos = presentes + ausentes + deLicencia`. No se considera horario ni descanso (decisión del cliente en esta fase; revisión abierta en FX-BE-1).
2. **Alcance por rol:** RH ve toda la sucursal; el gerente, solo los equipos que gestiona (`AlcanceService`); el colaborador no pide `/dashboard` (403).
3. **Cumpleaños y aniversarios:** `proximosCumpleanos` y `proximosAniversarios` solo para RH; el gerente recibe `null` (§12.2). Ventana `dias` de 1 a 90, 30 por defecto; fuera de rango da 400. Un 29-feb se celebra el 28-feb en años no bisiestos; aniversarios desde 1 año.
4. **Calendario:** `desde` y `hasta` obligatorios, máximo 62 días (400 si faltan, se invierten o exceden). Sin `empleadoId`, el propio; ajeno solo RH (403 a gerente y colaborador, 404 si no existe).
5. **Eventos del calendario:** `ausencia` (aprobadas, con `fechaFin`), `asistencia` (uno por día), `incidencia` (cualquier estado) y `hora_extra` (solo aprobadas).
6. **Vacíos de la especificación:** la ventana de «fuera de zona» es el día de hoy; los pendientes (incidencias, solicitudes, correcciones, horas extra) se cuentan sin límite de fecha.
7. **Caché:** `GET /dashboard` entra en la política `tablero` (15 s por usuario), por lo que puede tener hasta 15 s de retraso (ver [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]]).

## Implementación

### Backend (`app-backend/cotla-api/`)
- Módulo `Modules/Dashboard`: `DashboardService`, `CalendarService`, `FechasProximas`.
- `GET /api/v1/dashboard?dias=` — `rh` y `gerente`; devuelve `colaboradoresActivos`, `presentesHoy`, `ausentes`, `deLicencia`, `marcacionesFueraDeZonaHoy`, `incidenciasPendientes`, `solicitudesPendientes`, `correccionesPendientes`, `horasExtraPendientes`, `fecha`, `dias`.
- `GET /api/v1/calendar?desde=&hasta=&empleadoId=` — cualquier usuario autenticado; el calendario ajeno solo RH.

### Panel web (`app-frontend/`)
- `/dashboard`: indicadores con enlace a cada pantalla, «Mi jornada de hoy» (`GET /attendance/today`), cumpleaños y aniversarios (solo RH).
- `/calendario`: vista mensual; RH elige colaborador.

### App móvil (`app-movil/`)
- `src/lib/api-calendar.ts` (`getCalendar`) y `src/app/calendario.tsx` (ruta `/calendario`, entrada desde «Más»).

## Criterios de Aceptación

- [x] Los indicadores del dashboard son consistentes con los datos reales (invariante probado).
- [x] El gerente no ve datos fuera de su equipo ni cumpleaños ni aniversarios.
- [x] El colaborador recibe 403 en `/dashboard` y consulta solo su calendario; RH consulta el de otros.
- [x] Panel y móvil muestran el calendario con las cuatro categorías de evento.
- [ ] La regla de ausentes está confirmada por el cliente (FX-BE-1).
- [ ] El calendario web no tiene saltos de cabecera ni de nombre de mes (F-FE-2, F-FE-3).

## Pendientes para otras fases

- [ ] FX-BE-1 y F-FE-2/3 viven en [[rh-cotla-pendientes-sub-plan]].
- [ ] Revisar la caché de 15 s del dashboard con tráfico real (Fase 11).

## Relacionado

- [[rh-cotla-plan]] — plan padre con el estado general y las decisiones de stack.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos implementados del backend; §24 describe dashboard y calendario.
- [[rh-cotla-integracion-clientes-manual]] — contrato de API para web y móvil; sección «Dashboard y calendario (FASE 9)».
- [[rh-cotla-pendientes-sub-plan]] — deuda y trabajo futuro (FX-BE-1, F-FE-2, F-FE-3, F-FE-5).
- [[rh-cotla-fase-08-reportes-sub-plan]] — fase anterior: reportes con filtros y alcance por rol.
- [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]] — fase siguiente: cumpleaños y aniversarios del día como alertas.
- [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]] — caché de 15 s que afecta al dashboard.
