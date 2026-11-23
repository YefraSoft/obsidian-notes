---
tipo: sub-plan
titulo: "Pendientes — Deuda técnica centralizada (feats y fixes)"
autor: Efraín García
creado: 2026-11-20
actualizado: 2026-11-20
estado: pendiente
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - pendientes
  - deuda-tecnica
---

# Pendientes — Deuda técnica centralizada (feats y fixes)

> **Plan padre:** [[rh-cotla-plan]]
> Punto único de la deuda de backend, panel web y móvil tras cerrar la 1.0.0. Se divide en **Feats** (funcionalidad nueva o mejora) y **Fixes** (corrección, endurecimiento o deuda). Origen: bitácora del PM (`feats.md`, retirada del repo), [[rh-cotla-backend-sub-plan]], [[rh-cotla-progreso-backend-sub-plan]] (registro A/P), [[rh-cotla-frontend-sub-plan]] y [[rh-cotla-movil-sub-plan]].
> Leyenda de app: **BE** backend · **FE** panel web · **MOV** móvil · **INF** infraestructura.

## Pendientes

- [ ] Antes de enviar a las tiendas: resolver los tres ⚠ de móvil (eliminación de cuenta, `EXPO_PUBLIC_API_URL` en producción, galería).
- [ ] Antes de acumular datos reales: backups automáticos de PostgreSQL (F-INF-1).
- [ ] Definir la regla de ausentes del dashboard (FX-BE-1) con el cliente.

## Estado Actual

| Bloque | Abiertos |
|--------|----------|
| Feats | 17 |
| Fixes | 38 |
| Deuda aceptada (no se corrige) | 2 |

## Estado Actual por fase

Cada ítem nació en una fase; su sub-plan lleva el contexto y los criterios para cerrarlo.

| Fase | Feats abiertos | Fixes abiertos |
|------|----------------|----------------|
| [[rh-cotla-fase-02-autenticacion-colaboradores-sub-plan]] | `F-BE-3` contactos de emergencia, `F-BE-2` búsqueda de empleados, `F-MOV-1` biometría, `F-FE-4` vista móvil de empleados | `FX-MOV-1` eliminación de cuenta (⚠), `FX-BE-5` 503 sin SMTP, `FX-BE-10` enlace de recuperación en móvil |
| [[rh-cotla-fase-04-checador-sub-plan]] | — | `FX-BE-6` aviso de imports en pruebas del horario, `FX-MOV-5` enlace de privacidad en el checador |
| [[rh-cotla-fase-05-incidencias-correcciones-sub-plan]] | `F-BE-4` «Detectar ahora» con fuera de zona | — |
| [[rh-cotla-fase-06-ausencias-vacaciones-sub-plan]] | `F-BE-1` colisiones de permisos | — |
| [[rh-cotla-fase-07-horas-extra-sub-plan]] | `F-BE-5` eventos de horas extra al panel | `FX-MOV-3` galería prometida (⚠) |
| [[rh-cotla-fase-09-dashboards-calendario-sub-plan]] | `F-FE-2`, `F-FE-3` calendario del panel | `FX-BE-1` ausentes del dashboard |
| [[rh-cotla-fase-10-alertas-notificaciones-sub-plan]] | `F-BE-6` push móvil | `FX-BE-8` una sola instancia (pub/sub en memoria) |
| [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]] | `F-BE-7` OpenAPI del release | `FX-BE-2` revocación de sesiones, `FX-BE-3` rate limit y `/diagnostics/*`, `FX-BE-7` verificación real de fotos y SMTP, `FX-FE-1` CSP, `FX-FE-2` token en `localStorage` |
| [[rh-cotla-fase-12-despliegue-sub-plan]] | `F-INF-1` backups, `F-INF-2` CI/CD, `F-INF-3` carga inicial | `FX-BE-4` healthchecks y tag de MinIO, `FX-BE-9` demo 201–203, `FX-FE-3` puerto 3000, `FX-FE-9` imagen del panel, `FX-FE-11` recorrido con backend real, `FX-MOV-2` API en EAS (⚠), `FX-MOV-14` entrega a tiendas |
| Transversal | `F-FE-1` botón de ayuda, `F-FE-5` UX móvil del panel, `F-BE-8` decisiones del cliente (§31) | `FX-FE-4` a `FX-FE-8`, `FX-FE-10`, `FX-MOV-4`, `FX-MOV-6`, `FX-MOV-7` a `FX-MOV-13`, `FX-DOC-1` a `FX-DOC-3` |

## Decisiones de Diseño

- **Deuda aceptada** (decisión del usuario, no se corrige salvo cambio de contexto): **A1** CORS abierto (`AllowAnyOrigin`; autenticación por JWT en cabecera, sin cookies) y **A2** puerto `8080/8088` publicado con `X-Forwarded-For` confiado desde redes privadas (permite falsear IP y saltar el límite de 10 intentos/min desde la LAN).
- Regla de nomenclatura: `F-<app>-n` para feats, `FX-<app>-n` para fixes.

---

# FEATS

## Del PM (`feats.md`)

- [ ] **F-BE-1 · BE+FE** Manejo de colisiones en solicitudes de permisos.
- [ ] **F-BE-2 · BE+FE** Búsqueda de empleados: cachear la respuesta y tolerar variaciones de nombre (`Carlos` → `carlos`).
- [ ] **F-BE-3 · BE+FE** Aceptar contactos de emergencia en la información del colaborador (la entidad existe; falta endpoint y campo en `POST/PATCH /employees`).
- [ ] **F-BE-4 · BE+FE** «Detectar ahora» debe traer también las incidencias fuera de zona.
- [ ] **F-FE-1 · FE** Dar funcionamiento al botón de ayuda: incrustar el manual de uso como recurso.
- [ ] **F-FE-2 · FE** Calendario: reducir el *bouncing* del nombre de los meses (ancho fijo o mínimo; revisar en móvil) y mejorar la vista móvil.
- [ ] **F-FE-3 · FE** Cabecera de la tabla del calendario: evitar saltos al cambiar de mes.
- [ ] **F-FE-4 · FE** Mejorar la vista móvil del panel de empleados.
- [ ] **F-FE-5 · FE** Mejorar la experiencia UI en móvil, sobre todo cabeceras de tablas.

## Infraestructura y plataforma

- [ ] **F-INF-1 · INF** Backups automáticos de PostgreSQL con destino fuera del servidor y restauración probada (hoy: `pg_dump` manual).
- [ ] **F-INF-2 · INF** CI/CD, registry de imágenes con etiquetado por versión, smoke tests automatizados, plan de rollback formal, hosting definitivo y TLS propio.
- [ ] **F-INF-3 · INF** Documentar la carga de configuración inicial en producción (tienda, zona, horarios, saldos, usuarios).

## Móvil

- [ ] **F-MOV-1 · MOV** Acceso biométrico (huella / Face ID) con `expo-local-authentication` y token en SecureStore (se retiró de la UI por no ser funcional).

## Backend (v2)

- [ ] **F-BE-5 · BE** Eventos de horas extra hacia el panel (SSE) y bus (Redis) detrás de `PanelBroker` si hay más de una instancia.
- [ ] **F-BE-6 · BE** Push móvil (FCM): descartado por ahora; se conserva el registro de tokens (`dispositivos_push`).
- [ ] **F-BE-7 · BE** Exportar el OpenAPI como artefacto del release (en Production no se publica).
- [ ] **F-BE-8 · BE** Decidir con el cliente (§31): devengo de vacaciones (D1), D6 y D7 (ventana horaria, antigüedad máxima sin conexión, margen de reloj), corrección directa de marcación, asignación de gerentes, proveedor y conservación de archivos, canal de notificaciones.

---

# FIXES

## Backend

- [ ] **FX-BE-1 · BE+FE** Dashboard de asistencias: el conteo de ausentes incluye a RH y gerente (sin horario) y no considera día laboral ni hora de entrada; con base limpia aparecen 2 ausentes (111 y 112). Hoy: activos − presentes − de licencia (`DashboardService.cs:54`). Definir regla (¿excluir RH/gerente o quien no tenga horario?).
- [ ] **FX-BE-2** Revocación real de sesiones: `jti` en `sesiones_usuario` y validación en `OnTokenValidated`; hoy un usuario dado de baja conserva el token hasta 60 min (P2, prioridad alta, 1.0.1).
- [ ] **FX-BE-3** Rate limit general (hoy solo en autenticación) y proteger o retirar `/diagnostics/*` (P5).
- [ ] **FX-BE-4** `HEALTHCHECK` en Dockerfile/compose y fijar el tag de la imagen de MinIO (hoy `latest`) (P4).
- [ ] **FX-BE-5** Decidir si `POST /users/{id}/password-reset` responde 503 cuando SMTP no está configurado (hoy 204 sin enviar).
- [ ] **FX-BE-6** Aviso de orden de imports en `AttendanceHorarioBloqueoTests.cs`.
- [ ] **FX-BE-7** Verificar tras desplegar fases 10 y 11: compresión real de fotos contra MinIO, caché a través del túnel y envío SMTP real.
- [ ] **FX-BE-8** Una sola instancia del API: el pub/sub del panel está en memoria (P7, baja hasta escalar).
- [ ] **FX-BE-9** Operativa: si el servidor tiene los colaboradores demo 201–203 sembrados antes del cambio, desactivarlos manualmente (P8).
- [ ] **FX-BE-10** `PasswordReset:BaseUrl` apunta al panel web; el enlace del correo no abre la app móvil. Si se quiere, añadir `MobileBaseUrl` y un enlace puente https.

## Panel web

- [ ] **FX-FE-1** CSP: falta `Content-Security-Policy` (scripts en línea de Next + teselas OSM; requiere nonces y pruebas).
- [ ] **FX-FE-2** Token en `localStorage` con «mantener sesión» (`lib/auth.ts`); a futuro cookie `httpOnly` (cambio en backend).
- [ ] **FX-FE-3** Puerto 3000 publicado en todas las interfaces; decidir si se limita a loopback (el APK se sirve en `/descargas`).
- [ ] **FX-FE-4** Sin pruebas automatizadas (solo `typecheck` y lint): smoke tests de login, guard de sesión y clientes de API.
- [ ] **FX-FE-5** Protección de rutas solo en cliente (`components/session-guard.tsx`); documentarlo o añadir `proxy`.
- [ ] **FX-FE-6** ~20 etiquetas en español fuera de los `*.text.ts` (`lib/navegacion.ts`).
- [ ] **FX-FE-7** Accesibilidad: 7 `eslint-disable no-img-element` y un `alt` que es un nombre de archivo (`horas-extra-detalle.tsx`).
- [ ] **FX-FE-8** 2 `eslint-disable react-hooks/exhaustive-deps` en `mapa-geocerca.tsx` y `mapa-marcacion.tsx`.
- [ ] **FX-FE-9** Imagen runtime copia todo `node_modules` y no tiene `HEALTHCHECK`.
- [ ] **FX-FE-10** Revisar `ignoreScripts` de `package.json`, `@types/node ^20` y dependencias desactualizadas (`bun outdated`).
- [ ] **FX-FE-11** Recorrido en DevTools de todas las pantallas con el backend real.

## Móvil

- [ ] ⚠ **FX-MOV-1** Eliminación de cuenta: hay auto-registro pero no flujo de eliminación (Apple 5.1.1(v)); decidir botón «Solicitar eliminación» que abra correo/página a RH, o documentar el flujo ante las tiendas.
- [ ] ⚠ **FX-MOV-2** `eas.json`: `EXPO_PUBLIC_API_URL` solo en `preview`; añadir `env` al perfil `production` (verificar con `eas env:list`).
- [ ] ⚠ **FX-MOV-3** `horas-extra-nueva.tsx:325` promete «seleccionar de la galería» pero solo hay cámara; quitar el texto o implementar el selector.
- [ ] **FX-MOV-4** `expo-dev-client` en `dependencies`: moverlo a `devDependencies` o limitarlo al perfil `development`.
- [ ] **FX-MOV-5** Enlace al aviso de privacidad también en checador, auto-registro y solicitudes con documento (`abrirAvisoPrivacidad`).
- [ ] **FX-MOV-6** Dependencias aparentemente sin uso (`expo-glass-effect`, `expo-symbols`, `expo-device`, `expo-linking`, `expo-font`, `expo-system-ui`, `expo-status-bar`, `@expo/ui`, `expo-modules-core`): comprobar con `depcheck`.
- [ ] **FX-MOV-7** Conectividad: error de red genérico, sin reintento ni detección de conexión.
- [ ] **FX-MOV-8** Cinco `as any` por tipado de rutas: tipar `href` con `Href`.
- [ ] **FX-MOV-9** `app-info.ts:18` ignora errores al abrir el navegador; `console.warn` en `auth-context.tsx:169`.
- [ ] **FX-MOV-10** `buildNumber`/`versionCode` fijos y a la vez `autoIncrement` remoto: dejar una sola fuente.
- [ ] **FX-MOV-11** Confirmar que los nombres de `CREDITOS` se quieren visibles en el binario público.
- [ ] **FX-MOV-12** Solo versión web: token en `localStorage` y `global.css` importa Google Fonts remoto; no desplegar la web o aceptar el riesgo por escrito.
- [ ] **FX-MOV-13** `eslint` no instalado (`expo lint` falla), `tsc` marca `@/global.css` y no hay Jest; los JSDoc aún citan «FASE n §x».
- [ ] **FX-MOV-14** Entrega: heap de Gradle (`prebuild --clean` restaura `-Xmx2048m`), `eas init` por el cambio de slug, correo de soporte/capturas/ficha de tiendas.

## Documentación por corregir

- [ ] **FX-DOC-1** En [[rh-cotla-movil-sub-plan]]: el reintento no usa `expo-crypto` (UUID en TypeScript puro); existen `correcciones.tsx` y `horas-extra-detalle.tsx`; «cámara o galería» solo es cámara; confirmar en dispositivo la distinción GPS apagado vs baja precisión.
- [ ] **FX-DOC-2** En [[rh-cotla-frontend-sub-plan]]: menú de Fase 10 (solo «Ayuda» deshabilitada), Fase 12 (el servicio `frontend` ya entra en el `up` normal) y nota ¹ del índice (rutas sin `/api/v1`).
- [ ] **FX-DOC-3** Cerrar la fila «Dockerfile del panel» del backlog: alinear versión de bun (ya 1.4.4).

## Dependencias y Orden de Ejecución

1. Tiendas: FX-MOV-1 → FX-MOV-2 → FX-MOV-3 → FX-MOV-14.
2. 1.0.1 (seguridad): FX-BE-2, F-INF-1, FX-BE-3, FX-BE-4.
3. Producto: FX-BE-1, F-BE-1 a F-BE-4 (coordinar BE y FE; el backend va primero).
4. UI móvil del panel: F-FE-2 a F-FE-5.
5. Calidad: FX-FE-4, FX-MOV-13, F-INF-2.

## Relacionado

- [[rh-cotla-plan]] — plan padre: reglas de trabajo, diagramas, estado por fase y decisiones vigentes.
- [[rh-cotla-progreso-backend-sub-plan]] — registro de deuda aceptada (A1–A2) y pendiente (P1–P8) de donde salen los fixes del backend, con el runbook de despliegue.
- [[rh-cotla-backend-sub-plan]] — backlog de la versión 2 del backend (origen de varios feats).
- [[rh-cotla-frontend-sub-plan]] — «Futuras correcciones» del panel (auditoría del 2026-11-20), origen de los `FX-FE`.
- [[rh-cotla-movil-sub-plan]] — «Futuras correcciones» de la app móvil (auditoría del 2026-11-20), origen de los `FX-MOV`.
- [[rh-cotla-integracion-clientes-manual]] — contrato del API que cambia con `F-BE-1` a `F-BE-4`.
