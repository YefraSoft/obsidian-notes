# Plan de implementación — Nummex CSS System (2026-08-18)

> Esta nota documenta el refactor visual del frontend de NUMMEX.
> Complementa los planes previos de API y onboarding; no los sustituye.

## Estado actual

### Implementado en esta fase

- Se modularizó `src/styles/tokens.css` como archivo de entrada.
- Se crearon capas globales separadas:
  - `colors.css`
  - `resources.css`
  - `typography.css`
  - `spacing.css`
  - `motion.css`
  - `base.css`
  - `components.css`
- Se conservaron aliases `--nmx-*` para compatibilidad de transición.
- Se eliminaron estilos embebidos de componentes y páginas Astro migrados.
- Se crearon CSS externos por componente/página para layout principal.
- Se agregó sistema ligero de reveal con `IntersectionObserver` y soporte para `prefers-reduced-motion`.

## Fase 0 — Assets root, colores, fuentes y base global

### Objetivo

Crear un sistema base de diseño reutilizable y semántico antes de seguir migrando componentes.

### Entregado

- Variables semánticas de color (`primary`, `surface`, `text`, `footer`, `warning`, etc.).
- Recursos globales para assets reutilizables.
- Tipografías centralizadas con `@font-face`.
- Tokens de spacing, radios, container y botón.
- Motion tokens y utilidades base.
- Reset/base global y utilidades compartidas (`.container`, `.btn`, `.nmx-surface-glass`).

## Fase 1 — Migración de componentes Astro

### Objetivo

Sacar el CSS inline de los archivos `.astro` y dejar imports explícitos por componente o página.

### Entregado

#### Componentes

- `Header.astro`
- `Footer.astro`
- `PageHero.astro`
- `Hero.astro`
- `BlackBar.astro`
- `LoanSteps.astro`
- `NextStep.astro`
- `FaqHero.astro`
- `ContactHome.astro`

#### Páginas

- `contacto.astro`
- `prestamos.astro`
- `nosotros.astro`
- `cita.astro`
- `aviso-de-privacidad.astro`

## Fase 2 — Consolidación de CSS React

### Objetivo

Alinear los CSS ya separados de React al nuevo sistema de tokens semánticos.

### Pendiente

- Revisar y normalizar:
  - `ContactForm.css`
  - `LoanOnboardingForm.css`
  - `FaqSection.css`
  - `Accordion.css`
  - `ChatbotBubble.css`
- Reducir hardcodes repetidos de color, radios, sombras y transiciones.
- Evaluar si `LoanOnboardingForm.css` debe fragmentarse por submódulo.

## Fase 3 — Motion system y pulido visual

### Objetivo

Extender la capa de animación elegante a más elementos interactivos sin degradar rendimiento.

### Entregado parcialmente

- Reveal por scroll en bloques principales.
- Hover refinado en botones, navegación, cards y social links.
- Soporte para reduced motion.

### Pendiente

- Revisar microinteracciones en onboarding React.
- Ajustar timings y delays según QA visual.
- Añadir más coherencia en focus states y transiciones internas.

## Validación actual

- `bun run check` limpio.
- Sin errores de Astro ni TypeScript.
- No quedan bloques `<style>` en los Astro migrados.

## Siguientes pasos recomendados

1. Normalizar los CSS React al nuevo sistema semántico.
2. Revisar visualmente responsive en móvil, tablet y desktop.
3. Ejecutar `bun run build` después del QA visual.
4. Documentar convención de uso de tokens para nuevos componentes.
