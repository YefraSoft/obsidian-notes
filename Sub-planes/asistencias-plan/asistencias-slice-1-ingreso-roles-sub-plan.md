---
tipo: sub-plan
titulo: "Asistencias — Slice 1: Ingreso y roles"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-18
estado: pendiente
proyecto: asistencias
padre: "[[asistencias-plan]]"
tags:
  - plan
  - sub-plan
  - asistencias
  - slice
  - core
  - roles
---

# Asistencias — Slice 1: Ingreso y roles

> **Plan padre:** [[asistencias-plan]] · **Rama:** `feat/slice-1-ingreso-roles`
> `new-core` (PostgreSQL) es la fuente de identidad; el Core viejo (MariaDB) solo se consulta en el primer ingreso de cada persona. Incluye los roles de 4 niveles. Diagramas: [[asistencias-arquitectura-wiki]]; roles: [[asistencias-roles-permisos-wiki]].

## Pendientes

- [ ] Detallar el slice con el responsable y crear sus sub-tareas en Plane.
- [ ] Revisar el esquema del Core viejo solo para la lectura del registro: usuario/CURP, contraseña (algoritmo de validación), datos del docente, UA, URL de foto y horarios.
- [ ] Definir qué se migra en el primer ingreso y cómo se reflejan cambios posteriores del Core (baja, UA, horario, contraseña).
- [ ] Definir el hash de la credencial propia, el bloqueo por intentos y el rate-limit del login.
- [ ] Definir la siembra del primer `ADMIN` y el alta de `RH_ENLACE` y `RH_DIR`.
- [ ] Confirmar el significado de "crea su registro" (nivel 3), si `RH_ENLACE` puede rechazar checadas y si el nivel 3 ve su historial.

## Estado Actual

| Área | Estado |
|------|--------|
| Flujo de ingreso y roles | ✅ Definidos (ver wikis) |
| Esquema del Core viejo | ⏳ Por revisar |
| `new-core` | ⏳ Por crear |

## Alcance (vertical)

- **BD (`new-core`):** usuario, credencial, rol y asignación por usuario con alcance y vigencia, UA (referencia), bitácora de cambios de rol. Migraciones EF.
- **Core viejo:** `CoreDbContext` de **solo lectura**, sin tocar el esquema, usado solo por el servicio de registro.
- **API:** sesión (login), registro desde el Core viejo con migración, política por rol y filtro de alcance obligatorio, administración de roles (solo `ADMIN`).
- **Pantalla:** login (usuario o CURP + contraseña), aviso de privacidad y consentimiento, y panel de administración de usuarios y roles.
- **Docs:** actualizar [[asistencias-roles-permisos-wiki]] y la nota de testing del slice.

## Implementación

### Bloque CREAR
- [ ] Entidades y migraciones de identidad y roles; repositorios por intención.
- [ ] `CoreDbContext` de solo lectura y servicio de registro/migración.
- [ ] Servicio de sesión y verificación de roles/alcance.

### Bloque MODIFICAR
- [ ] Ajustes sobre el esqueleto del slice 0 (errores, configuración, políticas).

### Bloque ENLAZAR
- [ ] Controllers y Astro Actions de login; pantalla de login y de administración de roles; verificación con curl y pruebas manuales.

## Entrega

1. `feat/slice-1-ingreso-roles` desde `QA` → PR a `QA` → CI despliega **testing**.
2. Nota de entrega: `Testing/asistencias-slice-1-testing`; ajustes `fix/*` a `QA`.
3. PR a `staging` → PR a `main`.

## Criterios de Aceptación

- [ ] Ninguna migración ni escritura se ejecuta contra el Core viejo.
- [ ] Tras el primer ingreso, el usuario inicia sesión sin consultar el Core viejo.
- [ ] Todo usuario migrado tiene su rol de nivel 3 y su credencial propia en `new-core`.
- [ ] Solo `ADMIN` asigna roles; cada consulta aplica el alcance en la API.

## Relacionado

- [[asistencias-plan]]
- [[asistencias-arquitectura-wiki]]
- [[asistencias-roles-permisos-wiki]]
- [[asistencias-slice-0-esqueleto-sub-plan]]
- [[asistencias-slice-2-ua-geocerca-sub-plan]]
