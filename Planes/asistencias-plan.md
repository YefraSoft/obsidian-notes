---
tipo: plan
titulo: "Asistencias Tecnológico Superior de Jalisco — Plan de Desarrollo"
autor: "Efraín García"
creado: 2026-11-17
actualizado: 2026-11-23
estado: en-progreso
proyecto: asistencias
tags:
  - plan
  - asistencias
  - reconocimiento-facial
  - vertical-slices
---

# Asistencias Tecnológico Superior de Jalisco — Plan de Desarrollo

> Registro de asistencia (entrada/salida) del personal de las 17 UA (~16 000 docentes y administrativos), validado con GPS, video en vivo con reto de gestos y verificación facial; entrega incidencias y reportes a RH para nómina. Backend C#/.NET, app web Astro + React (v1), MAUI posible en v2. Se desarrolla por **vertical slices**; las tareas y entregables están en Plane (proyecto "App Asistencias", workspace `direccion-de-sistemas`).

## Pendientes

- [ ] Detallar el Slice 0 con el responsable y crear sus sub-tareas en Plane (cada slice se detalla al comenzar).
- [ ] Elegir el modelo de reconocimiento, el umbral y el formato de evidencia con la PoC del Slice 0.
- [ ] Definir qué se migra del Core en el primer ingreso y cómo se reflejan cambios posteriores (baja, UA, horario, contraseña).
- [ ] Definir el hash de la credencial propia en `new-core` y la política de bloqueo del login.
- [ ] Confirmar el significado de "crea su registro" (nivel 3), si `RH_ENLACE` puede rechazar checadas y la asignación del primer `ADMIN`.
- [ ] Definir cómo llega la `X-Api-Key` rotatoria del CDN, dónde se guardan la `private_route_key` y la `X-User-Key` de subida, y los nombres de colección e identificador.
- [ ] Subir el límite de tasa del CDN para Asistencias.
- [ ] Confirmar si `RH_DIR` administra horarios en la v1 (panel mínimo) o solo se migran del Core.
- [ ] Completar `POSTGRES_CONNECTION` (apunta a `new-core`) en `apps/asistencias` y `apps/asistencias-prod`.
- [ ] Definir SemVer, remoto de git y volumen de modelos en Dokploy.
- [ ] Autorizar la eliminación de las notas obsoletas `asistencias-panel-admin-sub-plan` y `asistencias-app-web-sub-plan` (su contenido ya está en los slices).

## Objetivo

Registrar la asistencia del personal de las **17 UA** con checadas confiables y dar a RH la información (checadas, incidencias, reportes) para calcular nómina, sin capturas manuales. La **v1 debe poder usarse en las 17 UA** (no hay piloto ni fechas definidas).

## Alcance

### Incluido en la v1
- Checada de entrada y salida desde el navegador del celular, solo en línea.
- Validación: GPS dentro de la geocerca de la UA + video/foto en vivo con reto de gestos + coincidencia de embedding con el de referencia.
- Reintento inmediato en rechazos y registro de cada intento rechazado con su motivo.
- Ingreso: la primera vez se valida contra el Core viejo (usuario o CURP + contraseña, solo lectura) y se migra el perfil a `new-core`; los ingresos posteriores son contra `new-core`.
- Roles de 4 niveles (Personal, `RH_ENLACE`, `RH_DIR`, `ADMIN`) guardados en `new-core` y ligados por usuario. Ver [[asistencias-roles-permisos-wiki]].
- Horarios migrados del Core en el registro; docentes con bloques variables y administrativos con horario corrido. `RH_DIR` administra horarios (alcance en la v1 por confirmar).
- Incidencias por bloque: retardo (≥ 15 min), falta (≥ 1 h), falta de entrada o salida, salida anticipada y ausencia del día.
- Panel de administración: UA, geocercas, tolerancias, personal y roles.
- Consulta y verificación de checadas e incidencias por rol y alcance; reportes en Excel (solo datos).
- Notificaciones por email-service: confirmación de checada, checada rechazada/incidencia, aviso de olvido.
- Aviso de privacidad y consentimiento.

### Fuera de la v1 (v2)
- Días inhábiles, permisos, incapacidades y comisiones.
- Panel de administración de horarios propio completo (en la v1 `RH_DIR` administra horarios; el alcance exacto está por confirmar).
- Checada sin conexión (diferida).
- App móvil (posiblemente MAUI).

## Roles

| Nivel | Rol | Puede |
|-------|-----|-------|
| 3 | Docente o Administrativo | `CREATE_OWN`: enviar su checada y crear su "registro"; solo sus datos |
| 2 | `RH_ENLACE` | `SEE_OWN`: ver los datos de su UA |
| 1 | `RH_DIR` | `SEE_ALL` y `EDIT_ALL`: ver todas las checadas, rechazar por inconsistencia y administrar horarios |
| 0 | `ADMIN` | `ALL`: control total del sistema |

Los roles viven en `new-core`, ligados por usuario. Detalle y matriz de permisos: [[asistencias-roles-permisos-wiki]].

## Estado Actual (resumen ejecutivo)

| Área | Estado |
|------|--------|
| Repositorio (`checadores-app`) | Inicializado: `.gitignore`, `CLAUDE.md`, `AGENTS.md`, Dockerfile y CI base |
| OpenBao | KV `apps/asistencias` y `apps/asistencias-prod` creados (falta `POSTGRES_CONNECTION`) |
| Plane | Proyecto `ASISTENCIAS` con un padre `{Slice N}` por slice |
| API / app web | ⏳ Sin código |
| Modelos ONNX | Disponibles en `modelos/` (fuera de git); modelo de producción por decidir en el Slice 0 |

| Slice | Estado |
|-------|--------|
| 0 — Esqueleto, despliegue y prueba de concepto del modelo | 🔄 En progreso (repo, CI base y KV listos) |
| 1 — Ingreso y roles | ⏳ Pendiente |
| 2 — UA y geocerca | ⏳ Pendiente |
| 3 — Enrolamiento (embeddings de referencia) | ⏳ Pendiente |
| 4 — Checada válida | ⏳ Pendiente |
| 5 — Horarios e incidencias | ⏳ Pendiente |
| 6 — Notificaciones | ⏳ Pendiente |
| 7 — Consulta, evidencia y reportes RH | ⏳ Pendiente |
| v2 — Excepciones, horarios completos, diferida y móvil | ⏳ Pendiente |

## Decisiones de Diseño (vigentes)

- **Vertical slices:** cada slice es una entrega completa (BD + API + pantalla + docs + verificación), vive en su rama `feat/slice-N-*` y sale por QA → staging → main con su fase de testing y ajustes. Cada slice se detalla al comenzarlo y entonces se crean sus sub-tareas en Plane.
- **Checada válida** = (1) GPS dentro de la geocerca de la UA, (2) video/foto con la cámara del dispositivo (sin subir archivos) y reto de gestos, (3) el embedding calculado en el cliente coincide con el de referencia, verificado por la API.
- **Procesamiento en el cliente:** el trabajo pesado (detección de rostro, reto de gestos con MediaPipe y cálculo del embedding) corre en el frontend y el backend confía en esos datos. De ahí en adelante el backend verifica comparando el embedding recibido contra el de referencia, la geocerca y las reglas. Riesgo aceptado: un cliente modificado puede enviar datos falsos; la seguridad buscada es que sea más fácil llegar a tiempo que modificar la app.
- **Elección del modelo por PoC (Slice 0, rama `feat/PoC`):** el backend lanza un reto, la app lo ejecuta y se verifica la imagen/embedding en tres escenarios (vector del cliente; cliente + backend con promedio y consistencia; vector del backend) midiendo tiempos y fidelidad; de ahí salen modelo, umbral, formato de evidencia y esquema de verificación. Hallazgo: `fr_yolo*` son detectores de cara, el único modelo de reconocimiento es SFace. Detalle en [[asistencias-slice-0-esqueleto-sub-plan]].
- **Evidencia:** se guarda el video de la persona y el reto en el CDN propio, en la forma más eficiente (clip corto de 1-3 s, ~480p, con fotograma JPEG; GIF solo si el CDN o el visor lo exigen). Las imágenes se consultan solo desde la app, nunca en reportes.
- **Datos e identidad:** el Core viejo (MariaDB, EF Core sin tocar el esquema) **solo se consulta para el registro del usuario** en su primer ingreso; ahí se migra su información a `new-core` (PostgreSQL, migraciones EF), donde viven también sus credenciales propias y sus roles. Todo inicio de sesión posterior es contra `new-core`.
- **Navegador y servicios:** lo ligero va por Astro Actions; el video/foto de evidencia va directo del navegador al CDN (sin API ni Actions). La evidencia se lee a través de un proxy de Astro (servidor) para RH/Admin.
- **Sin RabbitMQ en Asistencias:** avisos y post-proceso con tabla outbox y servicio en segundo plano; incidencias con tarea programada y bloqueo entre réplicas.
- **CDN:** exige `X-Api-Key` rotatoria y `X-User-Key`; el límite de tasa por key se sube desde el CDN para Asistencias. Detalle y riesgo aceptado en [[asistencias-arquitectura-wiki]].
- **Solo en línea en v1:** sin internet no se checa; la app lo avisa.
- **Reparto de herramientas:** Obsidian = plan, decisiones y documentación; Plane = tareas y entregables (los agentes no cambian el estado de las tareas de Plane).
- Los modelos ONNX no se versionan en git ni van en la imagen; se montan como volumen (huellas en `modelos/README.md` del repo).

## Arquitectura y Notas Técnicas

- **Stack:** API C#/.NET con arquitectura Clean de un solo proyecto (`Core`, `Application`, `Infrastructure`, `Api`, `Platform`) como `email-service`; EF Core (MariaDB Core viejo, PostgreSQL `new-core`); frontend Astro + React; ONNX Runtime Web y MediaPipe en el cliente.
- **Integraciones:** Core viejo (MariaDB, solo en el primer ingreso: datos del docente, UA, horarios, URL de foto), CDN privado (referencias y evidencia), email-service (notificaciones). Ver [[asistencias-slice-1-ingreso-roles-sub-plan]] y [[asistencias-slice-6-notificaciones-sub-plan]].
- **Diagramas y flujos:** contexto, componentes, ingreso, checada, rechazo, evidencia, incidencias, modelo de datos y despliegue en [[asistencias-arquitectura-wiki]].
- **Despliegue:** Dokploy; imágenes de cada servicio con su `Dockerfile` publicadas en Harbor; webhook de redespliegue desde el CI.
- **Secretos:** OpenBao en `secretos.tsj.mx`, mount `apps/`: `apps/asistencias` (desarrollo) y `apps/asistencias-prod` (producción); claves `KEY_ENCRYPTION` y `POSTGRES_CONNECTION`. Aquí solo rutas y nombres, nunca valores.
- **CI/CD (Gitea Actions):** push/PR compila API y app web; `QA` publica y despliega testing; `main` publica y despliega producción. Ver [[flujo-ci-cd-manual]] y [[versionado-semver-manual]].
- **Carga esperada:** ~16 000 personas con picos de entrada a primera hora; el backend compara vectores y recibe evidencia, por lo que se dimensiona la subida de clips y la comparación.
- Referencia de implementación: [[email-service-plan]]. Reglas de negocio: [[asistencias-reglas-de-negocio-wiki]].

## Archivos Clave del Proyecto

> **Sub-planes (10):** carpeta `Sub-planes/asistencias-plan/` en la bóveda local.

| Archivo | Propósito |
|---------|-----------|
| `CLAUDE.md` / `AGENTS.md` | Objetivo y reglas de trabajo del repo |
| `.gitignore` | Excluye `modelos/`, `.env*`, binarios |
| `modelos/README.md` | Registro y huellas de los modelos ONNX |
| `api/Dockerfile` | Imagen de la API para Harbor/Dokploy |
| `.gitea/workflows/ci.yml` | CI/CD |

## Entrega por slice

Todo slice recorre el mismo ciclo (ver también `AGENTS.md` del repo):

1. **Detallar** el slice con el responsable (objetivos, contrato, pantallas, pendientes) y crear sus sub-tareas en Plane.
2. **Implementar** en la rama `feat/slice-N-*` desde `QA`, por bloques CREAR → MODIFICAR → ENLAZAR, con entidad y migración, repositorio, servicio, contratos, controller con política, pantalla, documentación y verificación.
3. **PR a `QA`** (lo aprueba QA) → el CI publica y despliega **testing**.
4. **Fase de testing y ajustes:** nota de entrega `Testing/asistencias-slice-N-testing` y ajustes `fix/*` a `QA`.
5. **PR a `staging`** (candidata) → **PR a `main`** (producción); el slice cierra con su versión SemVer.
6. Un slice no empieza hasta cerrar el anterior en `main`, salvo acuerdo.

## SLICE 0 — Esqueleto, despliegue y prueba de concepto del modelo

> **Objetivo:** solución .NET y app Astro mínimas desplegables de punta a punta, y elección del modelo, umbral y formato de evidencia con una PoC de challenge. Detalle: [[asistencias-slice-0-esqueleto-sub-plan]].
> **Estado:** 🔄 En progreso

## SLICE 1 — Ingreso y roles

> **Objetivo:** `new-core`, login, registro desde el Core viejo con migración y roles de 4 niveles. Detalle: [[asistencias-slice-1-ingreso-roles-sub-plan]].
> **Estado:** ⏳ Pendiente

## SLICE 2 — UA y geocerca

> **Objetivo:** catálogo de las 17 UA con geocerca y verificación de ubicación. Detalle: [[asistencias-slice-2-ua-geocerca-sub-plan]].
> **Estado:** ⏳ Pendiente

## SLICE 3 — Enrolamiento (embeddings de referencia)

> **Objetivo:** embedding de referencia desde la foto del CDN, cliente del CDN y consentimiento. Detalle: [[asistencias-slice-3-enrolamiento-sub-plan]].
> **Estado:** ⏳ Pendiente

## SLICE 4 — Checada válida

> **Objetivo:** reto, captura en el cliente, subida directa al CDN, registro, intentos y rechazos. Detalle: [[asistencias-slice-4-checada-valida-sub-plan]].
> **Estado:** ⏳ Pendiente

## SLICE 5 — Horarios e incidencias

> **Objetivo:** horarios por bloque y cálculo de incidencias. Detalle: [[asistencias-slice-5-horarios-incidencias-sub-plan]].
> **Estado:** ⏳ Pendiente

## SLICE 6 — Notificaciones

> **Objetivo:** outbox, servicio en segundo plano y avisos por email-service. Detalle: [[asistencias-slice-6-notificaciones-sub-plan]].
> **Estado:** ⏳ Pendiente

## SLICE 7 — Consulta, evidencia y reportes RH

> **Objetivo:** consultas por rol y alcance, lectura de evidencia por el proxy de Astro y reportes en Excel. Detalle: [[asistencias-slice-7-consulta-reportes-sub-plan]].
> **Estado:** ⏳ Pendiente

## Dependencias y Orden de Ejecución

```
SLICE 0 — Esqueleto y PoC del modelo
    │
    └── SLICE 1 — Ingreso y roles
         │
         ├── SLICE 2 — UA y geocerca ──┐
         │                             │
         └── SLICE 3 — Enrolamiento ───┴── SLICE 4 — Checada válida
                                                │
                                                └── SLICE 5 — Horarios e incidencias
                                                         │
                                                         └── SLICE 6 — Notificaciones
                                                                  │
                                                                  └── SLICE 7 — Consulta y reportes
```

## Relacionado

- [[asistencias-arquitectura-wiki]]
- [[asistencias-roles-permisos-wiki]]
- [[asistencias-reglas-de-negocio-wiki]]
- [[asistencias-slice-0-esqueleto-sub-plan]]
- [[asistencias-slice-1-ingreso-roles-sub-plan]]
- [[asistencias-slice-2-ua-geocerca-sub-plan]]
- [[asistencias-slice-3-enrolamiento-sub-plan]]
- [[asistencias-slice-4-checada-valida-sub-plan]]
- [[asistencias-slice-5-horarios-incidencias-sub-plan]]
- [[asistencias-slice-6-notificaciones-sub-plan]]
- [[asistencias-slice-7-consulta-reportes-sub-plan]]
- [[email-service-plan]]
