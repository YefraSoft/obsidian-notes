---
tipo: wiki
titulo: "Mapa de bóvedas"
autor: "Efraín García"
creado: 2026-09-24
actualizado: 2026-09-24
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
├── <plan-padre>.md              (placeholder huérfano en la raíz)
├── Manuales/                    [manual]
│   ├── backend-manual.md
│   ├── news-service-manual.md
│   ├── servicio-cp-manual.md
│   └── to-production-manual.md
├── Metodología/                 [metodologia]
│   └── Metodología-Second-Brain.md
├── Planes/                      [plan]
│   ├── nummex-app-plan.md
│   ├── rh-cotla-plan.md
│   └── Tsj-Web-plan-desarollo.md
├── plantillas/                  [templates]
│   ├── AGENTS.md
│   ├── Template-Manual.md
│   ├── Template-Metodologia.md
│   ├── Template-Plan-de-trabajo.md
│   ├── Template-Proyecto.md
│   ├── Template-Sub-plan.md
│   └── Template-Wiki.md
├── Proyectos/                   [proyecto]
│   ├── Proyecto.md
│   └── proyectos.md
├── Sub-planes/                  [sub-plan]
│   ├── nummex-agent-rag-plan.md
│   └── nummex-cloudflare-deploy-plan.md
└── Wiki/                        [wiki]
    └── repositorio-imagenes.md
```

### 1.3 Notas por clase

| Clase | Carpeta | Notas |
|-------|---------|-------|
| `manual` | `Manuales/` | 4 |
| `metodologia` | `Metodología/` | 1 |
| `plan` | `Planes/` | 3 |
| `plantillas` | `plantillas/` | 7 (6 templates + AGENTS) |
| `proyecto` | `Proyectos/` | 2 |
| `sub-plan` | `Sub-planes/` | 2 |
| `wiki` | `Wiki/` | 1 (+ esta) |

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
│   ├── Portal de noticias/         (manual-news-service.md)
│   ├── Portal Institucional/       (Backend/, Frontend/, Minutas/)
│   ├── Portal web/
│   └── Presupuesto/                (insumos/)
├── 3. Operación/                   (4 notas .md)
│   ├── Acuerdos-21-09.md
│   ├── Efrain-Garcia-Proyectos.md
│   ├── Efrain-Garcia-Wiki.md
│   └── Proyectos Ana Espinoza.md
└── Talk/
```

### 2.3 Conteo por rama

| Rama | Archivos |
|------|----------|
| `1. Gobernanza/` | 46 |
| `2. Proyectos/` | 1032 |
| `3. Operación/` | 4 |
| `Readme.md` + raíz | 1 |

## 3. Esquema local vs cloud

| Dimensión | Bóveda local | Bóveda cloud |
|-----------|--------------|--------------|
| Orden | Por clase (`tipo`) | Por dominio |
| Raíces | `Manuales/`, `Planes/`, `Sub-planes/`, `Wiki/`, `Proyectos/`, `Metodología/`, `plantillas/` | `1. Gobernanza/`, `2. Proyectos/`, `3. Operación/`, `Talk/` |
| Control de versiones | Git | Nextcloud |
| Documentos binarios | No | Sí (PDF, DOCX, XLSX, PPTX, JPG, fuentes) |
| Reglas | `AGENTS.md` (raíz) + `plantillas/AGENTS.md` | `Readme.md` |

## Índice / Tabla de contenidos

- [[#1. Bóveda local]]
- [[#2. Bóveda cloud]]
- [[#3. Esquema local vs cloud]]

## Relacionado

- [[AGENTS]]