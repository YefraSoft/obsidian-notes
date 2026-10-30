---
tipo: testing
titulo: "{{title}}"
autor: ""
creado: {{date}}
actualizado: {{date}}
estado: draft            # vigente | en-progreso | pendiente | draft | completado
proyecto: ""
tags:
  - testing
  - qa
---

# {{title}} — Documentación para QA

> <Qué app/servicio cubre esta nota, en qué entorno se prueba y a quién va dirigida (1-2 líneas).>
> Esta nota orienta a QA sobre **qué hace** la app y **por dónde empezar**; no define cómo ejecutar las pruebas.

## Referencias

| Recurso | Enlace |
|---------|--------|
| Plan de desarrollo (bóveda local) | [[<proyecto>-<tema>-plan]] |
| Wiki (bóveda local) | [[<proyecto>-<tema>-wiki]] |
| Plan / wiki en Nextcloud | `2. Proyectos/<Proyecto>/<archivo>.md` |
| Repositorio git | <URL> (si aplica) |
| Wiki del repositorio | <URL> (si aplica) |
| Tareas en Plane | `SITEM-<n>` (si aplica) |

## Objetivo

<Qué problema resuelve la app y qué resultado se espera de ella. Un párrafo.>

## Funcionamiento

<Cómo funciona a grandes rasgos: actores, flujo principal y estados. Diagrama mermaid opcional.>

### Flujo principal

1. <Paso>
2. <Paso>

### Reglas de negocio relevantes

- <Regla que QA debe conocer para entender el comportamiento esperado.>

## Entornos y acceso

| Entorno | URL base | Autenticación | Notas |
|---------|----------|---------------|-------|
| Testing | `https://...` | <API key / Bearer / SSO> | <Cómo obtener acceso; nunca valores de secretos> |

## Endpoints

### Índice

| # | Método | Ruta | Autorización | Respuestas |
|---|--------|------|--------------|------------|
| 1 | `GET` | `/...` | <rol / política> | 200, 401 |

### 1. <Nombre del endpoint>

`<MÉTODO> /ruta`

- **Objetivo:** <qué hace>
- **Autorización:** <rol / política>
- **Reglas:** <validaciones, unicidad, límites>

**Request**

```json
{ }
```

**Response `200`**

```json
{ }
```

**Errores:** `400` <cuándo> · `404` <cuándo> · `409` <cuándo>

## Errores y códigos comunes

| Código | Cuándo |
|--------|--------|
| `400` | <Entrada inválida> |
| `401` | <Sin credenciales o inválidas> |

## Nota de pruebas

> [!note] Estado de pruebas por parte de desarrollo (opcional)
> - **¿Se realizaron pruebas?** <Sí / No>
> - **Alcance:** <unitarias / integración / manuales en testing; qué se cubrió>
> - **Qué no se probó:** <...>
> - **Fecha / versión:** <YYYY-MM-DD / x.y.z>

## Relacionado

- [[<proyecto>-<tema>-plan]]
- [[<proyecto>-<tema>-wiki]]
