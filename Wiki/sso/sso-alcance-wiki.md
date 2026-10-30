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

> **Objetivo:** el SSO es el **proveedor de identidad** (puerta / *API gateway* de identidad) entre los microservicios de MiTSJ: autentica y entrega a cada app la matriz de permisos que solicita. No decide reglas de negocio de las apps.

### 1. Dentro del alcance
- Autenticación (contraseña, Google Workspace, 2FA) y emisión de tokens OAuth 2.0 / OIDC.
- Registro de cuentas (API para apps autorizadas), validación de CURP con RENAPO (proveedor tercero), activación y ciclo de vida.
- Perfiles, roles, grupos, alcance UA/área y matriz de permisos por aplicación.
- Organismos y proveedores (entidad con ADMON y personal).
- Motor de API keys backend a backend (incl. CDN).
- Alta/suspensión de cuentas en Google Workspace.
- Administración por CLI y Consola; bitácora de seguridad.
- SDK MiTSJ para apps.

### 2. Fuera del alcance
- **Microservicio de Accesos (aparte):** torniquetes, Hikvision/HikCentral, vehículos y placas, fotografía de acceso, eventos de paso.
- Ciclo académico y reglas por periodo (apps responsables).
- Relaciones persona↔registro (cada app).
- Almacenamiento de archivos (CDN, app independiente).
- Envío de correos y mensajes (servicio de notificaciones), calendario institucional, bitácora de datos de las apps.
- Reconocimiento facial.

### 3. Fases propuestas (a refinar en los planes)
| Fase | Contenido |
|---|---|
| MVP | RF-01..09, 11..13, 16, 17, 19, 20 (bitácora), 21 — Servicios Escolares opera con el SSO |
| F2 | Organismos, vigencias/suplencias, permisos por campo, Google Workspace automático, dashboard |

### 4. Ideas diferidas
- Llaves de autorización excepcional de un solo uso (RN-DIF-01).
- Separación de funciones / permisos incompatibles (RN-PER-04).

### 5. Dependencias externas abiertas
Proveedor RENAPO, canal de celular (WhatsApp/SMS), tecnología del CDN, dominio(s) de Google Workspace, validación jurídica. Ver [[sso-decisiones-wiki]].

## Relacionado

- [[sso-reglas-negocio-wiki]]
- [[sso-requisitos-wiki]]
- [[sso-diseno-base-wiki]]
