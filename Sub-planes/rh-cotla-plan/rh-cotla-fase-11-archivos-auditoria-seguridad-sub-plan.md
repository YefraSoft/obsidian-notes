---
tipo: sub-plan
titulo: "Fase 11 — Archivos, auditoría, seguridad y rendimiento"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: en-progreso
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - archivos
  - auditoria
  - seguridad
  - rendimiento
  - fase-11
---

# Fase 11 — Archivos, auditoría, seguridad y rendimiento

> **Plan padre:** [[rh-cotla-plan]] — plan general del proyecto RH Cotla, con stack, roles y estado por fase.
> Descarga privada de archivos (§15), consulta de auditoría (§17), endurecimiento (§16) y rendimiento (Redis, caché con `ETag`, compresión a WebP). Backend completo; quedan abiertas la revocación real de sesiones y las mejoras de seguridad del panel.

## Pendientes

- [ ] Revocación real de sesiones: `jti` en `sesiones_usuario` y validación en `OnTokenValidated`; hoy un usuario dado de baja conserva el token hasta 60 min (FX-BE-2, prioridad alta, 1.0.1).
- [ ] `Content-Security-Policy` en el panel: requiere nonces por los scripts en línea de Next y las teselas de OpenStreetMap (FX-FE-1).
- [ ] Token en `localStorage` con «mantener sesión»; a futuro cookie `httpOnly` (cambio en backend) (FX-FE-2).
- [ ] Rate limit general y proteger o retirar `/diagnostics/*` (FX-BE-3).
- [ ] Backups automáticos de PostgreSQL con restauración probada (F-INF-1).
- [ ] Verificar en el servidor la compresión real de fotos contra MinIO y la caché a través del tunnel (FX-BE-7).
- [ ] Contacto de emergencia en el expediente: sin endpoint (F-BE-3).

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | Completo, sin migraciones: archivos privados, `GET /audit`, cabeceras, IP real, rate limit, Redis, caché, `ETag` y cola WebP. Suite 401/401 (`AuditEndpointsTests` 7, `SeguridadHttpTests` 8, `CacheTests` 10, `ETagTests` 13, `ImagenCompresorTests` 11, `ImagenesProcesoTests` 13) |
| Panel web | Hecho: `/auditoria` solo RH; descargas con `fetch` + `Authorization` + `blob:`, revalidación por `ETag`, extensión según `Content-Type`; cabeceras de seguridad en `next.config.ts`. Sin CSP |
| App móvil | El plan móvil no documenta trabajo propio de esta fase; consume fotos y descargas con `Bearer` (ver fases 4, 6 y 7) |
| Despliegue | Código listo; compresión real contra MinIO y caché por el tunnel sin verificar (FX-BE-7) |

## Alcance

Cubre archivos (almacenamiento S3, descarga privada), auditoría consultable por RH, seguridad HTTP y rendimiento. No cubre backups, CI/CD ni hosting (ver [[rh-cotla-fase-12-despliegue-sub-plan]]).

## Decisiones

1. **Descarga privada por la API con `Bearer`**, validando sesión y alcance en cada petición; sin URLs firmadas ni prefirmadas y sin exponer MinIO. Se guardan referencia y clave en base, no el binario.
2. **No se borra historial:** ningún proceso elimina archivos (salvo huérfanos de un guardado fallido). Excepción consciente: al comprimir una foto, el original se reemplaza por su WebP.
3. **Auditoría solo RH**, acotada a su sucursal; contiene datos personales (p. ej. CURP). No guarda contraseñas ni hashes (probado).
4. **Cabeceras** (`SecurityHeadersMiddleware`): `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Permissions-Policy` (cámara, micrófono, geolocalización y pago vacíos); fuera de Development, `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'` y HSTS con HTTPS.
5. **IP real:** `UseForwardedHeaders` primero; confía en `X-Forwarded-For/Proto` solo desde `ForwardedHeaders:TrustedNetworks` y toma la última entrada (`ForwardLimit = 1`). Lo usan el rate limit y `auditoria.ip`. Riesgo aceptado A2 (ver Fase 12).
6. **Rate limit:** 10 peticiones por minuto y por IP en el bucket `auth` (login, self-register, forgot-password, reset-password); el 429 no trae `Retry-After`.
7. **Redis:** `redis:7-alpine`, sin puertos publicados, `--requirepass ${REDIS_PASSWORD}`, AOF con volumen `redis-data`, `--maxmemory 256mb --maxmemory-policy volatile-lru`. `/health` lo reporta `Healthy` o `Degraded`, nunca `Unhealthy`; sin Redis el API responde igual.
8. **Caché de salida:** solo `GET` 200 con sesión válida. `catalogo` 10 min (`Cache:CatalogoSegundos`, por sucursal y rol, invalidada al escribir catálogos o `/employees`); `tablero` 15 s (`Cache:TableroSegundos`, por usuario) para `/dashboard` y `/reports/*`.
9. **`ETag`:** catálogos y fotos (`/attendance/{id}/photo`, `/overtime/{id}/photos/{archivoId}`) devuelven `ETag` y `Cache-Control: private, no-cache`; `If-None-Match` vigente da 304. Sesión y alcance se validan antes que el `ETag`.
10. **Compresión WebP:** solo fotos JPEG/PNG (`selfie_asistencia`, `foto_hora_extra`); lado mayor 1600 px, calidad 80, sin EXIF/GPS, orientación aplicada, máximo 40 MP. Cola Redis Streams `cotla:imagenes`, 2 trabajadores, reclamo a los 300 s, descarte tras 3 entregas; barrida al arrancar y cada 6 h. Los documentos de incapacidad no se tocan.
11. **Deuda aceptada A1** (CORS abierto, JWT por cabecera sin cookies) se mantiene; ver Fase 12.

## Implementación

### Backend (`app-backend/cotla-api/`)
- `Common/Storage` (`IFileStorage`, MinIO), `SecurityHeadersMiddleware`, `ForwardedHeadersSetup.cs`, `ResilientOutputCacheStore`, `ImagenesHostedService`, `ProcesadorImagenes`, `ImagenCompresor`.
- `GET /api/v1/audit` — solo `rh`; filtros `fechaDesde`, `fechaHasta`, `usuarioId`, `accion`, `entidad`, `entidadId`; paginado; 403 a gerente y colaborador.
- `GET /api/v1/attendance/{id}/photo` y `GET /api/v1/overtime/{id}/photos/{archivoId}` — autenticado con alcance; `ETag`/304.
- `GET /health` — incluye `postgres`, `almacenamiento` y `redis`.

### Panel web (`app-frontend/`)
- `/auditoria`: filtros de fechas, acción, entidad e ID; detalle con `datosAnteriores` y `datosNuevos`.
- Selfie descargable con extensión `.webp`, `.png` o `.jpg` según `Content-Type`.
- `next.config.ts`: cabeceras `nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy`, `Permissions-Policy`.

### App móvil (`app-movil/`)
- Sin pantallas nuevas en esta fase; no asumir `image/jpeg` al descargar fotos (contrato en el manual de integración).

## Criterios de Aceptación

- [x] RH descarga fotos y documentos con sesión y alcance (con `Bearer`).
- [x] La auditoría registra responsable, acción, fecha y cambio, sin contraseñas ni hashes.
- [x] Un endpoint protegido rechaza sesión inválida (401/403), también con caché y `ETag`.
- [x] Las cabeceras de seguridad salen en 200, 401 y 404, y la IP real se toma desde red de confianza sin dejarse falsear.
- [x] Sin Redis el API sigue respondiendo (sin caché y con fotos sin comprimir).
- [ ] Revocación real de sesiones (FX-BE-2).
- [ ] CSP en el panel (FX-FE-1).
- [ ] Compresión de fotos contra MinIO real verificada en servidor (FX-BE-7).

## Pendientes para otras fases

- [ ] F-INF-1 (backups), FX-BE-3 y FX-BE-4 (`HEALTHCHECK`) se atienden en [[rh-cotla-fase-12-despliegue-sub-plan]] y en 1.0.1.
- [ ] FX-FE-2 exige un cambio coordinado en backend y panel (cookie `httpOnly`).

## Relacionado

- [[rh-cotla-plan]] — plan padre con el estado general.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos del backend; §16, §27 a §31 y §32 (deuda).
- [[rh-cotla-integracion-clientes-manual]] — contrato para web y móvil; «Auditoría», «Caché y fotos», «CORS» y «Límite de intentos».
- [[rh-cotla-pendientes-sub-plan]] — deuda y trabajo futuro (FX-BE-2, FX-BE-3, FX-FE-1, FX-FE-2, F-INF-1).
- [[rh-cotla-fase-04-checador-sub-plan]] — origen de las selfies de asistencia.
- [[rh-cotla-fase-07-horas-extra-sub-plan]] — origen de las fotos de evidencia.
- [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]] — fase anterior; comparte Redis y el stream SSE.
- [[rh-cotla-fase-12-despliegue-sub-plan]] — fase siguiente: despliegue, compose y variables.
