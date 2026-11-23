---
tipo: sub-plan
titulo: Slice 11 — Autoservicio / kiosko
autor: Efraín García
creado: 2026-11-23
actualizado: 2026-11-23
estado: pendiente
proyecto: checadores
padre: "[[checadores-plan]]"
tags:
  - checadores
  - sub-plan
  - kiosko
  - autoservicio
  - slice-11
---

# Slice 11 — Autoservicio / kiosko

> **Plan padre:** [[checadores-plan]] · **Rama:** `feat/slice-11-autoservicio` · Plane: CHECADORES-12 (tareas 71–73)
> Vista para que cada persona consulte lo suyo y capture solicitudes, incluso desde una pantalla comunal.

## Pendientes

- [ ] Entrevista previa: diseño de pantallas y tiempo de inactividad.
- [ ] Inicio de sesión mientras no exista SSO (usuario y contraseña propios).
- [ ] Qué ve exactamente cada tipo de personal (saldos, horarios).

## Estado Actual

| Área | Estado |
|------|--------|
| Requisitos | ✅ Definidos |
| Implementación | ⏳ |

## Legacy → nuevo

| Legacy | Nuevo |
|--------|-------|
| Usuario `kiosko` compartido con autologin (`/mdaess`, `/mdaessk`) | Cuenta propia por persona (SSO a futuro) |
| Formularios dinámicos con temporizadores `timeoutModal`, `timeoutContainer`, `timeoutExitKioskoMode` | Pantallas fijas con cierre por inactividad |
| Salida del kiosko con credenciales (`/forms/exitKioskMode`) | Cierre de sesión normal |

## Decisiones

1. El kiosko se **conserva** como vista de autoservicio: **mis checadas** (día o periodo), **mis incidencias** y **capturar justificante o permiso** (reutiliza el slice 7).
2. Cada persona inicia sesión con **su propia cuenta**; se **elimina el usuario compartido `kiosko`**.
3. Sesión corta con cierre por inactividad; es un perfil de «empleado», no un rol especial.
4. Los formularios dinámicos desaparecen: son pantallas fijas.

## Alcance

Cubre consultas del propio usuario, captura de solicitudes y cierre por inactividad. No cubre administración de otros (slices 1 y 7).

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[checadores-plan]]. Ciclo CREAR → MODIFICAR → ENLAZAR.

## Implementación

### Paso 1 — CREAR
- [ ] `IMisDatosQueries` (checadas e incidencias del usuario autenticado); layout de autoservicio en Astro.

### Paso 2 — MODIFICAR
- [ ] Política que limita cada consulta al `Actor`; ocultar identificadores manipulables.

### Paso 3 — ENLAZAR
- [ ] `GET /mis/checadas`, `GET /mis/incidencias`, `GET /mis/saldos`; pantallas, captura de solicitud (slice 7) y temporizador de inactividad.

## Criterios de Aceptación

- [ ] Una persona no puede ver datos de otra cambiando un identificador.
- [ ] La sesión se cierra tras el tiempo de inactividad configurado.
- [ ] Se puede capturar un justificante o permiso desde la vista.

## Relacionado

- [[checadores-plan]]
- [[checadores-slice-01-identidad-sub-plan]]
- [[checadores-slice-07-permisos-sub-plan]]
