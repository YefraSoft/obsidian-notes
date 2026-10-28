---
tipo: wiki
titulo: "Alcance del SSO MiTSJ"
autor: "Efraín García"
creado: 2026-10-28
actualizado: 2026-10-28
estado: draft
proyecto: sso
tags:
  - sso
  - alcance
---

# Alcance del SSO MiTSJ

## Contenido

> Borrador. Se cierra al terminar las rondas de reglas de negocio.

### 1. Dentro del alcance
- Servicio de autenticación y autorización único del ERP (OAuth 2.0 / OIDC, C# / .NET, PostgreSQL).
- Interfaces: Acceso del SSO y Consola del SSO.
- API de registro, API de permisos, SDK MiTSJ y motor de API keys.
- Conector Hikvision (HikCentral central) y registro de vehículos.
- Bitácora de seguridad y dashboard.

### 2. Fuera del alcance
Ciclo académico y reglas por periodo, bitácora de datos de las apps, envío de correos/mensajes, calendario institucional, almacenamiento de archivos (CDN) y reconocimiento facial (etapa 2). Detalle en [[sso-diseno-base-wiki]] §18.

### 3. Primer consumidor (MVP)
Servicios Escolares. Fases 0-4 en [[sso-diseno-base-wiki]] (se revisarán cuando cierren las reglas).

### 4. Ideas diferidas
- Llaves de autorización excepcional de un solo uso (RN-DIF-01).

## Relacionado

- [[sso-reglas-negocio-wiki]]
- [[sso-requisitos-wiki]]
