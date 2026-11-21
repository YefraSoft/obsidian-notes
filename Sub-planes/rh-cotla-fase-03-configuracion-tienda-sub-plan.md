---
tipo: sub-plan
titulo: "Fase 3 — Configuración operativa y tienda"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: completado
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - fase-3
  - catalogos
  - tienda
---

# Fase 3 — Configuración operativa y tienda

> **Plan padre:** [[rh-cotla-plan]]
> Catálogos (departamentos, puestos, horarios, equipos y gerentes), tienda con zona geográfica y saldos iniciales de vacaciones: la configuración que el checador y las solicitudes necesitan. Backend y panel completos; el móvil no tiene pantallas propias en esta fase.

## Pendientes

- [x] Backend: catálogos, equipos con gerentes, tienda y saldos de vacaciones.
- [x] Panel web: configuración, ubicación con mapa y saldos de vacaciones.
- [ ] Política de devengo de vacaciones (D1 de la especificación original): diferida, por confirmar con el cliente (**F-BE-8**).
- [ ] Alinear la zona de la tienda entre `.env` (`20.638516, -103.302586`) y la base (`20.638487, -103.302512`, radio 10 m, tolerancia 10 min); la semilla solo aplica a tablas vacías.
- [ ] Acordar un radio mínimo de la zona: la base solo exige `> 0`.
- [ ] Documentar la carga de configuración inicial en producción (tienda, zona, horarios, saldos, usuarios) (**F-INF-3**).
- [ ] 2 `eslint-disable react-hooks/exhaustive-deps` en `mapa-geocerca.tsx` y `mapa-marcacion.tsx` (**FX-FE-8**).

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | ✅ Completa, sin migraciones nuevas (el modelo existe desde la Fase 1); contratos en [[rh-cotla-progreso-backend-sub-plan]] §11–§15 |
| Panel web | ✅ `/configuracion/*`, `/configuracion/ubicacion` con mapa Leaflet y `/vacaciones` |
| App móvil | — (sin pantallas de esta fase; consume `GET /store` y el saldo en otras fases) |
| Entregables clave | CRUD con baja lógica, gerentes m2m, tienda singular, saldo inicial y ajustes con signo y auditoría |

## Alcance

Cubre §4 de `docs/especificaciones.md` y los saldos iniciales (§8 de la especificación original). No cubre el motor de devengo, múltiples sucursales ni turnos rotativos (fuera de alcance), ni el consumo de saldo al aprobar ausencias ([[rh-cotla-fase-06-ausencias-vacaciones-sub-plan]]). El uso de la zona para etiquetar marcaciones vive en [[rh-cotla-fase-04-checador-sub-plan]].

## Decisiones

Cerradas el 25 de septiembre de 2026 (detalle en [[rh-cotla-progreso-backend-sub-plan]] §15):

1. **D1 — Devengo fuera de fase:** solo saldo inicial y ajuste manual.
2. **D2 — La tolerancia manda en el horario:** `horarios.tolerancia_entrada` es la fuente; la de la tienda es el valor por defecto que se copia al crear un horario y cambiarla no altera los existentes.
3. **D3 — Lectura abierta, escritura solo RH:** catálogos y tienda se leen con cualquier rol; los saldos se restringen por alcance.
4. **D4 — Ajuste con signo:** positivo suma a `diasGenerados`; negativo resta a `diasIniciales` sin dejarlos negativos; `diasUtilizados` no se toca desde aquí.
5. **D5 — `null` en `PATCH /employees` no desasigna.**
6. **Baja lógica:** `DELETE` es idempotente y el nombre dado de baja sigue reservado (409 al repetirlo). Nombre de 3 a 100 caracteres, único por sucursal; descripción máx. 250.
7. **404 para cualquier referencia que no resuelve en la sucursal** y 400 solo para «existe pero está dado de baja».
8. **Horarios:** entrada < salida; comida completa o ninguna; tolerancia de 0 a 240 min.
9. **Tienda:** latitud −90..90, longitud −180..180, radio > 0, tolerancia ≥ 0, RFC de 12 o 13 caracteres (se guarda en mayúsculas); coordenadas a 6 decimales.
10. **Saldo:** `diasIniciales` de 0 a 365; ajustes de −365 a 365 y distintos de 0; `diasDisponibles` es columna calculada, nunca se escribe.
11. **El listado de saldos es `rh-or-gerente`;** el colaborador consulta el suyo con el detalle por `empleadoId`.

## Implementación

### Backend

- [x] Departamentos, puestos, horarios y equipos con el mismo esqueleto (`GET` paginado con `activo` y `busqueda`, `POST`, `PATCH`, `DELETE`).
- [x] Gerentes y miembros de equipo; el alcance cambia de inmediato, sin nuevo login.
- [x] Asignación por `PATCH /employees/{id}` con `ReferenciaCatalogoService` (404/400, nunca 500).
- [x] Tienda: `GET` y `PATCH /store`; datos públicos en `GET /public/company`.
- [x] Saldos: lectura, `PUT` del saldo inicial (movimiento por la diferencia) y ajuste manual en una transacción.
- [x] Auditoría de toda escritura (`actualizar_tienda`, `definir_saldo_vacaciones`, `ajustar_saldo_vacaciones`, etc.).

### Endpoints

| Ruta | Rol |
|------|-----|
| `GET /api/v1/{departments\|positions\|schedules\|teams}` y `/{id}` | autenticado |
| `POST/PATCH/DELETE /api/v1/{departments\|positions\|schedules\|teams}` | `rh` |
| `GET /api/v1/teams/{id}/managers`, `/members` | autenticado (miembros con alcance) |
| `POST /teams/{id}/managers`, `DELETE /teams/{id}/managers/{empleadoId}` | `rh` |
| `GET /api/v1/store` | autenticado |
| `PATCH /api/v1/store` | `rh` |
| `GET /api/v1/public/company` | público |
| `GET /api/v1/vacation-balances` | `rh` y `gerente` |
| `GET /vacation-balances/{empleadoId}` y `/movements` | autenticado, con alcance |
| `PUT /vacation-balances/{empleadoId}`, `POST /vacation-balances/{empleadoId}/adjustments` | `rh` |

### Panel web

- [x] Departamentos, puestos, horarios y equipos — `/configuracion/*` (el redirect de `/configuracion` está en `next.config.ts`).
- [x] Equipos con gerentes y miembros — `/configuracion/equipos`.
- [x] Tienda y zona con mapa Leaflet — `/configuracion/ubicacion`.
- [x] Saldos de vacaciones — `/vacaciones` (filtros, detalle con movimientos, saldo inicial y ajustes con signo; lectura para RH y gerente).

### App móvil

- — Sin pantallas en esta fase. La lectura de `GET /store` y del saldo propio se trata en [[rh-cotla-fase-06-ausencias-vacaciones-sub-plan]].

## Criterios de Aceptación

- [x] Solo RH escribe configuración; gerente y colaborador reciben 403.
- [x] Horarios y zona persisten y son consultables.
- [x] El saldo inicial queda disponible para validar solicitudes.
- [x] Un catálogo dado de baja no se quita de quien lo tiene, solo deja de aceptarse en asignaciones nuevas.
- [x] Asignar un gerente amplía su alcance y retirarlo lo reduce (`TeamsEndpointsTests.Asignar_un_gerente_amplia_su_alcance_y_retirarlo_lo_reduce`).
- [ ] Zona de la tienda alineada entre `.env` y base.

## Pendientes para otras fases

- [ ] Motor de devengo de vacaciones (D1): [[rh-cotla-fase-06-ausencias-vacaciones-sub-plan]].
- [ ] Mejoras de vista móvil del panel y tablas (**F-FE-4**, **F-FE-5**): [[rh-cotla-pendientes-sub-plan]].
- [ ] Carga de configuración inicial en producción (**F-INF-3**): [[rh-cotla-fase-12-despliegue-sub-plan]].

## Relacionado

- [[rh-cotla-plan]] — plan padre.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos y decisiones D1–D5 de esta fase (§11–§15).
- [[rh-cotla-integracion-clientes-manual]] — contrato de catálogos, tienda, `public/company` y saldos para web y móvil.
- [[rh-cotla-pendientes-sub-plan]] — deuda con ids F-/FX-.
- [[rh-cotla-fase-02-autenticacion-colaboradores-sub-plan]] — fase anterior: expediente que referencia estos catálogos.
- [[rh-cotla-fase-04-checador-sub-plan]] — siguiente fase: usa zona, radio y horario configurados aquí.
- [[rh-cotla-fase-06-ausencias-vacaciones-sub-plan]] — consume el saldo inicial al aprobar solicitudes.
- [[rh-cotla-fase-12-despliegue-sub-plan]] — carga de configuración inicial en producción.
- Repo: `docs/especificaciones.md` (§4), `app-backend/`, `app-frontend/app/(panel)/configuracion/`.
