---
tipo: sub-plan
titulo: Slice 1 — Identidad, roles y permisos
autor: Efraín García
creado: 2026-11-23
actualizado: 2026-11-23
estado: pendiente
proyecto: checadores
padre: "[[checadores-plan]]"
tags:
  - checadores
  - sub-plan
  - identidad
  - slice-1
---

# Slice 1 — Identidad, roles y permisos

> **Plan padre:** [[checadores-plan]] · **Rama:** `feat/slice-01-identidad` · Plane: CHECADORES-2 (tareas 20–24)
> Quién entra y qué puede hacer; incluye la jerarquía «a su cargo» que usan aprobaciones e informes. La identidad queda detrás de un puerto para pasar a SSO sin tocar roles.

## Pendientes

- [ ] Entrevista previa: matriz exacta de permisos por rol y alcance por recurso.
- [ ] Política de contraseñas mientras no exista SSO (longitud, bloqueo, recuperación).
- [ ] Proveedor y fecha del SSO (solo se aísla aquí).
- [ ] Qué pasa cuando alguien tiene varios roles (administrador de unidad y docente a la vez).

## Estado Actual

| Área | Estado |
|------|--------|
| Roles y jerarquía | ✅ Definidos |
| Matriz de permisos | 🟡 Por definir |
| Implementación | ⏳ |

## Legacy → nuevo

| Legacy | Nuevo |
|--------|-------|
| Usuarios con contraseña en claro, JWT con rutas sin protección | Contraseña con hash, JWT validado en toda la API |
| Usuario `kiosko` compartido con autologin | Cuenta propia por persona |
| Permisos implícitos en `SoloMaestroPuedeVerJustificacionesAjenas` y similares | Roles y políticas explícitas |

## Decisiones

1. Roles: **administrador general**, **administrador de unidad**, **jefe de carrera**, **RH** y **empleado** (autoservicio).
2. Sesión por **cookie** en Astro; la API valida un **JWT**. Cierre por inactividad.
3. **Jerarquía por asignación explícita** en el panel: responsable por departamento o carrera, auditable.
4. El administrador general ve todos los registros y puede revertir decisiones de la unidad.
5. La identidad vive detrás de `IIdentityProvider`; el cambio a SSO no modifica roles ni políticas.
6. 401 uniforme sin credencial o con credencial inválida; 403 para rol sin permiso.

## Alcance

Cubre usuarios, credenciales, sesión, roles, permisos y asignación de responsables. No cubre catálogos de empleados (slice 2) ni aprobaciones (slice 7).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[checadores-plan]]: `IAuthService`, `IUsuariosQueries`, `IUsuarioRepository`, `IAsignacionResponsableRepository`. Ciclo CREAR → MODIFICAR → ENLAZAR.

## Implementación

### Paso 1 — Base de datos
- [ ] Tablas (propuesta) `usuarios`, `roles`, `usuario_roles`, `responsables_unidad`; clave única de usuario.

### Paso 2 — API (CREAR)
- [ ] `Application/Identidad`: `LoginRequest`, `SesionResponse`, `AuthService`, `UsuarioQueries`, `IIdentityProvider`, `Actor` y `Can/Require`.
- [ ] `Infrastructure`: repositorios, hasher de contraseñas (puerto `IPasswordHasher`), emisor de JWT.

### Paso 3 — API (MODIFICAR y ENLAZAR)
- [ ] Política de autorización por rol en `Platform`; controllers `POST /sesiones`, `GET /whoami`, `/usuarios`, `/responsables`.

### Paso 4 — Front
- [ ] Inicio de sesión con cookie (Action), layout protegido por rol, pantallas de usuarios y asignación de responsables, cierre por inactividad.

## Criterios de Aceptación

- [ ] Sin credencial o con credencial inválida: 401 uniforme.
- [ ] Un rol sin permiso recibe 403.
- [ ] Un empleado no ve datos de otro.
- [ ] Cambiar el proveedor de identidad no toca políticas ni roles.

## Relacionado

- [[checadores-plan]]
- [[checadores-slice-02-catalogos-sub-plan]]
- [[checadores-slice-11-autoservicio-sub-plan]]
