---
tipo: plan
titulo: "Nummex App — Plan de Desarrollo"
autor: "Efraín García"
creado: 2026-09-10
actualizado: 2026-09-11
estado: en-progreso
proyecto: nummex
tags:
  - nummex
  - plan
  - onboarding
  - libro-azul
  - api
  - web-app
---
# Nummex App — Plan de Desarrollo

> Microservicio público de onboarding + `web-app` (ASP.NET Core + PostgreSQL + Astro/React)
> Objetivos: Cotización real con Libro Azul → Onboarding conectado a API → Limpieza y validación final
> Consolidado: 2026-09-10

---

## Pendientes

- [ ] Retirar fallbacks, `IntegrationPendingError` y copy/UI de “integración pendiente” que aún existan.
- [ ] Simplificar `src/lib/loans/api.ts` para trabajar solo contra endpoints reales.
- [ ] Ejecutar validación final del frontend y del flujo local completo.

## Estado Actual (resumen ejecutivo)

| Área | Estado |
|------|--------|
| API de onboarding | Operativa y modular |
| Libro Azul | Cliente real integrado; catálogos y cotización disponibles |
| Documentos | Cloudinary operativo para imágenes y PDF |
| Onboarding React | Conectado a la API real; repositorio fake retirado |
| Amortización | Se calcula en frontend (2.9 % mensual / 1.45 % quincenal) |
| Persistencia | Postgres local definido en `db/prospect-schema.sql` |
| Reanudación | Sesión y caché local confirmada; sin lectura pública segura por `sessionKey` |
| Consultas administrativas | Protegidas con `X-API-Key` |
| Registro público | Rate limiting; sin autenticación de usuario final |

| Fase | Estado |
|------|--------|
| FASE 1 — API modular e integración Libro Azul | ✔ Completa |
| FASE 2 — Onboarding con API real | ✔ Completa |
| FASE 3 — Limpieza y validación final | ⏳ Pendiente |

---

## Decisiones de Diseño (vigentes)

### Integración y contrato público

- Orden de implementación: API primero y después `web-app`.
- Libro Azul usa el cliente real portado desde `erp-nummex/nummex-backend`; no se usan mocks.
- Los endpoints públicos se ajustan al contrato del frontend: `/loan-vehicle/*`, `/loan-quotes` y `/loan-documents/upload`.
- La valuación no se persiste.
- `requestedAmount` y `termMonths` pueden llegar como `null`; el frontend debe soportarlo.
- Los catálogos pueden incluir la clave adicional `rows`, además de la colección esperada por el frontend.

### Persistencia y documentos

- La base de datos es el Postgres local del compose, definido en `db/prospect-schema.sql`; no se usa Neon.
- La carga documental usa Cloudinary, bajo `CLOUDINARY_FOLDER/[prospect_id]:[phone]/...`.
- El residuo `DATABASE_URL` de Neon en `.env` puede retirarse: .NET usa `ConnectionStrings__DefaultConnection`.

### Frontend y sesión

- El CSS se centraliza con tokens; no se usan Tailwind ni SCSS.
- La sesión de onboarding guarda únicamente `prospectId` y `sessionKey` en `localStorage`.
- La reanudación usa datos confirmados en caché local mientras no haya consulta pública segura por `sessionKey`.
- La caché local debe reemplazarse por rehidratación desde backend cuando exista autenticación/JWT de usuario final.

### Seguridad administrativa

- Las consultas administrativas usan `X-API-Key`.
- Los endpoints de registro público están protegidos por rate limiting.

---

## Arquitectura y Notas Técnicas

- **API:** ASP.NET Core modular; `Program.cs` y el registro de módulos se mantienen limpios.
- **Frontend:** Astro + React, con flujo modular en `web-app/src/components/react/components/loan-onboarding/`.
- **Base de datos:** PostgreSQL local a través de Docker Compose.
- **Proveedor de valuación:** Libro Azul, con token de 90 min, bloqueo de reautenticación, reintento en 401, timeout de 10 s y un reintento de red.
- **Caché:** 6 h para catálogos de Libro Azul y 5 min para precio.
- **Política de valuación:** `round(Venta × 0.70, 2, AwayFromZero)`, con tope de 100 M y validaciones.
- **Documentos:** Cloudinary para imágenes y PDF.
- **Proxy:** Caddy sirve el frontend y enruta `/api/*` al backend.

---

## Archivos Clave del Proyecto

| Archivo | Propósito |
|---------|-----------|
| `api/Common/LibroAzul/*` | Cliente, opciones, caché y errores de Libro Azul |
| `api/Common/Modules/*` | Patrón de módulos (`IModule`, `ModuleRegister`) |
| `web-app/src/lib/loans/api.ts` | Contrato del frontend con la API |
| `web-app/src/components/react/components/loan-onboarding/` | Flujo React del onboarding |
| `web-app/src/styles/tokens.css` | Base del sistema visual centralizado |
| `compose.yml` | Servicios locales e integración de contenedores |
| `caddy/Caddyfile` | Proxy de la aplicación |
| `db/prospect-schema.sql` | Esquema local de prospectos |

---

## FASE 1 — API modular e integración Libro Azul

> **Objetivo:** Convertir la API en un microservicio público para onboarding y extracción administrativa.
> **Estado: ✔ Completa** — la integración, endpoints y documentación base fueron aplicados.

### 1.1 Integración Libro Azul

- [x] Crear cliente real bajo `api/Common/LibroAzul/` y configurarlo con `LIBRO_AZUL_BASE_URL`, `LIBRO_AZUL_USER` y `LIBRO_AZUL_PASSWORD`.
- [x] Implementar autenticación con token, TTL de 90 minutos, bloqueo para reautenticación y reintento ante 401.
- [x] Implementar catálogos en cascada y consulta de precio con caché.
- [x] Aplicar timeout de 10 s, un reintento de red y `LibroAzulException`.
- [x] Portar la política de valuación de `LoanValuationPolicy.cs`.

### 1.2 Endpoints públicos

- [x] Exponer `GET /loan-vehicle/years`, `/brands`, `/models` y `/versions`.
- [x] Exponer `POST /loan-quotes` para la valuación de vehículo.
- [x] Exponer `POST /loan-documents/upload` para cargar imágenes y PDF a Cloudinary.
- [x] Conservar las operaciones públicas de prospecto, vehículo, cotización, detalle, referencias y documentos.
- [x] Mantener `GET /Prospects` y `GET /Prospects/{id}` para extracción administrativa con `X-API-Key`.

### 1.3 Limpieza y documentación

- [x] Retirar el scaffolding de `Endpoints/Test/*`.
- [x] Documentar objetivo, alcance, endpoints, configuración y despliegue en `api/README.md`.
- [x] Mantener Postgres local como persistencia; no usar Neon.

### 1.4 Validación aplicada

- [x] Compilar la API sin errores.
- [x] Confirmar que el upload a Cloudinary devuelve URL.
- [x] Confirmar que `GET /Prospects` devuelve información para administración.

---

## FASE 2 — Onboarding conectado a la API real

> **Objetivo:** Sustituir la simulación local por el flujo React conectado a la API, sin romper el diseño actual.
> **Estado: ✔ Completa** — el repositorio fake fue retirado del flujo y el frontend compila correctamente.

### 2.1 Sesión y flujo de onboarding

- [x] Persistir la sesión mínima (`prospectId`, `sessionKey`) y una caché local de respuestas confirmadas.
- [x] Reescribir `useLoanOnboardingFlow` para crear y actualizar prospectos mediante `loanApi`.
- [x] Guardar vehículo y solicitar cotización real de Libro Azul.
- [x] Guardar cotización, detalle, referencias y documentos mediante la API real.
- [x] Mantener la amortización en frontend.

### 2.2 Catálogo, documentos y experiencia

- [x] Conectar el catálogo vehicular en cascada: año → marca → modelo → versión.
- [x] Subir documentos y documentos del vehículo al endpoint real.
- [x] Conservar la etapa `quote`, la tabla de amortización entre `quote` y `personal`, la navegación hacia atrás y el scroll a `section#loan-onboarding-form`.
- [x] Eliminar el repositorio fake del flujo de onboarding.

### 2.3 Validación aplicada

- [x] Ejecutar `bun run check`.
- [x] Ejecutar `bun run build`.
- [x] Confirmar que la cotización y los documentos usan los endpoints reales.

---

## FASE 3 — Limpieza y validación final

> **Objetivo:** Cerrar deuda temporal del frontend y validar el flujo completo contra la API real.
> **Estado: ⏳ Pendiente.**

### 3.1 Limpieza de integración temporal

- [ ] Retirar fallbacks restantes e `IntegrationPendingError`.
- [ ] Eliminar mensajes y UI de “integración pendiente”.
- [ ] Simplificar `src/lib/loans/api.ts` para trabajar solo con endpoints reales.
- [ ] Mantener `tokens.css` como base visual y revisar el CSS del onboarding sin regresiones.

### 3.2 Validación funcional

- [ ] Ejecutar `bun run check` y `bun run build`.
- [ ] Validar el onboarding completo: catálogo, cotización Libro Azul, reanudación y carga a Cloudinary.
- [ ] Ejecutar `docker compose up --build` cuando aplique.
- [ ] Verificar el proxy `/api/*` mediante Caddy y la visibilidad administrativa de `GET /Prospects`.

---

## Dependencias y Orden de Ejecución

```
FASE 1 (API + Libro Azul) — ✔ Completa
    │
    ├── 1.1: Cliente, autenticación, caché y valuación
    ├── 1.2: Catálogos, cotización, documentos y prospectos
    ├── 1.3: Limpieza y documentación
    └── 1.4: Validación de API
         │
         ▼
FASE 2 (Onboarding real) — ✔ Completa
    │
    ├── 2.1: Sesión local y persistencia mediante API
    ├── 2.2: Catálogo, cotización, documentos y UX
    └── 2.3: Validación de frontend
         │
         ▼
FASE 3 (Limpieza final) — ⏳ Pendiente
    │
    ├── 3.1: Retiro de deuda temporal y simplificación
    └── 3.2: Validación local y con Docker Compose
```
