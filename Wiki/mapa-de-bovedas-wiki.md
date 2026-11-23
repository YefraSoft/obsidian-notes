---
tipo: wiki
titulo: "Mapa de bóvedas"
autor: "Efraín García"
creado: 2026-09-24
actualizado: 2026-11-18
estado: vigente
tags:
  - wiki
  - boveda
  - mapa
  - rutas
---

# Mapa de bóvedas

> Registro de rutas y esquema de las dos bóvedas: la **local** (repo git en `Documents/Obsidian Vault`, organización por clase) y la **cloud** (Nextcloud, cuenta `efraintics@Tics-Efrain-Docs`, organización por dominio). Mantener actualizado este mapa cada vez que cambie una ruta o estructura.

## Contenido

- [[#1. Bóveda local]]
- [[#2. Bóveda cloud]]
- [[#3. Esquema local vs cloud]]

## 1. Bóveda local

### 1.1 Ruta

| Campo | Valor |
|-------|-------|
| Ruta | `/Users/efraintics/Documents/Obsidian Vault/` |
| Repo | Git (versionada) |
| Organización | Por clase (`tipo`) |

### 1.2 Árbol

```
Obsidian Vault/
├── AGENTS.md
├── Manuales/                    [manual]
│   ├── backend-manual.md
│   ├── cdn-service-manual.md
│   ├── flujo-ci-cd-manual.md
│   ├── news-service-manual.md
│   ├── servicio-cp-manual.md
│   ├── to-production-manual.md
│   └── versionado-semver-manual.md
├── Metodología/                 [metodologia]
│   └── Metodología-Second-Brain.md
├── Planes/                      [plan]
│   ├── asistencias-plan.md
│   ├── email-service-plan.md
│   ├── nummex-app-plan.md
│   ├── nummex-backend-v2-plan.md
│   ├── rh-cotla-plan.md
│   └── Tsj-Web-plan-desarollo.md
├── plantillas/                  [templates]
│   ├── AGENTS.md
│   ├── Template-Manual.md
│   ├── Template-Metodologia.md
│   ├── Template-Plan-de-trabajo.md
│   ├── Template-Proyecto.md
│   ├── Template-Sub-plan.md
│   ├── Template-Testing.md
│   └── Template-Wiki.md
├── Proyectos/                   [proyecto]
│   ├── Proyecto.md
│   └── proyectos.md
├── Sub-planes/                  [sub-plan]  (una subcarpeta por plan padre)
│   ├── asistencias-plan/            (10: slices 0 a 7 + 2 obsoletas)
│   ├── checadores-plan/             (13: slices 00 a 12)
│   ├── email-service-plan/          (10)
│   ├── nummex-app-plan/             (2: agent-rag, cloudflare-deploy)
│   ├── nummex-backend-v2-plan/      (3: fase 1, fase 2, frontend fase A)
│   ├── rh-cotla-plan/               (19: fases 01 a 12, backend, frontend, móvil, pendientes…)
│   └── Tsj-Web-plan-desarollo/      (1: pase-directo-api-aspirantes)
├── Testing/                     [testing]  (documentación de entrega para QA)
│   └── email-service-datos-maestros-testing.md
└── Wiki/                        [wiki]
    ├── asistencias-arquitectura-wiki.md
    ├── asistencias-reglas-de-negocio-wiki.md
    ├── asistencias-roles-permisos-wiki.md
    ├── mapa-de-bovedas-wiki.md
    ├── repositorio-imagenes.md
    └── sso/                     (7 wikis del SSO)
```

### 1.3 Notas por clase

| Clase | Carpeta | Notas |
|-------|---------|-------|
| `manual` | `Manuales/` | 7 |
| `metodologia` | `Metodología/` | 1 |
| `plan` | `Planes/` | 6 |
| `plantillas` | `plantillas/` | 8 (7 templates + AGENTS) |
| `proyecto` | `Proyectos/` | 2 |
| `sub-plan` | `Sub-planes/<plan-padre>/` | 58 |
| `testing` | `Testing/` | 1 |
| `wiki` | `Wiki/` | 5 (+ 7 en `sso/`) |

## 2. Bóveda cloud

### 2.1 Ruta

| Campo | Valor |
|-------|-------|
| Cuenta | `efraintics@Tics-Efrain-Docs` |
| Ruta | `/Users/efraintics/Library/CloudStorage/Nextcloud-cloud.tecmm.mx-punketos` |
| Organización | Por dominio |

### 2.2 Árbol (máx. 3 niveles)

```
Nextcloud-cloud.tecmm.mx-punketos/
├── Readme.md
├── 1. Gobernanza/                  (46 archivos)
│   ├── 1.1 Marco Normativo/
│   │   ├── Lineamientos/           (13 PDFs: Acreditación, Convalidación, Educación a Distancia, Posgrado, Movilidad, Tutorías, Residencia, Servicio Social, Equivalencias, Salida Lateral, Titulación, Traslado)
│   │   └── Reglamentos/
│   ├── 1.2 Identidad Visual/
│   │   ├── fuentes/
│   │   ├── imagenes/               (Hoja membretada.jpg)
│   │   └── logos/
│   └── 1.3 Plantillas/
│       ├── Artefactos/
│       ├── Documentos/             (Acta de Constitución, Acuerdos, Convenios, Cronograma, Informe de Estado, Interesados Clave, Minutas, Nombramientos, Oficios, Plan Anual, Presentaciones, Product BackLog, Requerimientos, Scrum Diario, Solicitud de Desarrollo, Solicitud de Licencias)
│       └── Scripts/                (Gha/)
├── 2. Proyectos/                   (1032 archivos)
│   ├── Admisiones/                 (Minutas/)
│   ├── Almacen/
│   ├── Base de Conocimientos/
│   ├── Bolsa de Trabajo/
│   ├── Convenios/                  (Minutas/)
│   ├── Gestión Académica/          (Planes de Estudio/: IIAL, IIAS, IIND, IINF, IMCT, ISAU, ISIC, ITIC, LADM, LTUR, Maestrías)
│   ├── Gestión Documental/         (insumos/)
│   ├── Migracion Edcore/
│   ├── Pagos/                      (insumos/)
│   ├── Plazas/                     (data/ 0.0.1 y 1.0.0)
│   ├── Servicio de correo/         (plan y sub-planes de email-service)
│   ├── Portal de noticias/         (manual-news-service.md)
│   ├── Portal Institucional/       (Backend/, Frontend/, Minutas/)
│   ├── Portal web/
│   └── Presupuesto/                (insumos/)
├── 3. Operación/                   (6 notas .md + Testing/ + adjuntos/)
│   ├── Acuerdos-21-09.md
│   ├── Efrain-Garcia-Proyectos.md
│   ├── Efrain-Garcia-Wiki.md
│   ├── flujo-ci-cd-manual.md       (manual genérico del equipo: ramas, CI/CD y despliegue)
│   ├── Proyectos Ana Espinoza.md
│   ├── versionado-semver-manual.md (manual genérico del equipo: versión SemVer con make commit-* sobre infra/utils/push.sh)
│   ├── adjuntos/                   (SVG exportados de los diagramas Mermaid de los manuales)
│   └── Testing/                    (notas `testing` por proyecto, para QA)
└── Talk/
```

### 2.3 Conteo por rama

| Rama | Archivos |
|------|----------|
| `1. Gobernanza/` | 46 |
| `2. Proyectos/` | 1032 |
| `3. Operación/` | 12 (6 notas en la raíz, 1 en `Testing/` y 5 SVG en `adjuntos/`) |
| `Readme.md` + raíz | 1 |

## 3. Esquema local vs cloud

| Dimensión | Bóveda local | Bóveda cloud |
|-----------|--------------|--------------|
| Orden | Por clase (`tipo`) | Por dominio |
| Raíces | `Manuales/`, `Planes/`, `Sub-planes/<plan-padre>/`, `Wiki/`, `Proyectos/`, `Metodología/`, `Testing/`, `plantillas/` | `1. Gobernanza/`, `2. Proyectos/`, `3. Operación/`, `Talk/` |
| Control de versiones | Git | Nextcloud |
| Documentos binarios | No | Sí (PDF, DOCX, XLSX, PPTX, JPG, fuentes) |
| Reglas | `AGENTS.md` (raíz) + `plantillas/AGENTS.md` | `Readme.md` |
| Manuales genéricos del equipo (`Manuales/`) | Por clase | `3. Operación/` (los de un proyecto, en `2. Proyectos/<proyecto>/`) |
| Diagramas | Bloques de código `mermaid` en la nota | El mismo bloque más el SVG exportado en `3. Operación/adjuntos/` (Nextcloud Text puede no renderizar Mermaid) |

## Índice / Tabla de contenidos

- [[#1. Bóveda local]]
- [[#2. Bóveda cloud]]
- [[#3. Esquema local vs cloud]]

## Relacionado

- [[AGENTS]]