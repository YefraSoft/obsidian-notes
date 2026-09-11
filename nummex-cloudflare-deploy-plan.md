---
tags:
  - nummex
  - plan
  - infraestructura
  - cloudflare
  - caddy
  - ssh
created: 2026-09-10
status: pendiente
proyecto: nummex
---
# Nummex — Despliegue de Pruebas con Cloudflare

> Frontend Astro + API ASP.NET Core + PostgreSQL + Qdrant + agente interno, publicados desde este equipo mediante Cloudflare Tunnel y Caddy.
> Objetivo: dejar `dev.yefrasoft.com` disponible para pruebas y habilitar administración remota por SSH sin abrir puertos en el router.

---

## Estado actual

| Área | Estado |
|---|---|
| Frontend | Astro en `nummex-web`; debe construirse como sitio estático |
| API | ASP.NET Core en `nummex-backend`, escucha internamente en `8080` |
| Datos | PostgreSQL y Qdrant deben permanecer internos |
| Agente | Servicio interno en `4000`; no debe exponerse al host ni a Internet |
| Proxy | Existe un `Caddyfile`, hoy solo sirve estáticos |
| Compose | Requiere corregir la ruta `web-app-nummex` por `nummex-web` |
| Cloudflare | No hay `cloudflared` instalado ni túnel configurado aún |

## Decisiones de diseño vigentes

- La aplicación de pruebas será pública en `https://dev.yefrasoft.com`.
- El frontend y la API compartirán origen: la API vivirá en `https://dev.yefrasoft.com/api/*`.
- SSH se publicará exclusivamente en `ssh.dev.yefrasoft.com` mediante Cloudflare Tunnel y Cloudflare Access.
- Cloudflare Access usará One-time PIN enviado al correo administrador de Cloudflare.
- Caddy será el único servicio HTTP que escucha en el host, limitado a `127.0.0.1:8088`.
- PostgreSQL, backend, Qdrant y agente no publicarán puertos al host ni a Internet.
- Los secretos locales, credenciales de servicios y token/credenciales del túnel no se versionan.

## Topología objetivo

```text
Internet
  ├─ https://dev.yefrasoft.com
  │    Cloudflare Tunnel → 127.0.0.1:8088 → Caddy
  │                                              ├─ /api/* → backend:8080
  │                                              └─ /*     → frontend estático
  └─ ssh://ssh.dev.yefrasoft.com
       Cloudflare Access + Tunnel → sshd local :22

Red Docker interna
  backend ↔ PostgreSQL
  agent   ↔ Qdrant / backend / Ollama del host
```

---

## FASE 1 — Preparación del stack Docker

> **Objetivo:** dejar los servicios correctamente conectados y sin exposición accidental.

### 1.1 Compose y variables públicas

- [ ] Corregir el contexto de build de `frontend-build`: `./web-app-nummex` → `./nummex-web`.
- [ ] Construir el frontend con `PUBLIC_SITE_URL=https://dev.yefrasoft.com`.
- [ ] Construir el frontend con `PUBLIC_API_URL=/api`.
- [ ] Configurar `CORS_ALLOWED_ORIGINS=https://dev.yefrasoft.com` para el backend.
- [ ] Usar configuración de ejecución no desarrolladora para el backend publicado.

### 1.2 Aislamiento y disponibilidad

- [ ] Retirar el mapeo público de `5432:5432` de PostgreSQL.
- [ ] Retirar el mapeo público de `8080:8080` del backend.
- [ ] Confirmar que Qdrant y el agente siguen sin mapeos de puertos al host.
- [ ] Configurar políticas de reinicio para los servicios que deben recuperarse al reiniciar Docker o el equipo.
- [ ] Mantener healthchecks para PostgreSQL, backend, agente y frontend cuando aplique.

### Criterios de aceptación

- [ ] El frontend construye desde `nummex-web`.
- [ ] La API responde desde la red interna Docker.
- [ ] `5432`, `8080`, `4000` y Qdrant no quedan publicados en interfaces del equipo.

---

## FASE 2 — Caddy como borde HTTP local

> **Objetivo:** servir la aplicación completa detrás de una sola entrada HTTP local.

### 2.1 Rutas de Caddy

- [ ] Configurar el sitio para el host `dev.yefrasoft.com`.
- [ ] Enrutar `/api/*` hacia `backend:8080` sin quitar el prefijo `/api`.
- [ ] Servir el contenido estático del frontend para el resto de rutas.
- [ ] Conservar fallback SPA hacia `/index.html`.
- [ ] Activar compresión `zstd` y `gzip`.

### 2.2 Enlace local

- [ ] Publicar el contenedor Caddy solo como `127.0.0.1:8088:80`.
- [ ] No solicitar certificados públicos desde Caddy; Cloudflare termina HTTPS antes del túnel.
- [ ] Mantener backend y frontend en la red Docker para que Caddy resuelva el servicio `backend` por nombre.

### Criterios de aceptación

- [ ] `curl http://127.0.0.1:8088` devuelve el frontend.
- [ ] `curl http://127.0.0.1:8088/api/loan-vehicle/years` alcanza el backend.
- [ ] Una ruta del frontend sin archivo físico carga la SPA.

---

## FASE 3 — Cloudflare Tunnel y registro DNS

> **Objetivo:** crear la entrada pública sin abrir puertos del router ni del firewall hacia Internet.

### 3.1 Servicio local del túnel

- [ ] Instalar `cloudflared` en Fedora.
- [ ] Autenticar la máquina contra la zona `yefrasoft.com` en Cloudflare.
- [ ] Crear un túnel nombrado para este equipo.
- [ ] Guardar sus credenciales fuera del repositorio.
- [ ] Instalar y habilitar el servicio `cloudflared` con `systemd` para que inicie después de reinicios.

### 3.2 Registro de subdominios en Cloudflare

> [!warning] No crear registros A/AAAA hacia la IP residencial o pública del equipo. Los hostnames del túnel deben crear o usar CNAME administrados por Cloudflare hacia el identificador del túnel.

- [ ] En **Cloudflare Zero Trust → Networks → Tunnels**, abrir el túnel creado.
- [ ] Agregar Public Hostname `dev.yefrasoft.com` con servicio `http://127.0.0.1:8088`.
- [ ] Confirmar que Cloudflare creó el CNAME `dev` hacia `<UUID-del-tunel>.cfargotunnel.com` dentro de la zona `yefrasoft.com`.
- [ ] Agregar Public Hostname `ssh.dev.yefrasoft.com` con servicio `ssh://127.0.0.1:22`.
- [ ] Confirmar que Cloudflare creó el CNAME `ssh.dev` hacia `<UUID-del-tunel>.cfargotunnel.com`.
- [ ] Verificar que ambos registros estén con proxy de Cloudflare activo y que no existan registros DNS conflictivos.
- [ ] Definir regla final del túnel: `http_status:404` para cualquier hostname no reconocido.

### Criterios de aceptación

- [ ] El túnel figura como **Healthy** en Cloudflare.
- [ ] `https://dev.yefrasoft.com` carga desde una red externa.
- [ ] No se requiere redirección de puertos en el router.

---

## FASE 4 — Cloudflare Access y SSH

> **Objetivo:** permitir administración remota con doble frontera: identidad Cloudflare y autenticación SSH por llave.

### 4.1 Servidor SSH local

- [ ] Confirmar que `sshd` está instalado, activo y escucha en `127.0.0.1:22` o la interfaz local requerida por el túnel.
- [ ] Verificar acceso del usuario `yefrasoft` mediante llave pública.
- [ ] Desactivar `PermitRootLogin`.
- [ ] Desactivar `PasswordAuthentication` después de confirmar una llave funcional.
- [ ] Mantener `PubkeyAuthentication yes`.

### 4.2 Aplicación Access

- [ ] En **Cloudflare Zero Trust → Access → Applications**, crear una aplicación de tipo **Self-hosted**.
- [ ] Asociar el hostname `ssh.dev.yefrasoft.com`.
- [ ] Activar el proveedor de identidad **One-time PIN**.
- [ ] Crear una política **Allow** para el correo administrador de Cloudflare.
- [ ] Verificar que no exista una política de bypass involuntaria.

### 4.3 Cliente SSH

- [ ] Instalar `cloudflared` en cada equipo cliente autorizado.
- [ ] Añadir al archivo `~/.ssh/config` del cliente:

```sshconfig
Host nummex-dev
  HostName ssh.dev.yefrasoft.com
  User yefrasoft
  ProxyCommand cloudflared access ssh --hostname %h
  IdentityFile ~/.ssh/id_ed25519
```

- [ ] Conectar usando `ssh nummex-dev`; completar el código de Cloudflare Access y validar la llave SSH.

### Criterios de aceptación

- [ ] Una conexión SSH abre la autenticación de Cloudflare Access antes de llegar a `sshd`.
- [ ] Un usuario no autorizado por Cloudflare no puede iniciar el túnel SSH.
- [ ] Una llave SSH no autorizada no puede iniciar sesión, aunque la identidad Access sea válida.

---

## FASE 5 — Validación operativa y recuperación

> **Objetivo:** confirmar la publicación externa y la persistencia del stack.

### 5.1 Validación funcional

- [ ] Construir y levantar el stack con Docker Compose.
- [ ] Comprobar el frontend y el endpoint de catálogos a través de Caddy local.
- [ ] Comprobar `https://dev.yefrasoft.com` y una ruta SPA desde una red externa.
- [ ] Comprobar el flujo de API desde el frontend bajo el mismo origen.
- [ ] Confirmar que el agente, Qdrant y PostgreSQL solo son visibles en la red Docker.

### 5.2 Recuperación y monitoreo mínimo

- [ ] Reiniciar Docker y confirmar recuperación automática de los contenedores necesarios.
- [ ] Reiniciar el equipo y confirmar que Docker y `cloudflared` recuperan la publicación.
- [ ] Revisar logs de Caddy, backend y `cloudflared` ante fallas de acceso.
- [ ] Documentar cualquier IP, puerto local o credencial fuera de esta nota, en almacenamiento seguro.

## Comandos de referencia

```bash
# Stack de la aplicación
docker compose up -d --build
docker compose ps
docker compose logs -f frontend backend

# Túnel y SSH (en el host)
sudo systemctl status cloudflared
sudo systemctl status sshd

# Validación local
curl -I http://127.0.0.1:8088
curl http://127.0.0.1:8088/api/loan-vehicle/years

# Cliente remoto
ssh nummex-dev
```

## Riesgos y notas de seguridad

- El backend contiene integraciones y secretos sensibles; los archivos `.env` locales no deben copiarse a esta bóveda ni al repositorio.
- Publicar la web de pruebas implica que sus endpoints públicos serán accesibles desde Internet; el rate limiting debe mantenerse activo.
- Cloudflare Access protege SSH, pero no reemplaza las llaves SSH ni el endurecimiento de `sshd`.
- No se deben abrir puertos 22, 80, 443, 5432 ni 8080 en el router para esta arquitectura.
