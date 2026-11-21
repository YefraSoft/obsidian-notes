---
tipo: sub-plan
titulo: "Frontend sobre la API v2 — Fase A: funnel, citas y cuenta"
autor: "Efraín García"
creado: 2026-11-01
actualizado: 2026-11-01
estado: en-progreso
proyecto: nummex
padre: "[[nummex-backend-v2-plan]]"
tags:
  - nummex
  - sub-plan
  - frontend
  - backend-v2
  - funnel
  - citas
---

# Frontend sobre la API v2 — Fase A: funnel, citas y cuenta

> **Plan padre:** [[nummex-backend-v2-plan]]
> Migra `nummex-web` (Astro + React) de la API v1 a `nummex-backend-v2` para el funnel del prospecto, las citas y la cuenta. La API v1 queda arriba solo para el chatbot. Relacionado con [[nummex-backend-v2-fase-2-sub-plan]].

## Pendientes

- [ ] **Pruebas de usuario** (las hace el responsable): registro → vehículo → cotización → datos → referencias → documentos → cita; "Recuperar progreso"; `/cuenta` (crear contraseña, iniciar y cerrar sesión). Primer punto a probar: el año del vehículo (`Number(year.name)`) y el formato que devuelve Libro Azul.
- [ ] Completar en el `.env` local `Odoo__Api`, `Jwt__Issuer`, `Jwt__Audience` y `Jwt__Key` (mínimo 32 bytes), y poner `Storage__Endpoint` en `minio:9000` con `Storage__UseSsl=false` antes de `docker compose up` (v2 los exige).
- [ ] Commits: primero en el submódulo `nummex-web` (cliente v2, funnel, citas, cuenta), después el puntero del submódulo y compose/Caddyfile en la raíz.
- [ ] Lecturas del funnel en v2 (`GET` de vehículo, cotización, datos, referencias, documentos y cita): hoy solo se conoce `funnel`. Sin ellas, la cotización no se recupera tras "Recuperar progreso" y los formularios no se precargan. Es parte de la Fase 3 (cuando el prospecto sea cliente) salvo que se adelante.
- [ ] Regla de edad de la fecha de nacimiento: v2 solo exige "anterior a hoy y no más de 120 años"; si se quiere 18+ hay que agregarla en v2 y en el formulario.
- [ ] Contraseña: v2 solo valida longitud (8 a 20); decidir si se agregan reglas de complejidad.
- [ ] Menú principal: agregar "Mi cuenta" requiere decidir el diseño del header; hoy se llega a `/cuenta` desde el final del funnel.
- [ ] Limpieza: textos sin uso en `prestamos.ts` (`warningsTitle`, `integrationStatus.*`), `Dockerfile` del frontend obsoleto (arranca `dist/server/entry.mjs` que no existe con `output: static`), variables de v1 sobrantes en el compose (Cloudinary, Libro Azul y Odoo ya no se usan en v1), `AGENTS.md` de la raíz desactualizado y `apply-migration.sh` (no registra `__EFMigrationsHistory`; solo sirve en BD nueva).
- [ ] Tests del frontend: nada cubre `loanApiV2`, `parseApiError` ni `resumeStageFor`; los escribe el responsable.
- [ ] Sincronizar citas con Odoo y cancelar o confirmar citas (backend, fuera de esta fase).

## Estado Actual

| Área | Estado |
|------|--------|
| Paso 1 — compose y Caddy (`backend-v2`; `/api/chat*` → v1, resto → v2) | ✅ (`docker compose config -q` y `caddy validate`; sin desplegar) |
| Paso 2 — cliente tipado v2 en `src/lib/loans/v2/` | ✅ |
| Paso 3 — funnel sobre v2 (`birthDate`, parentesco, documentos por archivo, recuperar progreso) | ✅ (check, test y build en verde; sin probar en navegador) |
| Paso 4 — pantalla de citas (`/cita`) | ✅ (sin probar en navegador) |
| Paso 5 — cuenta (`/cuenta`: crear contraseña, login, sesión por refresh) | ✅ (sin probar en navegador) |
| Base de datos local | recreada de cero con `initial.sql` (los datos de v1 se descartaron) |

## Alcance

Cubre el cliente HTTP, el funnel completo, la pantalla de citas y la cuenta mínima del frontend, y el enrutado de Caddy. **No cubre** la migración de datos de v1, el chat (sigue en v1 y el agente solo consulta `GET /health` de v1), el área de cliente (Fase 3 de v2) ni lecturas del funnel.

## Decisiones de diseño

- **Mismo dominio, Caddy reparte:** una sola `PUBLIC_API_URL`; la cookie `nummex.session` (Path `/api`, `SameSite=None; Secure`) no cruza dominios.
- **Reanudar:** el funnel arranca desde el inicio; "Recuperar progreso" pide correo y fecha de nacimiento y retoma según `funnel` (1 contacto … 9 usuario). El `vehicleId` se guarda en el navegador; si falta, se repite la etapa de vehículo.
- **Cotización:** primero se cotiza con Libro Azul y después se guarda el vehículo; la cotización se guarda al ver la amortización, solo con monto solicitado y plazo.
- **Token de acceso solo en memoria;** la sesión se restaura con la cookie de refresh.
- **Errores:** se muestra el `detail` de ProblemDetails de v2 (en español), con campos mapeados y manejo de 429, 403 y red.
- **Citas:** rango de fechas calculado en `America/Mexico_City`; una sola cita futura por contacto (reagendar reemplaza).

## Archivos Clave

- `nummex-web/src/lib/loans/v2/` (`client.ts`, `errors.ts`, `types.ts`, `session.ts`).
- `nummex-web/src/components/react/components/loan-onboarding/` (funnel), `AppointmentScheduler.tsx` y `AccountPanel.tsx`.
- `nummex-web/src/pages/cita.astro` y `cuenta.astro`; textos en `src/text/es/`.
- Raíz: `docker-compose.yml` (servicio `backend-v2`) y `Caddyfile`.

## Criterios de Aceptación

- [x] `bun run check`, `bun test` y `bun run build` sin errores tras cada paso.
- [x] `docker compose config -q` y `caddy validate` correctos.
- [ ] El flujo completo funciona en navegador detrás de Caddy, con la cookie de sesión emitida por `POST /api/contacts`.
- [ ] `/api/chat` llega a v1 y el resto de `/api` a v2.

## Relacionado

- [[nummex-backend-v2-plan]]
- [[nummex-backend-v2-fase-2-sub-plan]]
