---
tipo: sub-plan
titulo: "App móvil (Expo) — Plan por fases"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: completado
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - movil
  - expo
  - fases
---
# App móvil (Expo) — Plan por fases

> **Plan padre:** [[rh-cotla-plan]]
> **Nota (2026-11-20):** este plan por app queda como respaldo. El estado vigente vive en los sub-planes por fase (tabla «Estado Actual» de [[rh-cotla-plan]]) y la deuda en [[rh-cotla-pendientes-sub-plan]].

> Alcance: `app-movil/` (Expo SDK 57, React Native, TypeScript).
> Fuente funcional: `docs/especificaciones.md` (v1.4), sobre todo §20 (pantallas móviles), §4 (autenticación), §5.3 (auto-registro), §6 (checador), §7 (ausencias), §8 (saldos), §9 (horas extra), §10 (alertas), §13 (calendario) y §18 (manejo de internet).
> Dependencias de API: [[rh-cotla-progreso-backend-sub-plan]] y [[rh-cotla-integracion-clientes-manual]].
> Plan del backend: [[rh-cotla-backend-sub-plan]]. Plan del panel web: [[rh-cotla-frontend-sub-plan]].

Leyenda: ✅ hecho · 🟡 parcial · ⏳ pendiente · 🔓 pendiente y desbloqueado (el backend ya tiene el contrato) · 🔒 pendiente y bloqueado por backend.

---

## Índice

| Fase | Estado | Pantallas / qué ataca (especificación) | Backend | Detalle |
|------|:------:|----------------------------------------|---------|---------|
| 2 — Autenticación, cuentas y colaboradores | 🟡 | §20 login, recuperación, auto-registro, cambio de contraseña; §4, §5.3; «Mi perfil» ✅ (solo lectura) | ✅ | [→](#fase-2--autenticación-cuentas-y-colaboradores) |
| 4 — Checador de asistencia | 🟡 | §6.3 cámara frontal ✅, §6.4 GPS ✅, §6.2 jornada ✅, historial con foto y detalle ✅; errores GPS ⏳; offline §18 pendiente | ✅ (offline pendiente) | [→](#fase-4--checador-de-asistencia) |
| 5 — Incidencias y correcciones | ✅ | Solicitar corrección de una marcación (§6.8) — `correccion-nueva.tsx`, botón desde detalle | ✅ desplegado 13 nov (FASE 5) | [→](#fase-5--incidencias-y-correcciones) |
| 6 — Ausencias y vacaciones | ✅ | Solicitar ausencia, mis solicitudes con estado y cancelación (§7); saldo ✅ (§8) | ✅ desplegado 13 nov (FASE 6) | [→](#fase-6--ausencias-y-vacaciones) |
| 7 — Horas extras | 🟡 | Nueva hora extra con hasta 5 fotos, detalle y estado (§9) — `horas-extra-nueva.tsx` ✅, listado ✅ | ✅ Fase 7 completa | [→](#fase-7--horas-extras) |
| 9 — Dashboards y calendario | ✅ | Calendario personal (§13) — rejilla mensual con indicadores, leyenda y eventos del día | ✅ Fase 9 completa | [→](#fase-9--dashboards-y-calendario) |
| 10 — Alertas y notificaciones | ✅ | Bandeja de notificaciones con contador no leídas (§10) — `notificaciones.tsx`, badge en «Más» | ✅ Fase 10 completa | [→](#fase-10--alertas-y-notificaciones) |
| 12 — Despliegue | 🟡 | Build Android release local ✅; builds EAS, Google Play y App Store (§29) ⏳ | 🟡 | [→](#fase-12--despliegue) |

**Estado al 20 de noviembre de 2026:** el backend está completo y las fases 5, 6, 9 y 10 están implementadas en la app; las fases 2, 4 y 7 quedan 🟡 por los pendientes anotados en cada una. El trabajo restante es publicación (Fase 12), pruebas automatizadas, offline (§18) y las nuevas features propuestas (biométrico).

---

## Pulido para publicación (20 de noviembre de 2026)

- [x] Icono, adaptive icon, monocromo, splash y favicon con el logo CT del panel web (fondo `#F2D21A`); assets de plantilla de Expo eliminados.
- [x] Metadatos: nombre «Cotla Colaborador», slug `cotla-colaborador`, esquema `cotla://`, bundle/package `com.yefrasoft.cotla`, `buildNumber` 1 / `versionCode` 1, versión **1.0.0** (fuente única: `app.json`, leída en `src/lib/app-info.ts`).
- [x] Sin `RECORD_AUDIO`; `ITSAppUsesNonExemptEncryption: false`.
- [x] Login limpio: sin banner, sin biométrico, sin turno/«Red Segura»/«Mantener sesión» (eran maqueta); logo casco; solo número de empleado numérico (`POST /auth/login` no acepta correo); auto-registro destacado.
- [x] Retiradas las referencias «§x.x» visibles al usuario y la fila «Servidor».
- [x] Créditos discretos en Cuenta.
- [x] Aviso de privacidad: enlace en el pie del login y en Cuenta que abre `https://by.yefrasoft.com/privacidad` (página pública del panel; constante `PRIVACY_URL` en `src/lib/app-info.ts`). Esa misma URL se captura en App Store Connect y Play Console.

- [x] Inicio (`(tabs)/index.tsx`) era una vista estática sin scroll y cortaba el contenido: ahora usa `ScrollView` como `explore.tsx`.
- [x] Retirados restos de plantilla de Expo (`hint-row`, `external-link`, `collapsible`, `reset-project`) y reescrito `README.md`.
- [x] Icono, splash y login verificados en dispositivo.

### Tareas de desarrollo / entrega (no son deuda)

- [ ] Heap de Gradle: `prebuild --clean` restaura `-Xmx2048m` y el release falla con `OutOfMemoryError`; hoy se ajusta a mano (`-Xmx6g`).
- [ ] EAS: por el cambio de slug, ejecutar `eas init` para re-vincular el proyecto.
- [ ] Tiendas: correo de soporte, capturas y ficha (la URL de privacidad ya está: `https://by.yefrasoft.com/privacidad`).

### Nuevas features propuestas

- [ ] Acceso biométrico (huella / Face ID) con `expo-local-authentication` y token en SecureStore. Se retiró de la UI porque no era funcional; entra como feature nueva.

### Deuda técnica abierta

- [x] **Recuperación de contraseña:** decisión del 20 de noviembre de 2026: el backend **no se modifica**; `PasswordReset:BaseUrl` sigue apuntando al panel web (compartido con RH) y el enlace del correo se abre en el navegador. La pantalla `reset-password` de la app queda disponible por `cotla://reset-password`, pero ningún correo la usa hoy. Si se quiere abrir la app desde el correo habría que añadir un canal móvil (`MobileBaseUrl`) y un enlace puente https, porque los clientes de correo no vuelven clicable un esquema propio.
- [ ] **Lint y pruebas:** `eslint` no está instalado (`expo lint` falla), `tsc` marca `@/global.css` en `theme.ts` y no hay Jest.
- [ ] **Comentarios:** los JSDoc aún citan «FASE n §x» (no visibles al usuario).

### Futuras correcciones (auditoría del 20 de noviembre de 2026)

Hallazgos de una revisión del código contra este plan. Quedan **registrados como corrección futura (post 1.0.0)**; ninguno está resuelto. Los marcados con ⚠ conviene resolverlos antes de enviar a las tiendas.

**Bloqueantes de publicación**
- [ ] ⚠ **Eliminación de cuenta:** la app permite auto-registro (`(auth)/self-register.tsx`) pero no tiene forma de iniciar la eliminación de cuenta, que Apple exige en ese caso (guía 5.1.1(v); verificar la vigente). El backend no tiene endpoint de borrado y la política de privacidad dice «solicitud por correo a RH». Decidir: botón «Solicitar eliminación de cuenta» que abra un correo o página a RH, o documentar el flujo ante las tiendas.
- [ ] ⚠ **API en el build de producción:** `eas.json` define `EXPO_PUBLIC_API_URL` solo en `preview`; `production` no tiene `env` y `.env` está ignorado. Sin la variable en EAS, la app falla con «Falta EXPO_PUBLIC_API_URL». Verificar con `eas env:list` o añadir `env` al perfil `production`.
- [ ] ⚠ **Galería prometida y no implementada:** `horas-extra-nueva.tsx:325` dice «seleccionar de la galería», pero solo existe la cámara y `expo-image-picker` no está en `package.json`. Quitar el texto (y el comentario de la línea 50) o implementar el selector.

**Importantes**
- [ ] `expo-dev-client` está en `dependencies` (`package.json:11`): puede quedar en el binario de tienda. Moverlo a `devDependencies` o limitarlo al perfil `development`, y revisar el binario de release.
- [ ] Enlace al aviso de privacidad solo en login y Cuenta; falta en el checador (selfie y GPS), el auto-registro y las solicitudes con documento de incapacidad. Verificar si la especificación (§18, §29) lo exige; usar `abrirAvisoPrivacidad` de `src/lib/app-info.ts`.
- [ ] Dependencias que parecen sin uso: `expo-glass-effect`, `expo-symbols`, `expo-device`, `expo-linking`, `expo-font`, `expo-system-ui`, `expo-status-bar`, `@expo/ui`, `expo-modules-core`. Comprobar con `depcheck` antes de quitarlas (algunas pueden ser requeridas por `expo-router`).
- [ ] Conectividad: el error de red es genérico y no hay reintento ni detección de conexión (el reintento idempotente del checador cubre solo ese flujo).

**Menores**
- [ ] Cinco `as any` por tipado de rutas (`explore.tsx:206`, `horas-extra-nueva.tsx:176`, `horas-extra.tsx:74` y `:239`, `asistencia-detalle.tsx:408`): tipar `href` con `Href`.
- [ ] `src/lib/app-info.ts:18` ignora errores al abrir el navegador (`.catch(() => {})`).
- [ ] `console.warn` en `src/lib/auth-context.tsx:169`: quitarlo o dejarlo tras `__DEV__`.
- [ ] `buildNumber` y `versionCode` fijos en `app.json` y, a la vez, `autoIncrement` con `appVersionSource: remote` en `eas.json`: dejar una sola fuente de verdad.
- [ ] Los nombres de los desarrolladores en `CREDITOS` (`src/lib/app-info.ts`) quedan en el binario público: confirmar que se quieren visibles.
- [ ] Solo versión web: el token va en `localStorage` (`session-storage.web.ts`) y `src/global.css` importa Google Fonts por URL remota (rompe el typecheck). No desplegar la web o aceptar el riesgo por escrito.

**Documentación de este plan por corregir** (verificado en el código, pero no probado en dispositivo)
- [ ] `:116` dice que el reintento usa `expo-crypto`; el UUID es TypeScript puro (`src/lib/uuid.ts`).
- [ ] `:117` dice que GPS apagado y baja precisión no se distinguen; `checador.tsx:222` ya usa `hasServicesEnabledAsync` y distingue precisión. Confirmar en dispositivo y marcar.
- [ ] `:156` dice que falta la pantalla de correcciones; existe `src/app/correcciones.tsx`, enlazada desde `(tabs)/explore.tsx`.
- [ ] `:196` dice que falta el detalle de horas extra; existe `src/app/horas-extra-detalle.tsx`. Verificar que muestre las fotos.
- [ ] `:191` marca «cámara o galería» como hecho; solo hay cámara.
- [ ] No se verificaron `android/`, `ios/`, el heap de Gradle ni las variables de EAS.

## Pendientes globales de la app

- [ ] Corregir la línea de estado del plan anterior: el checador **no** está bloqueado por FASE 4; el backend ya lo cubre.
- [x] Reescribir `README.md` (plantilla de create-expo-app): documentar el checador, `EXPO_PUBLIC_API_URL`, `eas.json` y el esquema `cotla://`. Ojo: menciona `reset-project`, que borraría `src/app`.
- [ ] Añadir pruebas automatizadas (hoy no hay `test`, ni Jest, ni `tsc` en scripts).
- [x] Recorrido de la app en build: el usuario verificó icono, splash, login e Inicio en dispositivo (20 nov 2026)

---

## Fase 2 — Autenticación, cuentas y colaboradores

> **Estado: 🟡.** Todo lo de §2.5 completo, incluida «Mi perfil» (solo lectura).
> La edición de datos queda pendiente de decisión del cliente y de un endpoint del backend.

### 2.5 Pantallas de la app

- [x] Login — `(auth)/login.tsx`
- [x] Recuperación de contraseña — `(auth)/forgot-password.tsx` y `(auth)/reset-password.tsx` (lee `?token=`)
- [x] Cambio de contraseña — `change-password.tsx`, con entrada desde Inicio
- [x] Auto-registro (estado `Pendiente`) — `(auth)/self-register.tsx`
- [x] Cierre de sesión desde Inicio, que borra el token del dispositivo
- [x] Sesión persistente con `expo-secure-store` (nativo) y `localStorage` (web); se revalida contra `GET /auth/me` al arrancar
- [x] «Mi perfil» como pantalla propia — `profile.tsx`: expediente en lectura vía
  `GET /employees/{empleadoId}` (§5.1 datos personales + §5.2 laborales), entrada desde
  Inicio y desde «Más».
  - [ ] Edición de datos personales: **dependencia de backend**. Hoy solo existe
    `POST /auth/change-password`; no hay ruta para que un usuario cambie correo o teléfono
    (`integracion-clientes.md` §«Lo que el backend todavía no expone»). Preguntar al
    cliente quién debe editarlos (§5.1 no lo define) antes de construir el formulario.

> **Notas de la fase:**
> - **Enlace de recuperación:** el correo abre el panel web (decisión: backend sin cambios). Esta pantalla solo se alcanza por `cotla://reset-password`.
> - **Entrada por enlace no verificada:** el launcher de dev ignora la ruta del deep link. Confirmar en build de producción.

---

## Fase 4 — Checador de asistencia

> **Estado: 🟡.** El checador y el historial con detalle están completos; faltan los
> errores específicos de GPS y el offline.

### 4.3 Captura de selfie y GPS

- [x] Cámara frontal obligatoria (`CameraView facing="front"`) — §6.3
- [x] Sin selección desde galería (no hay `expo-image-picker` en el checador)
- [x] Permiso de ubicación y envío de latitud, longitud y precisión — §6.4
- [x] Reintento idempotente con UUID (`expo-crypto`), reutilizado al reintentar
- [ ] Errores: GPS apagado y baja precisión no se distinguen hoy (solo permiso y red) — §19
- [x] Errores de cámara sin permiso y de red

### 4.4 Jornada e historial

- [x] Jornada del día con `siguientesPermitidas`, `faltantesObligatorias` y horario (`checador.tsx`); hora en `America/Mexico_City`
- [x] Marcaciones obligatorias faltantes del día
- [x] 🔓 **Historial de asistencia** con hora, tipo, zona y foto — `historial.tsx`
  (`GET /attendance`, filtros por tipo, paginación «Cargar más»). Entrada desde el
  checador y desde «Más».
- [x] 🔓 Detalle de una marcación con foto y zona — `asistencia-detalle.tsx`
  (`GET /attendance/{id}` + `GET /attendance/{id}/photo` como Blob → data URI vía
  `src/lib/blob.ts`; muestra retardo, precisión, coordenadas, radio y
  `correccionAprobada` si RH aprobó una corrección).
  - Nota: en el emulador la cámara virtual sin escena genera selfies negras (solo
    visible el timestamp); el flujo de descarga y render está verificado.

### 4.5 Sincronización offline — pendiente de decisión

> **D7 provisional:** solo en línea con reintento idempotente. La estrategia offline la define el cliente (§18, §31).

- [ ] Confirmar con el cliente: offline total o solo tolerancia de cortes
- [ ] Si aplica: almacenamiento local y sincronización (depende de backend, `plan-backend.md` Fase 4.5)
- [ ] Política de conflictos (hora del dispositivo, GPS, duplicados)

---

## Fase 5 — Incidencias y correcciones

> **Estado: ✅.** Cliente `POST /attendance-corrections` completo. Botón «Solicitar
> corrección» desde el detalle de marcación, formulario con motivo obligatorio y
> hora/tipo propuestos opcionales. La marcación original se conserva intacta.

### 5.3 Móvil

- [x] 🔓 Solicitar corrección de una marcación (motivo, marcación relacionada) — §6.8
  - API: `src/lib/api-corrections.ts` (`createCorrection`, `listCorrections`, `getCorrection`)
  - Pantalla: `src/app/correccion-nueva.tsx` — formulario con motivo, hora y tipo propuestos
  - Entrada: botón en `asistencia-detalle.tsx` cuando no hay corrección aprobada
- [ ] Listar correcciones del usuario (pendiente de pantalla)

---

## Fase 6 — Ausencias y vacaciones

> **Estado: ✅.** Saldo (lectura) ✅; solicitudes completas ✅ (13 nov 2026), una vez que
> el backend desplegó FASE 6 (`/leave-requests`). Contrato en `integracion-clientes.md`.

### 6.6 Pantallas de ausencias

- [x] Vacaciones y saldo, lectura (`vacaciones.tsx`: `GET /vacation-balances/{id}` y `/movements`) — §8
- [x] Solicitar ausencia (vacaciones, permisos, incapacidad con documento adjunto) — §7
  (`solicitud-nueva.tsx` + `src/lib/api-leave.ts`, multipart con `File` de
  `expo-file-system`; documento obligatorio en incapacidad §7.3, tomado con la cámara;
  fechas validadas como no pasadas y `fin ≥ inicio` en el cliente, la autoridad es el
  servidor).
- [x] Mis solicitudes: lista con estado (`solicitudes.tsx`, `GET /leave-requests`)
- [x] Detalle de solicitud, con motivo de rechazo visible — §7.4 (en la tarjeta de la
  lista; no hay pantalla de detalle aparte)
- [x] Cancelar solicitud pendiente — §7.5 (`POST /leave-requests/{id}/cancel` con
  confirmación; `409` si ya fue revisada)

> Nota: **solo lectura de estados del colaborador**. Aprobar/rechazar es de RH y vive
> en el panel web, no aquí.

---

## Fase 7 — Horas extras

> **Estado: 🟡.** Pantalla de registro y listado completas. Falta el detalle
> individual con fotos descargables. Cliente API completo.

### 7.2 Evidencias

- [x] 🔓 Adjuntar hasta **5 fotografías** por solicitud de horas extra desde cámara o galería (§9.1) — `expo-image-picker` + `expo-camera` en `horas-extra-nueva.tsx`

### 7.3 Pantallas de horas extra

- [x] 🔓 Nueva hora extra (fecha, hora inicial y final, concepto); el total lo calcula el backend (§9.2) — `src/app/horas-extra-nueva.tsx`
- [ ] 🔓 Detalle y evidencias (fotos descargables) — pendiente de pantalla de detalle
- [x] 🔓 Listado de horas extra con estado — `src/app/horas-extra.tsx` (`GET /overtime`)

---

## Fase 9 — Dashboards y calendario

> **Estado: 🔓. Desbloqueada el 13 nov 2026:** el backend completó FASE 9
> (`GET /api/v1/calendar`, etc., ver `integracion-clientes.md`
> §«Dashboard y calendario»).

### 9.4 Calendario personal

- [x] 🔓 Calendario con ausencias aprobadas, asistencia e incidencias, horas extra aprobadas (§13)
  - API: `src/lib/api-calendar.ts` (`getCalendar`)
  - Pantalla: `src/app/calendario.tsx` — rejilla mensual, indicadores por categoría, leyenda, lista de eventos del día seleccionado
  - Entrada: enlace «Calendario» en la pestaña «Más»

---

## Fase 10 — Alertas y notificaciones

> **Estado: 🔓. Desbloqueada el 13 nov 2026:** el backend completó FASE 10
> (`GET /notifications`, etc., ver `integracion-clientes.md` §«Notificaciones»).
> El push móvil queda descartado por ahora (solo bandeja + correo).

### 10.5 Alertas

- [x] 🔓 Pantalla de alertas y cumpleaños (§10) — `src/app/notificaciones.tsx` (`GET /notifications`, contador `unread-count`)
- [x] 🔓 Marcar como leídas — `POST /notifications/{id}/read` y `POST /notifications/read-all`
  - API: `src/lib/api-notifications.ts` (`listNotifications`, `getUnreadCount`, `markNotificationRead`, `markNotificationUnread`, `markAllNotificationsRead`)
  - Pantalla: lista con estado, badge de no leídas en pestaña «Más» (`explore.tsx`)
  - Tocar una notificación: alterna leída/no leída

---

## Fase 12 — Despliegue

> **Estado: 🟡.** App con identidad, icono, metadatos y aviso de privacidad listos; el build Android release local se compiló (ver README). Falta la publicación.

- [x] Identidad de tienda: icono, nombre, bundle/package, versión 1.0.0 y aviso de privacidad (sección «Pulido para publicación»)
- [x] Build Android release local (`prebuild` + `./gradlew assembleRelease`; heap de Gradle documentado en README)
- [ ] Builds reproducibles con EAS (`eas.json`; requiere `eas init` por el cambio de slug)
- [ ] Publicación en Google Play y App Store (§29)
- [ ] Smoke test de la app contra la API desplegada
- [x] Deep link de recuperación: ya no aplica; el correo abre el panel web (decisión del 20 de noviembre de 2026)

---

## Dependencias con otras apps y con el backend

- La app no modifica `app-backend` ni `app-frontend`. Si necesita un endpoint o cambio de configuración, se anota en `plan-backend.md` y se pide al usuario que coordine.
- `PasswordReset:BaseUrl` del canal móvil: se mantiene el panel web (decidido el 20 de noviembre de 2026).
- Lo que la app consume de los endpoints existentes está en `src/lib/api-*.ts` y en [[rh-cotla-integracion-clientes-manual]].
