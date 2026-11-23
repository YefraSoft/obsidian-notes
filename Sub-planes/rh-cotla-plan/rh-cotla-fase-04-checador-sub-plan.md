---
tipo: sub-plan
titulo: "Fase 4 — Checador de asistencia"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: en-progreso
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - fase-4
  - checador
  - asistencia
---

# Fase 4 — Checador de asistencia

> **Plan padre:** [[rh-cotla-plan]]
> Marcaciones con hora de servidor, selfie, GPS, zona, secuencia y ventana de horario. Backend y panel completos; la app móvil marca en línea, pero falta el error específico de GPS y la captura sin conexión.

## Pendientes

- [x] Backend: marcaciones, consulta, idempotencia, zona, secuencia, D6 (ventana) y D7 (hora capturada).
- [x] Panel web: lista de asistencia con filtros, detalle con selfie y mapa, «Mi jornada de hoy».
- [x] Móvil: checador con cámara frontal y GPS, jornada del día, historial y detalle.
- [ ] Móvil: distinguir GPS apagado de baja precisión; `checador.tsx` ya usa `hasServicesEnabledAsync`, pero falta confirmarlo en dispositivo (**FX-DOC-1**).
- [ ] Móvil: marcación sin conexión; el backend la soporta y la app solo marca en línea. Falta la decisión del cliente (offline total o tolerancia de cortes) y la política de conflictos.
- [ ] Confirmar con el cliente los límites provisionales de D6 y D7: ventana de 30 min antes / 60 min después, antigüedad máxima de 24 h y margen de reloj de 120 s (**F-BE-8**).
- [ ] Política de conflictos: GPS desactualizado y fotos pendientes de subir.
- [ ] Error de conectividad genérico y sin reintento ni detección de red en la app (**FX-MOV-7**).
- [ ] Enlace al aviso de privacidad en el checador, que captura selfie y GPS (**FX-MOV-5**).
- [ ] Verificar en el servidor la compresión real de fotos contra MinIO (**FX-BE-7**).
- [ ] Precisión mínima del GPS y semántica del retardo, definidas por el cliente.

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | ✅ Completo; D6 y D7 implementadas el 23-oct-2026 con límites provisionales; suite en verde (258 de 258 en esa fecha) y pila real verificada |
| Panel web | ✅ `/asistencia`, detalle con selfie y mapa, tarjeta «Mi jornada de hoy» |
| App móvil | 🟡 Checador, jornada, historial y detalle listos; faltan errores de GPS y el offline |
| Entregables clave | Idempotencia por `eventoDispositivoId`, Haversine, ventana horaria con auditoría, selfie en multipart, `IFileStorage` sobre MinIO |

## Alcance

Cubre §5 de `docs/especificaciones.md`: las cuatro marcaciones, la selfie, el GPS, la zona, la secuencia, la ventana de horario y la consulta. No cubre incidencias ni correcciones de marcaciones ([[rh-cotla-fase-05-incidencias-correcciones-sub-plan]]), ni la consulta por reportes ([[rh-cotla-fase-08-reportes-sub-plan]]). La edición directa de marcaciones por RH no existe: solo hay correcciones aprobadas.

## Decisiones

Cerradas el 2 de octubre de 2026 y revisadas el 23 de octubre de 2026 (detalle en [[rh-cotla-progreso-backend-sub-plan]] §17–§18):

1. **Hora oficial del servidor,** día calendario en `America/Mexico_City`; el cliente nunca envía la hora oficial.
2. **Secuencia:** entrada → [salida a comer → regreso] → salida; cada tipo una vez al día; comida opcional pero abierta debe cerrarse antes de la salida; violación: 409.
3. **Zona:** Haversine contra el radio de la tienda; `dentro_de_zona` o `fuera_de_zona`. **Fuera de zona no bloquea**, solo etiqueta.
4. **D6 — Ventana de horario:** `[entrada − 30 min, salida + 60 min]`, una sola para las cuatro marcaciones; fuera de ella 409 con «Fuera de horario: puedes marcar de HH:mm a HH:mm.» y registro `marcacion_rechazada_fuera_de_horario` en auditoría. Sin horario no hay bloqueo. Provisional.
5. **D7 — Hora capturada:** `registradaEnDispositivoAt` (ISO 8601 con zona); si difiere del servidor menos de 120 s se usa la del servidor; teléfono adelantado: 400; más de 24 h: 409; anterior a otra marca del día: 409. Provisional.
6. **Idempotencia:** `eventoDispositivoId` obligatorio (máx. 100, UUID por intento); mismo id, colaborador y tipo devuelve 200 con la original; con otro colaborador o tipo, 409.
7. **D8 — Selfie en `multipart/form-data`** dentro de la marcación: JPEG o PNG verificado por firma, máx. 5 MB (`Asistencia:FotoMaxBytes`).
8. **D9 — SDK de MinIO** detrás de `IFileStorage` (`Common/Storage`); ruta `sucursales/{id}/asistencias/{yyyy}/{MM}/{guid}.{ext}`; fotos servidas por el API con alcance, no por URL directa.
9. **Retardo** solo informativo en la entrada: `max(0, hora local − (horaEntrada + toleranciaEntrada))`; se calcula al responder con el horario vigente y no genera incidencia.
10. **Límites configurables** por variables `ASISTENCIA_*` (`BLOQUEAR_FUERA_DE_HORARIO`, `MINUTOS_ANTES_DE_ENTRADA`, `MINUTOS_DESPUES_DE_SALIDA`, `TOLERANCIA_RELOJ_SEGUNDOS`, `MAX_HORAS_SIN_CONEXION`).
11. **Cámara frontal sin galería** en la app; GPS con aviso si la precisión supera 50 m, pero deja marcar. Las fotos se recomprimen a WebP (máx. 1600 px, calidad 80, sin EXIF).

## Implementación

### Backend

- [x] `AttendanceService.RegistrarAsync`: validación, idempotencia, estado laboral, secuencia, zona, subida y transacción con `pg_advisory_xact_lock(sucursal, empleado)`.
- [x] Consulta: `today` con `siguientesPermitidas`, `faltantesObligatorias`, `horario` y `ventanaMarcacion`; listado con filtros; detalle con la zona de la tienda; foto.
- [x] Contrato aditivo: `origen`, `sincronizadaAt` y `desfaseSegundos` en `AsistenciaResponse`.
- [x] Pruebas: `AttendanceEndpointsTests`, `ReglaHorarioTests`, `ReglaSincronizacionTests`, `AttendanceHorarioBloqueoTests`, `AttendanceSincronizacionTests`.
- [x] Recorrido en la pila real (entrada, comida, salida, reintento, fuera de zona, alcance).

### Endpoints

| Ruta | Rol |
|------|-----|
| `POST /api/v1/attendance/check-in`, `lunch-out`, `lunch-in`, `check-out` | autenticado (cualquier rol; `multipart/form-data`) |
| `GET /api/v1/attendance/today` | autenticado (propio) |
| `GET /api/v1/attendance` | autenticado: `rh` todo, `gerente` su equipo, `colaborador` lo suyo |
| `GET /api/v1/attendance/{id}`, `GET /attendance/{id}/photo` | autenticado, con alcance |

Filtros de `GET /attendance`: `fechaDesde`, `fechaHasta`, `empleadoId`, `departamentoId`, `equipoId`, `tipo`, `estadoZona` (no enviar vacíos; 400). Respuestas de marcación: 201 creada, 200 reintento, 400, 403 (no activo), 409, 503 (almacenamiento caído, nada se guarda).

### Panel web

- [x] Lista de asistencia — `/asistencia` con filtros alineados al API y columnas de Departamento y Equipo.
- [x] Detalle con selfie (descarga con `fetch` como `blob:`) y mapa de coordenadas.
- [x] «Mi jornada de hoy» en `/dashboard` con `attendance/today`.

### App móvil

- [x] Checador — `checador.tsx`: cámara frontal, permiso de ubicación, latitud, longitud y precisión, reintento idempotente con UUID reutilizado, errores de cámara y de red.
- [x] Jornada del día con `siguientesPermitidas`, `faltantesObligatorias` y horario.
- [x] Historial — `historial.tsx` (filtro por tipo, «Cargar más»).
- [x] Detalle — `asistencia-detalle.tsx` (foto, zona, retardo, coordenadas, radio, `correccionAprobada`).
- [ ] Error de GPS apagado frente a baja precisión (por confirmar en dispositivo).
- [ ] Sincronización sin conexión: almacenamiento local y envío de `registradaEnDispositivoAt`.

## Criterios de Aceptación

- [x] La entrada guarda hora del servidor, GPS y selfie.
- [x] Se detecta `fuera_de_zona` y se guarda sin bloquear.
- [x] Comida opcional; secuencia inválida y duplicados rechazados con 409.
- [x] Reintento con el mismo UUID devuelve 200 sin duplicar.
- [x] Fuera de la ventana responde 409 y queda auditado.
- [x] RH consulta el detalle por colaborador y día; gerente su equipo; colaborador lo suyo (403 en lo ajeno).
- [ ] Marcación sin conexión desde la app móvil.
- [ ] Límites de D6 y D7 confirmados por el cliente.

## Pendientes para otras fases

- [ ] «Detectar ahora» debe traer también las incidencias fuera de zona (**F-BE-4**): [[rh-cotla-fase-05-incidencias-correcciones-sub-plan]].
- [ ] Errores de red y reintento general en la app (**FX-MOV-7**): [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]].
- [ ] Recompresión de fotos y caché verificadas en el servidor (**FX-BE-7**): [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]].
- [ ] Tienda y zona del servidor alineadas con `.env`: [[rh-cotla-fase-03-configuracion-tienda-sub-plan]].

## Relacionado

- [[rh-cotla-plan]] — plan padre.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos del checador, orden de reglas, decisiones D6–D9 y verificaciones en la pila real (§17–§18).
- [[rh-cotla-integracion-clientes-manual]] — contrato de marcación, reintentos, offline y consulta para web y móvil.
- [[rh-cotla-pendientes-sub-plan]] — deuda con ids F-/FX-.
- [[rh-cotla-fase-03-configuracion-tienda-sub-plan]] — fase anterior: zona, radio, horarios y tolerancia que usa el checador.
- [[rh-cotla-fase-05-incidencias-correcciones-sub-plan]] — fase siguiente: incidencias y correcciones sobre estas marcaciones.
- [[rh-cotla-fase-08-reportes-sub-plan]] — reportes de asistencia.
- [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]] — alertas y notificaciones.
- [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]] — archivos (fotos) y auditoría.
- Repo: `docs/especificaciones.md` (§5), `app-backend/`, `app-frontend/app/(panel)/asistencia/`, `app-movil/src/app/checador.tsx`.
