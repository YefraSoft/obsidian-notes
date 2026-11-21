---
tipo: sub-plan
titulo: "Panel web RH — Plan por fases"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: completado
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - frontend
  - nextjs
  - fases
---
# Panel web RH — Plan por fases

> **Plan padre:** [[rh-cotla-plan]]
> **Nota (2026-11-20):** este plan por app queda como respaldo. El estado vigente vive en los sub-planes por fase (tabla «Estado Actual» de [[rh-cotla-plan]]) y la deuda en [[rh-cotla-pendientes-sub-plan]].

> Alcance: `app-frontend/` (Next.js 16 / React 19, bun).
> Fuente funcional: `docs/especificaciones.md` (v1.4), sobre todo §21 (pantallas del panel web), §5 (expediente), §6.5 (zona), §12 (dashboards), §13 (calendario), §14 (mapas), §15 (fotografías) y §17 (auditoría).
> Dependencias de API: [[rh-cotla-progreso-backend-sub-plan]] y [[rh-cotla-integracion-clientes-manual]]. Cada ítem indica si el backend ya lo cubre.
> Plan del backend: [[rh-cotla-backend-sub-plan]]. Plan de la app móvil: [[rh-cotla-movil-sub-plan]].

Leyenda de estado: ✅ hecho · 🟡 parcial · ⏳ pendiente · 🔓 pendiente y desbloqueado (el backend ya tiene el contrato) · 🔒 pendiente y bloqueado por backend.

---

## Índice

| Fase | Estado | Pantallas / qué ataca (especificación) | Backend | Detalle |
|------|:------:|----------------------------------------|---------|---------|
| 2 — Autenticación, cuentas y colaboradores | ✅ | §21.1 login, expediente, activar/desactivar; §5 expediente; usuarios y roles (`progreso-backend.md` §19) | ✅ | [→](#fase-2--autenticación-cuentas-y-colaboradores) |
| 3 — Configuración operativa y tienda | ✅ | §21.1 horarios, tienda y zona con mapa (§6.5, §14); saldos de vacaciones (§8) | ✅ | [→](#fase-3--configuración-operativa-y-tienda) |
| 4 — Checador de asistencia | ✅ | §21.1 consulta de asistencia con selfie, coordenadas y mapa (§6, §14, §15) | ✅ | [→](#fase-4--checador-de-asistencia) |
| 5 — Incidencias y correcciones | ✅ | §21.1 revisar correcciones (§6.8); incidencias | ✅ | [→](#fase-5--incidencias-y-correcciones) |
| 6 — Ausencias y vacaciones | ✅ | §21.1 solicitudes, aprobar/rechazar, saldos (§7, §8) | ✅ | [→](#fase-6--ausencias-y-vacaciones) |
| 7 — Horas extras | ✅ | §21.1 aprobar horas extra; §21.2 consulta del gerente (§9) | ✅ | [→](#fase-7--horas-extras) |
| 8 — Reportes | ✅ | Vistas con filtros, sin exportación (§11) | ✅ | [→](#fase-8--reportes) |
| 9 — Dashboards y calendario | ✅ | Dashboard RH y gerente (§12); consulta de calendarios (§13) | ✅ | [→](#fase-9--dashboards-y-calendario) |
| 10 — Alertas y notificaciones | ✅ | Alertas del equipo según permisos (§10); campana y eventos en vivo | ✅ ¹ | [→](#fase-10--alertas-y-notificaciones) |
| 11 — Archivos, auditoría y seguridad | ✅ | Pantalla de auditoría (§17); fotos con ETag y WebP (§15) | ✅ ¹ | [→](#fase-11--archivos-auditoría-y-seguridad) |
| 12 — Despliegue | 🟡 | Build de Next, dominio y hosting del panel (§29) | 🔒 Fase 12 | [→](#fase-12--despliegue) |

> ¹ Implementado contra el contrato de [[rh-cotla-integracion-clientes-manual]]. En la verificación del 12-nov-2026 `https://api.yefrasoft.com` respondía `404` en `/notifications`, `/audit` y `/panel/events`: la API desplegada aún no incluye las fases 10 y 11. Hasta que se redespliegue, esas pantallas muestran el error del servidor.

**Trabajo desbloqueado hoy** (el backend ya responde): todo construido. Lo que queda depende del despliegue del backend (fases 10 y 11 en producción) y del dominio/TLS del panel.

---

## Pendientes globales del panel

- [x] Script `typecheck` (`tsc --noEmit`) en `package.json`.
- [x] `README.md` real: bun, `NEXT_PUBLIC_API_URL`, `PASSWORD_RESET_BASE_URL`, comandos y rutas.
- [ ] Recorrido en DevTools de todas las pantallas con el backend real.

### Endurecimiento aplicado antes de 1.0.0 (20 de noviembre de 2026)

- [x] Bun **1.4.4** como versión única (`Dockerfile`, `package.json`, README). La imagen del panel se construyó con `--frozen-lockfile` y arrancó: `/` y `/privacidad` responden 200.
- [x] `NEXT_PUBLIC_API_URL` obligatoria: se eliminó el fallback a una IP de red local (`lib/api/client.ts`) y el `Dockerfile` falla si el build-arg falta o llega vacío.
- [x] Sesión no persistente por defecto (la casilla «mantener sesión» del login arranca en `false`): el token va a `sessionStorage` salvo que el usuario lo pida.
- [x] Cabeceras de seguridad en `next.config.ts`: `X-Content-Type-Options`, `X-Frame-Options: DENY`, `Referrer-Policy`, `Permissions-Policy`.
- [x] Enlace a `/privacidad` en el pie del login y de todas las pantallas del panel (`components/ui/legal-notice.tsx`), y la ruta documentada en el README.

### Futuras correcciones (auditoría del 20 de noviembre de 2026)

Registradas como corrección futura; ninguna está resuelta.

**Importantes**
- [ ] **CSP:** falta `Content-Security-Policy`. Next inyecta scripts en línea y Leaflet carga teselas de OpenStreetMap: requiere diseño con nonces y pruebas dedicadas.
- [ ] **Token en `localStorage`** cuando el usuario marca «mantener sesión» (`lib/auth.ts`). A futuro, cookie `httpOnly` (exige cambio en el backend).
- [ ] **Puerto 3000 publicado en todas las interfaces** (`docker-compose.yml`, `"3000:3000"`) aunque el túnel usa `127.0.0.1`. No se cerró porque el APK se sirve en `/descargas` y puede descargarse desde la red local; decidir si se limita a loopback.
- [ ] **Sin pruebas automatizadas:** solo `typecheck` y lint. Al menos smoke tests de login, guard de sesión y clientes de la API.
- [ ] **Despliegue pendiente:** la URL pública `/privacidad` y `/api/v1/public/company` devuelven 404 hasta redesplegar panel y API (no es deuda de código).

**Menores**
- [ ] Protección de rutas solo en el cliente (`components/session-guard.tsx`); el API es el guard real. Documentarlo o añadir `proxy`.
- [ ] ~20 etiquetas en español fuera de los `*.text.ts` (`lib/navegacion.ts`).
- [ ] Accesibilidad: 7 `eslint-disable no-img-element` y un `alt` que es un nombre de archivo (`horas-extra-detalle.tsx`).
- [ ] 2 `eslint-disable react-hooks/exhaustive-deps` en `components/ui/mapa-geocerca.tsx` y `mapa-marcacion.tsx`.
- [ ] La imagen runtime copia todo `node_modules` (con devDependencies) y no tiene `HEALTHCHECK`.
- [ ] Verificar el campo `ignoreScripts` de `package.json`, `@types/node ^20` y las dependencias desactualizadas (`bun outdated`).
- [ ] `Caddyfile` obsoleto (Caddy no está en el compose): borrarlo o documentarlo.

**Documentación de este plan por corregir**
- [ ] Fase 10, entradas de menú: solo existe «Ayuda» deshabilitada en `lib/navegacion.ts`; Nómina y Documentos ya no existen.
- [ ] Fase 12: el servicio `frontend` ya no usa `--profile web`; entra en el `docker compose up` normal.
- [ ] Nota ¹ del índice: usa rutas sin `/api/v1`; hoy `/api/v1/notifications`, `/audit` y `/panel/events` existen en la API desplegada (responden 401 sin sesión).

---

## Fase 2 — Autenticación, cuentas y colaboradores

> **Estado: ✅.** Login, recuperación de contraseña, plantilla, cuenta y usuarios/roles completos.

### 2.6 Pantallas del panel (RH)

- [x] Login (`/`)
- [x] Recuperación de contraseña (`/olvido-contrasena`, `/reset-password`)
- [x] Colaboradores: alta, edición y expediente (`/plantilla`) — §5
- [x] Activar, rechazar y desactivar colaboradores
- [x] Mi cuenta y sesiones (`/cuenta`): cambio de contraseña, `auth/me`, `auth/sessions`, `auth/logout`
- [x] **Usuarios / roles** (`/usuarios`): cliente `lib/api/users.ts` y pantalla con listado (filtros estado/rol/búsqueda), detalle con sesiones activas, cambio de rol, activar/rechazar/desactivar, revocar sesiones y reset de contraseña (`progreso-backend.md` §19). Solo RH.
- [x] Los 409 de «último RH» y autodegradación se muestran como error de la acción (el backend ya aplica las guardas).
- [x] `revokeAllSessions` (`lib/api/auth.ts`): se conserva para una futura pantalla de sesiones/auditoría; la revocación desde el panel usa `/users/{id}/sessions/revoke`.
- [x] El rol se administra solo en `/usuarios`; se retiró el selector de la ficha del colaborador (el alta nace con rol `colaborador` por defecto del backend).

---

## Fase 3 — Configuración operativa y tienda

> **Estado: ✅.** Catálogos, horarios, tienda y saldos de vacaciones completos.

### 3.6 Pantallas del panel (RH)

- [x] Configuración de departamentos, puestos, horarios y equipos (`/configuracion/*`)
- [x] Equipos con gerentes y miembros (`/configuracion/equipos`)
- [x] Configuración de tienda y zona con mapa Leaflet (`/configuracion/ubicacion`) — §6.5, §14
  - Los mapas de Leaflet se aíslan (`isolate`) para que sus z-index internos no se pinten por encima del overlay de los diálogos; el redirect de `/configuracion` vive en `next.config.ts`.
- [x] **Saldos de vacaciones** (`/vacaciones`): listado con filtros (búsqueda, departamento, equipo, con/sin saldo), detalle con movimientos paginados, y definición del saldo inicial / ajustes con signo (solo RH). Lectura para RH y gerente.

---

## Fase 4 — Checador de asistencia

> **Estado: ✅.** Consulta de asistencia para RH.

### 4.6 Asistencia en el panel (RH)

- [x] Lista de asistencia (`/asistencia`) — §6
- [x] Detalle con selfie y mapa de coordenadas (§14, §15)
- [x] Filtros por rango de fechas, colaborador, área, tipo y zona, alineados con `GET /attendance` (`fechaDesde`, `fechaHasta`, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estadoZona`)
- [x] Columnas de **Departamento** y **Equipo** en el listado (se resuelven por `empleadoId` desde el catálogo, porque la marcación no los trae)
- [x] `attendance/today` (`obtenerJornadaHoy`) usado en el dashboard como «Mi jornada de hoy» (jornada del usuario autenticado)

---

## Fase 5 — Incidencias y correcciones

> **Estado: ✅.** Consumen `/incidents` y `/attendance-corrections` ([[rh-cotla-integracion-clientes-manual]]).

### 5.3 Panel RH

- [x] Incidencias (`/incidencias`): listado con filtros (fechas, colaborador, departamento, equipo, tipo, estado); RH resuelve o descarta con nota opcional y ejecuta «Detectar ahora»; el gerente solo lee
- [x] Correcciones (`/correcciones`): listado con filtros (colaborador, estado); RH aprueba o rechaza con motivo obligatorio (§6.8)
- [x] Detalle de asistencia muestra `correccionAprobada` junto a la marcación original, que no cambia
- [x] Menú: Incidencias y Correcciones para `rh` y `gerente`

---

## Fase 6 — Ausencias y vacaciones

> **Estado: ✅.** Consume `/leave-requests`.

### 6.6 Panel RH

- [x] Solicitudes (`/solicitudes`): lista con filtros por estado, tipo, fechas, colaborador y equipo (§7)
- [x] Detalle con documentos (descarga con `fetch` + `blob:`), aprobar y rechazar con motivo (solo RH, solo `pendiente`); los `409` se muestran como error de la acción
- [x] Saldos de vacaciones (ver Fase 3); los movimientos incluyen el tipo `uso` generado al aprobar vacaciones
- Pendiente de backend: devengo automático (D1) y notificaciones (Fase 10)

---

## Fase 7 — Horas extras

> **Estado: ✅.** Consume `/overtime`. El registro (`POST`) es de la app móvil.

### 7.3 Panel RH y gerente

- [x] Horas extra (`/horas-extra`): listado con filtros (fechas, colaborador, departamento, equipo, estado); RH aprueba o rechaza con motivo (§9.3); el gerente solo consulta (§21.2)
- [x] Detalle con evidencias fotográficas (descarga con `fetch` + `blob:`); `totalHoras` lo calcula el servidor y se muestra tal cual
- [x] Menú: «Horas extra» para `rh` y `gerente`

---

## Fase 8 — Reportes

> **Estado: ✅.** Consume `/reports/*`. Solo pantalla, sin exportación (§27).

### 8.6 Vistas de reportes

- [x] Reportes (`/reportes`) con una pestaña por reporte y filtros alineados con el backend: marcaciones (§11.1), incidencias (§11.2), horas extra (§11.3) y saldos y solicitudes (§11.4)
- [x] Marcaciones: el gerente no ve selfie ni coordenadas (§3.2); incidencias sin `estado` muestran solo pendientes (se avisa en pantalla); horas extra con el total de todo el filtro
- [x] El `400` por rango de fechas inválido se muestra como error del listado

---

## Fase 9 — Dashboards y calendario

> **Estado: ✅.** Consume `/dashboard` y `/calendar`.

### 9.4 Dashboards y calendarios

- [x] Dashboard con pantalla (`/dashboard`) — §12
- [x] Indicadores reales (`GET /dashboard`): activos, presentes, ausentes, de licencia, fuera de zona y pendientes de incidencias, solicitudes, correcciones y horas extra, con enlace a cada pantalla; se retiró `dashboard-demo.ts`
- [x] «Mi jornada de hoy» real en el dashboard (`GET /attendance/today`)
- [x] Cumpleaños y aniversarios desde el dashboard (solo RH; el gerente recibe `null` y no los ve, §12.2)
- [x] Dashboard de gerente con alcance de equipo (§12.2): lo aplica el backend
- [x] Colaborador: no se pide `/dashboard` (403); solo ve su jornada
- [x] Calendario (`/calendario`): vista mensual con ausencias aprobadas, asistencia, incidencias y horas extra aprobadas (§13); RH elige colaborador

---

## Fase 10 — Alertas y notificaciones

> **Estado: ✅.** Consume `/notifications` y `/panel/events` (SSE).

### 10.5 Panel RH y gerente

- [x] Campana del encabezado con contador de no leídas (`GET /notifications/unread-count`: al abrir, cada 60 s, con cada evento y al cambiar la lectura)
- [x] Notificaciones (`/notificaciones`): todas o sin leer, marcar leída o no leída, marcar todas, enlace a la pantalla según `datos`; RH puede «Generar alertas de hoy» (cumpleaños y aniversarios)
- [x] Eventos en vivo (`components/layout/panel-eventos-provider.tsx`, `lib/panel-eventos.ts`): una conexión SSE por pestaña con `fetch` y `Authorization` (el `EventSource` no sirve), solo `rh` y `gerente`, reconexión con espera creciente. Con cada evento recargan los listados (`useListaFiltrada`, asistencia), el dashboard y la campana
- [ ] Entradas de menú deshabilitadas hoy (`lib/navegacion.ts`: Nómina, Documentos, Ayuda): habilitar solo cuando exista su alcance funcional (Nómina queda fuera del MVP, §27)

---

## Fase 11 — Archivos, auditoría y seguridad

> **Estado: ✅.** Consume `/audit`; las fotos ya llegan con `ETag` y en WebP.

### 11.1 Descarga protegida

- [x] Las descargas siguen con `fetch` + `Authorization` + `blob:`; el navegador revalida con `ETag` (`304`) por sí solo
- [x] Las fotos pasan a WebP poco después de subirse: la descarga de la selfie usa la extensión según el `Content-Type` (`.webp`, `.png` o `.jpg`)

### 11.2 Auditoría

- [x] Pantalla de auditoría solo RH (`/auditoria`): filtros de fechas, acción, entidad e ID; detalle con `datosAnteriores` y `datosNuevos`. Contiene datos personales: gerente y colaborador no la ven (el backend responde `403`)

---

## Fase 12 — Despliegue

> **Estado: 🟡.**

- [x] `Dockerfile` y `.dockerignore` del panel (contrato del compose: `ARG NEXT_PUBLIC_API_URL` antes de `next build`, Node 22 y bun 1.3.x, `next start -p 3000 -H 0.0.0.0`). **Sin construir ni probar todavía**
- [x] Variables por ambiente (`NEXT_PUBLIC_API_URL`) documentadas en `README.md`; `.env*` queda fuera de la imagen
- [ ] Activar el servicio `frontend` del compose (`--profile web`): lo hace el equipo del backend
- [ ] Dominio, TLS y proxy inverso del panel
- [ ] Smoke test del panel contra la API desplegada

---

## Dependencias con otras apps y con el backend

- El panel no modifica `app-backend` ni `app-movil`. Si necesita un endpoint nuevo, se anota en `plan-backend.md` y se pide al usuario que coordine.
- Los orígenes de CORS del panel los declara el backend; si falta un origen, se pregunta al usuario, no se edita el backend.
