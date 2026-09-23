---
tags:
  - devops
  - deploy
  - docker
  - ssh
  - shell
  - versionado
  - manual
  - guia
created: 2026-09-18
status: vigente
---

# Manual de despliegue — `to-production.sh`

> Manual para que el equipo publique sus apps en los servidores del proyecto. Define **dos herramientas** según el tipo de proyecto, un **sistema de versionado común** y un modelo **multiserver** reutilizable. Relacionado con [[Proyecto]] y [[Tsj-Web-plan-desarollo]].

- **Lenguaje:** Bash (`#!/usr/bin/env bash`, con `set -euo pipefail`).
- **Comunicación con el servidor:** SSH + SCP sin contraseña (llave pública `ed25519`).
- **Versionado:** archivo de texto `docs/versions/version-history.txt` (semver `X.Y.Z`, flags `-M` / `-U` / `-F`).
- **Multi-servidor:** arreglo `SERVER_PROFILES` en formato `"alias|usuario@host|/ruta/remota"`.
- **Dónde vive el script:** copiado a la **raíz del repo** de cada app (`chmod +x`).

| Tipo de app | Herramienta | Qué entrega al servidor |
|-------------|-------------|-------------------------|
| Backend / app con `Dockerfile` | `to-production.sh` | Imagen Docker (`<imagen>:<version>` y `<imagen>:latest`) |
| Sitio estático (build → `dist`) | `to_production_static.sh` | Carpeta `latest` servida por el web server |

---

## 1. Configuración previa (una sola vez)

### 1.1 Copiar tu llave SSH al servidor (para que no pida contraseña)

```bash
# Generar llave si no tienes una
ssh-keygen -t ed25519 -C "tu-correo@example.com"

# Copiarla a cada servidor
ssh-copy-id root@tecmm.mx
ssh-copy-id root@vinculacion.tsj.mx
```

> [!warning] macOS / Windows
> Si `ssh-copy-id` no existe (macOS antiguo o Git Bash que no lo trae):
> ```bash
> brew install ssh-copy-id
> ```
> O manualmente, pegando tu llave pública en `authorized_keys`:
> ```bash
> cat ~/.ssh/id_ed25519.pub | ssh root@tecmm.mx "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
> ```

Verifica que la conexión ya no pida contraseña:

```bash
ssh root@tecmm.mx "hostname"
ssh root@vinculacion.tsj.mx "hostname"
```

> [!note] Permisos del usuario remoto
> - **`to-production.sh`**: el usuario debe poder ejecutar `docker` (si no es `root`: `sudo usermod -aG docker <usuario>` y volver a iniciar sesión).
> - **`to_production_static.sh`**: el usuario debe poder escribir en la ruta web servida.

### 1.2 Colocar y activar los scripts

Copia la(s) herramienta(s) a la **raíz de tu repo** y dale permiso de ejecución:

```bash
chmod +x to-production.sh to_production_static.sh
```

### 1.3 Crear el historial de versiones

```bash
mkdir -p docs/versions
printf "2026-09-18 | 0.0.1 | - | Release inicial | -\n" > docs/versions/version-history.txt
```

Formato de cada línea (igual en ambas herramientas):

```
FECHA | VERSION | TIPO | DESCRIPCION | ARCHIVO
```

> [!info] Cómo se lee el historial
> El script toma la **última línea** (`tail -n 1`), extrae el 2º campo con `awk -F'|'`, le quita los espacios y lo usa como versión base. Si ese campo **no** cumple `^X.Y.Z$`, el script **aborta** — corrige la última línea antes de publicar.

---

## 2. `to-production.sh` (apps con Dockerfile)

### Qué debes cambiar en tu proyecto

| Variable | Qué poner | Ejemplo |
|----------|-----------|---------|
| `SERVICE_DIRECTORY` | Carpeta que contiene el `Dockerfile` (contexto de build) | `"${REPOSITORY_ROOT}/tesj-core"` |
| `IMAGE_NAME` | Nombre de tu imagen | `"backend-tsj"` |
| `IMAGE_TAG` | Etiqueta base (siempre `latest`) | `"latest"` |
| `SERVER_PROFILES` | Tus servidores: `"alias\|usuario@host\|/ruta/remota/para/imagenes"` | ver §4 |

Además tu app debe tener un `Dockerfile` dentro de `SERVICE_DIRECTORY`.

> [!tip] Dockerfile mínimo (backend .NET)
> ```dockerfile
> FROM mcr.microsoft.com/dotnet/aspnet:10.0-alpine AS base
> WORKDIR /app
> EXPOSE 8080
> FROM mcr.microsoft.com/dotnet/sdk:10.0-alpine AS build
> WORKDIR /src
> COPY . .
> RUN dotnet publish -c Release -o /app/publish --no-restore
> FROM base AS final
> COPY --from=build /app/publish .
> ENTRYPOINT ["dotnet", "tu-app.dll"]
> ```

### Funcionamiento

1. Calcula la siguiente versión desde `version-history.txt` (`-M` mayor, `-U` menor, `-F` fix).
2. Construye con **buildx** para `linux/amd64` con doble etiqueta: `imagen:<version>` e `imagen:latest`.
3. Comprime con `docker save ... | gzip` en `Images/<imagen>-<version>-<sha>-<fecha>.tar.gz`.
4. Crea la ruta remota con `ssh`, sube el `.tar.gz` con `scp` y lo carga con `ssh "docker load"`.
5. Registra el release en `version-history.txt`.

> [!note] El script SOLO entrega la imagen
> Para que el servidor use la nueva versión, tu `compose.yml`/`Caddyfile` referencia `imagen:latest` (o `imagen:<version>`) y se aplica con `docker compose up -d <servicio>`. Publicar no equivale a reiniciar el servicio.

### Uso

```bash
./to-production.sh --list-servers
./to-production.sh --dry-run -S tecmm -F "descripcion"   # no construye ni sube
./to-production.sh -S tecmm -F "descripcion del fix"
./to-production.sh -S vinc -U "nueva funcionalidad"
./to-production.sh -S tecmm -M "version mayor"
```

---

## 3. `to_production_static.sh` (sitios estáticos)

### Qué debes cambiar en tu proyecto

| Variable | Qué poner | Ejemplo |
|----------|-----------|---------|
| `BUILD_COMMAND` | El **build de tu herramienta** | `"npm ci && npm run build"`, `"ng build"`, `"vite build"` |
| `DIST_DIRECTORY_NAME` | Carpeta que genera tu build | `dist`, `build`, `out`, `.output/public`, ... |
| `SERVER_PROFILES` | Tus servidores con la **ruta web servida** del sitio: `"alias\|usuario@host\|/ruta/web/servida"` | ver §4 |

No se necesita `Dockerfile`: la herramienta ya compila el sitio.

### Funcionamiento (sistema de "tags" como docker)

El web server / proxy del servidor apunta **SIEMPRE** a `latest`:

```
<RUTA_WEB>/latest
```

Ejemplo (Caddy):

```caddyfile
root * /home/tecmm-mx-infra/www/mi-app/latest
```

Flujo del script:

1. Calcula la siguiente versión desde `version-history.txt`.
2. Ejecuta `BUILD_COMMAND` (se omite con `--skip-build`).
3. Valida que la carpeta de build tenga contenido.
4. **Verifica con `find`** que **no exista otra carpeta `latest`** dentro de la ruta definida (más de una → aborta).
5. Si existe `latest` remota, la **archiva**: `mv <ruta>/latest <ruta>/<version-actual>`.
6. Sube el build nuevo como `latest` (`scp -r`).
7. Registra el release en `version-history.txt`.

Resultado en el servidor:

```
www/mi-app/
├── 0.0.1/        # version archivada
├── 0.0.2/        # version archivada
└── latest/       # SIEMPRE apuntado por el web server
```

> [!example] Primer despliegue
> Si aún **no existe** la carpeta `latest` en el servidor, no hay nada que archivar: el script solo crea y llena `latest` (la semilla `0.0.1` del historial queda registrada aunque no haya carpeta `0.0.1` física).

> [!warning] No romper `latest`
> El `find` guarda contra estructuras rotas (ej. copias `latest` anidadas). Si el script aborta con "hay N carpetas latest", resuélvelas manualmente — nunca borres una versión que aún necesites.

### Uso

```bash
./to_production_static.sh --list-servers
./to_production_static.sh --dry-run -S tecmm -F "descripcion"
./to_production_static.sh -S tecmm -F "descripcion del fix"
./to_production_static.sh -S vinc -U "nueva funcionalidad"
./to_production_static.sh -S tecmm -M "version mayor"

# Re-subir el build existente sin recompilar (rollback de codigo)
./to_production_static.sh --skip-build -S tecmm -F "re-subir build"
```

### Rollback a una versión anterior

```bash
ssh root@tecmm.mx "rm -rf /ruta/web/mi-app/latest && mv /ruta/web/mi-app/0.0.1 /ruta/web/mi-app/latest"
```

---

## 4. Versionado + multiserver (común a ambos)

### Reglas de versionado

| Flag | Incremento | Resultado |
|------|-----------|-----------|
| `-F` | fix | `x.y.Z` |
| `-U` | menor | `x.Y.0` |
| `-M` | mayor | `X.0.0` |

- **Semilla inicial**: siempre `0.0.1`.
- La siguiente versión se calcula de la última línea del historial (ver §1.3).
- **`-S`** indica el servidor; una ejecución publica **un solo servidor**.

> [!tip] Publicar en varios servidores
> Ejecuta el script **una vez por servidor** con su `-S`: el historial queda una sola vez (la edición del archivo remoto no afecta al historial local).

### Agregar un servidor

Añade una línea a `SERVER_PROFILES` en el script (cada app ajusta la ruta a su necesidad):

```bash
SERVER_PROFILES=(
  "tecmm|root@tecmm.mx|/home/tecmm-mx-infra/app-images"
  "vinc|root@vinculacion.tsj.mx|/root/vinculacion-apps/docker-images"
  "mi-server|usuario@host|/ruta/remota"
)
```

> [!note] Ruta según herramienta
> En `to-production.sh` la ruta es el **folder de imágenes** (ej. `/home/tecmm-mx-infra/app-images`). En `to_production_static.sh` es la **ruta web servida** del sitio (ej. `/home/tecmm-mx-infra/www/mi-app`).

### Buenas prácticas

- Corre siempre `--dry-run` antes de publicar para validar servidor, versión y descripción.
- **Comitea** antes de publicar: en `to-production.sh` el SHA del repo queda embebido en el archivo (`<imagen>-<version>-<sha>-<fecha>.tar.gz`).
- El `.tar.gz` local queda en `Images/` para re-uso; el historial se edita **solo** línea a línea (el script agrega al final).

---

## 5. Troubleshooting

| Síntoma | Solución |
|---------|----------|
| SSH pide contraseña | Revisa §1.1: tu llave pública debe estar en `~/.ssh/authorized_keys` del usuario remoto. |
| `No se encontro ... Dockerfile / version-history` | Ajusta `SERVICE_DIRECTORY` (solo Docker) o crea `docs/versions/version-history.txt` con la semilla. |
| El build no genera contenido | Verifica que `BUILD_COMMAND` y `DIST_DIRECTORY_NAME` coincidan con la salida real de tu herramienta. |
| "La ultima version no tiene el formato esperado" | Corrige la **última línea** del historial a `... \| X.Y.Z \| ...` y vuelve a correr. |
| "hay N carpetas latest" | Existen `latest` duplicadas bajo la ruta; resuélvelas manualmente antes de publicar. |
| `docker load` falla por espacio | En el servidor: `df -h` y limpia con `docker system prune` (cuidado con imágenes en uso). |
| El SHA no cambia | Hay cambios sin commitear; el SHA refleja el último commit, no el working tree. |

---

## Relacionado

- [[Proyecto]] — vista general del proyecto.
- [[Tsj-Web-plan-desarollo]] — plan de desarrollo con FASEs, pendientes y decisiones.
- [[backend-manual]] — contrato de la API del backend `tsj-core` (referencia de lo que publica `to-production.sh`).
- Repos que ya usan este sistema: `tsj-web` (`backend/to-production.sh`, `backend/to_production_static.sh`).

---

## Referencias de archivos

| Archivo | Descripción |
|---------|-------------|
| `to-production.sh` | Despliegue de imágenes Docker (buildx → save → scp → `docker load`). |
| `to_production_static.sh` | Despliegue de sitios estáticos (build → `scp` → `latest`). |
| `docs/versions/version-history.txt` | Historial de versiones (`FECHA \| VERSION \| TIPO \| DESCRIPCION \| ARCHIVO`); semilla `0.0.1`. |
| `Images/` | Archivos `.tar.gz` generados por `to-production.sh` (no versionado; ignorado por git). |
| `docker buildx` | Builder multi-plataforma usado para `linux/amd64`. |