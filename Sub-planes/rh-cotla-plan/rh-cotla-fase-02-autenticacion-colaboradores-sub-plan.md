---
tipo: sub-plan
titulo: "Fase 2 — Autenticación, cuentas y colaboradores"
autor: Efraín García
creado: 2026-10-22
actualizado: 2026-11-20
estado: en-progreso
proyecto: rh-cotla
padre: "[[rh-cotla-plan]]"
tags:
  - rh-cotla
  - sub-plan
  - fase-2
  - autenticacion
  - colaboradores
---

# Fase 2 — Autenticación, cuentas y colaboradores

> **Plan padre:** [[rh-cotla-plan]]
> Login con JWT, RBAC con alcance por rol, expediente del colaborador, auto-registro y gestión de usuarios y sesiones. Backend y panel completos; la app móvil queda parcial (sin edición de datos propios).

## Pendientes

- [x] Backend: auth, RBAC, colaboradores, usuarios y roles, sesiones.
- [x] Panel web: login, recuperación, plantilla, cuenta y usuarios.
- [x] Móvil: login, recuperación, cambio de contraseña, auto-registro y «Mi perfil» de solo lectura.
- [ ] Móvil: edición de datos personales. Depende de decisión del cliente y de un endpoint del backend (hoy solo existe `POST /auth/change-password`).
- [ ] Revocación real de sesiones: hoy `revocada_at` es informativa y el JWT vive hasta expirar (**FX-BE-2**, prioridad alta, 1.0.1).
- [ ] `POST /users/{id}/password-reset` responde 204 sin enviar correo si no hay SMTP; decidir si pasa a 503 (**FX-BE-5**).
- [ ] Enlace de recuperación abre el panel web, no la app (**FX-BE-10**; decisión del 20-nov-2026: backend sin cambios).
- [ ] Contacto de emergencia en el expediente: la entidad existe sin endpoint ni campo (**F-BE-3**).
- [ ] Búsqueda de empleados con caché y tolerancia a mayúsculas (**F-BE-2**).
- [ ] Token en `localStorage` con «mantener sesión» (**FX-FE-2**) y protección de rutas solo en cliente (**FX-FE-5**).
- [ ] Móvil, bloqueante de tiendas: no hay flujo de eliminación de cuenta pese al auto-registro (**FX-MOV-1**).
- [ ] Acceso biométrico en móvil (**F-MOV-1**), retirado de la UI por no ser funcional.
- [ ] Rate limit general y proteger o retirar `/diagnostics/*` (**FX-BE-3**).

## Estado Actual

| Área | Estado |
|------|--------|
| Backend | ✅ Completo y validado con pruebas automatizadas y contra `docker compose`; pendientes de v2 listados arriba |
| Panel web | ✅ Login, recuperación, plantilla (alta, edición, activar, rechazar, desactivar), Mi cuenta y Usuarios |
| App móvil | 🟡 Login, recuperación, auto-registro, cambio de contraseña, «Mi perfil» solo lectura; falta edición de datos |
| Entregables clave | JWT 60 min, BCrypt, `AlcanceService`, `/users` con guardas de último RH, 10 intentos/min por IP |

## Alcance

Cubre cuentas, autenticación, RBAC, expediente (§2 y §3 de `docs/especificaciones.md`), auto-registro, sesiones y usuarios/roles. No cubre los catálogos que el expediente referencia ([[rh-cotla-fase-03-configuracion-tienda-sub-plan]]), ni la auditoría consultable ([[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]]). Quedan fuera por decisión: cambio de correo o teléfono por el propio usuario, expediente con documentos e historial laboral, edición del contacto de emergencia.

## Decisiones

1. **Identificador de acceso = número de empleado** (máx. 50); `POST /auth/login` no acepta correo. El número lo genera el cliente y se valida con `GET /employees/validate-number`.
2. **Contraseña** mínimo 8 y máximo 200 caracteres, hash BCrypt; nunca se almacena en claro.
3. **JWT stateless**, vigencia 60 min (`JwtSettings:ExpirationMinutes`); claims de `sub`, `jti`, rol, `numero_empleado`, `empleado_id` y `sucursal_id`. La revocación es informativa: la mitigación es la ventana corta.
4. **Autorización en dos capas:** la policy decide quién llama (`autenticado`, `rh-only`, `rh-or-gerente` en `Common/Config/Security/Models/Policies.cs`); `AlcanceService` decide sobre qué filas. RH ve todo, gerente su equipo vía `equipos_gerentes` (solo lectura), colaborador solo lo suyo. Fuera de alcance responde 403.
5. **Estados de cuenta** `pendiente`, `activo`, `inactivo`, `rechazado`; solo `activo` entra (otro estado: 403).
6. **Límite de intentos:** 10 peticiones por minuto por IP en login, self-register, forgot-password y reset-password; 429 sin `Retry-After`.
7. **Recuperación:** `forgot-password` siempre 202; token de un solo uso, 30 min, hash SHA-256; una nueva solicitud invalida la anterior.
8. **Guardas de RH** (`Common/Security/RhGuard.cs`): la sucursal conserva al menos un RH activo y un RH no se degrada, desactiva ni rechaza a sí mismo (409).
9. **`null` en `PATCH /employees` no desasigna** (D5); un catálogo inexistente da 404 y uno dado de baja 400.
10. **Errores en español** en validación, framework y dominio (`Common/Errors`).
11. **El rol se administra solo en `/usuarios`:** se retiró el selector de rol de la ficha del colaborador en el panel.

## Implementación

### Backend

- [x] `POST /auth/login`, `POST /auth/logout`, `POST /auth/change-password`, `POST /auth/forgot-password`, `POST /auth/reset-password`, `GET /auth/me`, `GET /auth/sessions`.
- [x] Colaboradores: `GET/POST /employees`, `GET/PATCH /employees/{id}`, `activate`, `deactivate` (con motivo), `reject`, `self-register`, `validate-number`.
- [x] Usuarios: `GET /users`, `GET /users/{id}`, `PUT /users/{id}/role`, `activate`, `reject`, `deactivate`, `sessions/revoke`, `password-reset`.
- [x] Migración `Fase2AutenticacionYColaboradores` (columna `empleados.correo`, `restablecimientos_password`, estado `rechazado`).
- [x] SMTP de desarrollo y `PASSWORD_RESET_BASE_URL` por `docker-compose.yml`.

### Endpoints

| Ruta | Rol |
|------|-----|
| `POST /api/v1/auth/login` | público |
| `POST /api/v1/auth/forgot-password`, `reset-password` | público |
| `POST /api/v1/employees/self-register` | público (rate limit) |
| `GET /api/v1/auth/me`, `GET /auth/sessions`, `POST /auth/logout`, `POST /auth/change-password` | autenticado |
| `GET /api/v1/employees`, `GET /employees/{id}` | autenticado, con alcance |
| `POST /auth/sessions/revoke-all?usuarioId=` | `rh` |
| `GET /employees/validate-number`, `POST /employees`, `PATCH /employees/{id}`, `activate`, `reject`, `deactivate` | `rh` |
| `/api/v1/users` (todas las rutas) | `rh` |

### Panel web (`app-frontend/`)

- [x] Login — `/`; recuperación — `/olvido-contrasena` y `/reset-password`.
- [x] Plantilla (alta, edición, expediente, activar, rechazar, desactivar) — `/plantilla`.
- [x] Mi cuenta y sesiones — `/cuenta`.
- [x] Usuarios y roles — `/usuarios` (listado con filtros, detalle con sesiones, cambio de rol, reset, revocar); solo RH.
- [x] Los 409 de «último RH» y autodegradación se muestran como error de la acción.

### App móvil (`app-movil/`)

- [x] Login — `(auth)/login.tsx`; recuperación — `(auth)/forgot-password.tsx` y `(auth)/reset-password.tsx` (por `cotla://reset-password`).
- [x] Cambio de contraseña — `change-password.tsx`; auto-registro — `(auth)/self-register.tsx`.
- [x] Sesión persistente con `expo-secure-store`, revalidada con `GET /auth/me`; cierre de sesión desde Inicio.
- [x] «Mi perfil» — `profile.tsx` (solo lectura, `GET /employees/{empleadoId}`).
- [ ] Edición de datos personales (depende de backend y decisión del cliente).
- [ ] Entrada por deep link sin verificar en build de producción.

## Criterios de Aceptación

- [x] Auto-registro → `pendiente` → RH activa → el colaborador opera (`EmployeesEndpointsTests.Colaborador_pendiente_no_puede_iniciar_sesion_hasta_ser_activado`).
- [x] Login de cuenta `pendiente`, `rechazado` o `inactivo` devuelve 403.
- [x] El colaborador no accede a datos ajenos (403); el gerente ve solo su equipo.
- [x] Cambio y recuperación de contraseña completos; token de un solo uso y guardado solo como hash.
- [x] Contraseñas solo como hash BCrypt.
- [x] Guardas de último RH y autodegradación (409).
- [ ] Recorrido en DevTools de todas las pantallas del panel con el backend real (**FX-FE-11**).
- [ ] Revocación de sesiones efectiva sobre el token ya emitido (**FX-BE-2**).

## Pendientes para otras fases

- [ ] Aviso de privacidad en el auto-registro móvil (**FX-MOV-5**): [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]].
- [ ] CSP del panel (**FX-FE-1**) y cookie `httpOnly`: [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]].
- [ ] Eliminación de cuenta y tiendas (**FX-MOV-1**): [[rh-cotla-fase-12-despliegue-sub-plan]].

## Relacionado

- [[rh-cotla-plan]] — plan padre con el orden de las fases y las decisiones de stack.
- [[rh-cotla-progreso-backend-sub-plan]] — contratos de `/auth`, `/employees` y `/users`, pruebas y limitación de la revocación (§4, §5, §9, §19).
- [[rh-cotla-integracion-clientes-manual]] — contrato de API para web y móvil: login, recuperación, colaboradores y sesiones.
- [[rh-cotla-pendientes-sub-plan]] — deuda con ids F-/FX- citados arriba.
- [[rh-cotla-fase-01-fundaciones-sub-plan]] — fase anterior: modelo de datos y seeds de roles y RH.
- [[rh-cotla-fase-03-configuracion-tienda-sub-plan]] — fase siguiente: catálogos que el expediente referencia.
- [[rh-cotla-fase-11-archivos-auditoria-seguridad-sub-plan]] — auditoría y endurecimiento de seguridad.
- [[rh-cotla-fase-12-despliegue-sub-plan]] — publicación del panel y la app.
- Repo: `docs/especificaciones.md` (§2, §3), `app-backend/`, `app-frontend/`, `app-movil/`.
