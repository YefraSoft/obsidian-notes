---
tipo: sub-plan
titulo: "App móvil MAUI (descartada)"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: completado
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - historico
  - maui
---
# App móvil MAUI (descartada)

> **Plan padre:** [[rh-cotla-plan]]

> Alcance: `app-movil-maui/` (.NET MAUI, .NET 10, patrón MVVM).
> Fuente funcional: `docs/especificaciones.md` (v1.4), sobre todo §4 (autenticación) y §20 (pantallas móviles).
> Dependencias de API: [[rh-cotla-progreso-backend-sub-plan]] y [[rh-cotla-integracion-clientes-manual]].
> App de referencia (migración): `app-movil/` (Expo SDK 57) y su plan [[rh-cotla-movil-sub-plan]].

> **Migración gradual.** `app-movil-maui` es el destino de la migración de `app-movil`,
> pero **`app-movil` se conserva intacto como referencia** hasta completar el port.
> Este plan porta el comportamiento, no el código: cada pantalla se reescribe en
> C#/XAML con MVVM.

Leyenda: ✅ hecho · 🟡 parcial · ⏳ pendiente · 🔓 pendiente y desbloqueado · 🔒 bloqueado por backend.

---

## Índice

| Fase | Estado | Pantallas / qué ataca (especificación) | Backend | Detalle |
|------|:------:|----------------------------------------|---------|---------|
| 1 — Login y sesión | 🟡 | §4 / §20.1 login, sesión persistente, restauración con `/auth/me`, cierre de sesión; Home provisional | ✅ | [→](#fase-1--login-y-sesión) |
| 2 — Resto de autenticación | ⏳ | Recuperación de contraseña, auto-registro, cambio de contraseña | ✅ | [→](#fase-2--resto-de-autenticación) |
| 4 — Checador de asistencia | ⏳ | Cámara y GPS, jornada, historial | ✅ (offline ⏳) | [→](#fase-4--checador-de-asistencia) |
| 6 — Ausencias y vacaciones | ⏳ | Solicitudes, saldo | 🔒 Fase 6 | [→](#fase-6--ausencias-y-vacaciones) |
| 12 — Despliegue | ⏳ | Builds Android / iOS | 🔒 | [→](#fase-12--despliegue) |

**Trabajo desbloqueado hoy:** Fase 2 completa (el backend ya expone los contratos) y el Home provisional como pantalla propia.

---

## Pendientes globales de la app

- [ ] Sin pruebas automatizadas (no hay proyecto de test).
- [ ] Verificar el comportamiento real en emulador/dispositivo: login correcto, 401/403 y persistencia tras reinicio.
- [ ] Añadir la aplicación a la tabla de alcance de `AGENTS.md` y decidir el plan de retirada de `app-movil` (requiere autorización del usuario; los documentos compartidos no se editan sin pedirlo).
- [ ] Sustituir el `MainPage` de plantilla por las pantallas reales (ya retirado en Fase 1).

---

## Fase 1 — Login y sesión

> **Estado: 🟡.** Login, sesión persistente y gating hechos; el Home es provisional.

### 1.1 Infraestructura

- [x] Paquetes `CommunityToolkit.Mvvm` (MVVM con source generators) y `Microsoft.Extensions.Http` (DI de `HttpClient`).
- [x] `MauiProgram.cs`: DI de `ApiOptions`, `HttpClient` con pooling de handlers, `SessionStore`, `ApiClient`, `AuthService`, `IAppNavigation`, ViewModels, páginas y `AppShell`.
- [x] `Resources/Raw/appsettings.json`: `Api.BaseUrl` y `Api.TimeoutMs` (config fuera del código).
- [x] `Resources/Styles/CotlaTheme.xaml`: paleta y estilos transcritos de `app-movil/src/constants/theme.ts`.

### 1.2 Capa de datos (paridad con `app-movil/src/lib/api-*.ts`)

- [x] `Models/`: `PerfilUsuario`, `LoginRequest`, `LoginResponse`, `ProblemDetails`, `ApiOptions`.
- [x] `Services/ApiClient.cs`: timeout configurable, `Bearer` en peticiones autenticadas, mapeo de errores en español (400/401/403/404/409/429 + `detail`/`errors`), cierre de sesión local ante un 401.
- [x] `Services/AuthService.cs`: `login`, `getMe`, `logout` (`POST|GET /api/v1/auth`).
- [x] `Services/SessionStore.cs`: token en `SecureStorage` (clave `cotla.accessToken`) y perfil solo en memoria.

### 1.3 MVVM

- [x] `ViewModels/BaseViewModel.cs`: `IsBusy`, `Error`, `HasError`, `IsNotBusy`.
- [x] `ViewModels/LoginViewModel.cs`: `LoginCommand`, `ToggleShowPasswordCommand`, `BiometricCommand` (aún no habilitado), validación de campos vacíos y restauración de sesión con `/auth/me` una sola vez por instancia.
- [x] `ViewModels/HomeViewModel.cs`: resumen del perfil y `LogoutCommand`.
- [x] `Views/LoginPage.xaml` y `Views/HomePage.xaml`: binding puro, sin lógica en code-behind.

### 1.4 Navegación y gating

- [x] `AppShell` con dos raíces: `//login` y `//home` (`FlyoutBehavior="Disabled"`).
- [x] Login navega a `//home` al autenticar; Home cierra sesión y vuelve a `//login`.
- [x] `App.xaml.cs` resuelve `AppShell` por DI.

### 1.5 UI (paridad alta)

- [x] Estructura del login de Expo: aviso de protocolo, branding "CT / Ferretería Cotla", banner de turno, campos con icono, mostrar/ocultar contraseña, "Mantener sesión activa", caja de error, botón "Iniciar Sesión" con indicador de carga y footer.
- [x] Biométrico y auto-registro presentes pero **deshabilitados** (no funcionales en el original).
- [x] Enlace "¿Olvidaste tu contraseña?" deshabilitado (Fase 2).

### 1.6 Android

- [x] `AndroidManifest.xml`: `android:usesCleartextTraffic="true"` (**solo desarrollo**, para `http://10.0.2.2:8080`).
- [x] Retirado el `MainPage` de plantilla (contador).

### Validación de la fase

- [x] `dotnet build -f net10.0-android` sin advertencias ni errores.
- [ ] Prueba manual en emulador contra el backend (ver [Entorno](#entorno-de-compilación)).

> **Notas de la fase:**
> - El comportamiento lógico está portado íntegro desde `auth-context.tsx`; solo queda la verificación en dispositivo.
> - `RememberSession` se replica en la UI, pero la persistencia del token es siempre activa (igual que en la app Expo).

---

## Fase 2 — Resto de autenticación

> **Estado: ⏳. Desbloqueada:** los contratos existen (`/forgot-password`, `/reset-password`, `/change-password`, `POST /employees` para auto-registro).

- [ ] Recuperación de contraseña (`forgot-password` + `reset-password` con token del correo).
- [ ] Auto-registro de colaborador (estado `pendiente`) — §5.3.
- [ ] Cambio de contraseña (`change-password`), con entrada desde Home.
- [ ] «Mi perfil» como pantalla propia (decidir qué datos puede editar el colaborador; §5.1 no lo define).

---

## Fase 4 — Checador de asistencia

> **Estado: ⏳.** Portar el checador de `app-movil` (cámara frontal, GPS, jornada).

- [ ] Cámara frontal y captura de selfie.
- [ ] Permiso y envío de ubicación (latitud, longitud, precisión).
- [ ] Jornada del día (`GET /attendance/today`) y secuencia de marcaciones.
- [ ] Historial con foto (`GET /attendance/{id}/photo`).
- [ ] Estrategia offline (§18) — pendiente de decisión del cliente.

---

## Fase 6 — Ausencias y vacaciones

> **Estado: ⏳. Bloqueada por backend (salvo lectura de saldo ✅).**

- [ ] Saldo de vacaciones (lectura).
- [ ] Solicitar ausencia y ver mis solicitudes.

---

## Fase 12 — Despliegue

- [ ] Builds reproducibles de Android (y iOS cuando haya entorno Apple).
- [ ] Smoke test contra la API desplegada.

---

## Entorno de compilación

- .NET 10 con workload `maui-android`. El SDK de Android (.NET 10 / Android 36) **exige JDK 21**.
  En esta máquina el JDK por defecto es el 25, así que hay que apuntar a un JDK 21:

  ```sh
  JAVA_HOME=/usr/lib/jvm/temurin-21-jdk dotnet build -f net10.0-android
  # o, equivalente:
  dotnet build -f net10.0-android -p:JavaSdkDirectory=/usr/lib/jvm/temurin-21-jdk
  ```

- iOS y MacCatalyst no se pueden compilar en Linux; solo se valida Android por ahora.
- En emulador Android, `10.0.2.2` apunta al `localhost` del host. En dispositivo físico,
  cambiar `Api.BaseUrl` en `Resources/Raw/appsettings.json` a la IP LAN del host.

---

## Dependencias con otras apps y con el backend

- No se edita `app-backend` ni `app-frontend`. Si se necesita un endpoint o cambio de
  configuración, se anota en [[rh-cotla-backend-sub-plan]] y se pide al usuario que coordine.
- El contrato consumido está en `Models/` y `Services/`, y en [[rh-cotla-integracion-clientes-manual]].

## Relacionado

- [[rh-cotla-plan]] — plan vigente (la app móvil es Expo).
- [[rh-cotla-movil-sub-plan]] — plan de la app móvil Expo que reemplazó a MAUI.
