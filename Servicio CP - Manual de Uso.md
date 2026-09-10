# 📮 Servicio CP — Manual de Uso

> Microservicio REST en **Go** que expone el catálogo oficial de Códigos Postales de México (SEPOMEX) bajo la subruta `/api-cp/`. Soporta la **cascada completa** `Estado → Municipio → Ciudad → Colonia` para selects dinámicos, consulta por CP estilo buscador de direcciones y búsqueda por nombre. **CORS abierto**.

## Descripción general

| | |
|---|---|
| **Lenguaje** | Go 1.27 (`net/http` stdlib) |
| **Puerto** | `8050` |
| **Ruta base** | `/api-cp/` |
| **Datos** | CSV SEPOMEX · ~159,000 registros en memoria |
| **Runtime** | Docker (alpine) · gestionado por compose de tecmm-mx-infra |

**Estructura del código:**
- `cmd/server/main.go` — arranque y carga del catálogo
- `internal/store/` — índices en memoria (por CP, por ciudad, por municipio, búsqueda)
- `internal/api/` — handlers HTTP + middleware CORS
- `deploy/Caddyfile.api-cp` — bloque de reverse proxy de referencia

## Ejecución local

### Opción A — Go directo
```bash
go run ./cmd/server
# Escucha en :8080 (default) usando ./data/CPdescarga.csv
```

### Opción B — Docker Compose
```bash
docker compose up --build
# http://localhost:8050
```

### Variables de entorno

| Variable | Default (local) | Descripción |
|---|---|---|
| `CP_ADDR` | `:8080` | Dirección de escucha HTTP |
| `CP_CSV_PATH` | `data/CPdescarga.csv` | Ruta del catálogo SEPOMEX |

> [!info] Imagen Docker
> En la imagen ya viene configurado `CP_ADDR=:8050` y `CP_CSV_PATH=/data/CPdescarga.csv` — el CSV va **incrustado** en la imagen, no requiere volúmenes.

## Endpoints

**Base URL:**
- Local (Docker): `http://localhost:8050`
- Producción pública (vía Caddy): `https://tecmm.mx/api-cp/`
- Producción directa (solo desde el servidor): `http://127.0.0.1:8050`

Todos los endpoints son **GET** y responden `application/json; charset=utf-8`. Además de `/api-cp/health`, existe `GET /health` en raíz como healthcheck interno del contenedor.

### 1. GET /api-cp/health
Verifica que el servicio esté vivo e indica cuántos registros cargó.

```bash
curl https://tecmm.mx/api-cp/health
```

```json
{"registros":159102,"status":"ok"}
```

### 2. GET /api-cp/cp/{codigo}
Consulta un código postal — devuelve estado, municipios, ciudades y todas las colonias asociadas en una sola llamada.

| Parámetro | Tipo | Regla |
|---|---|---|
| `codigo` | path | Exactamente **5 dígitos** numéricos |

```bash
curl https://tecmm.mx/api-cp/cp/06600
```

```json
{
  "c_estado": "09",
  "ciudades": [{"clave": "06", "nombre": "Ciudad de México"}],
  "codigo_postal": "06600",
  "colonias": [
    {
      "codigo_postal": "06600",
      "asenta": "Juárez",
      "tipo_asenta": "Colonia",
      "municipio": "Cuauhtémoc",
      "estado": "Ciudad de México",
      "ciudad": "Ciudad de México",
      "c_estado": "09",
      "c_mnpio": "015",
      "id_asenta_cpcons": "0930",
      "zona": "Urbano",
      "c_cve_ciudad": "06"
    }
  ],
  "estado": "Ciudad de México",
  "municipios": [{"clave": "015", "nombre": "Cuauhtémoc"}]
}
```

**Errores:**
- `400` — formato inválido: `{"error":"el código postal debe tener 5 dígitos"}`
- `404` — no existe: `{"error":"código postal no encontrado: 00000"}`

### 3. GET /api-cp/estados
Catálogo completo de estados (32 entidades), ordenado por nombre.

```bash
curl https://tecmm.mx/api-cp/estados
```

```json
[
  {"clave": "01", "nombre": "Aguascalientes"},
  {"clave": "02", "nombre": "Baja California"},
  ...
]
```

La `clave` es la clave INEGI del estado y se usa en los endpoints siguientes.

### 4. GET /api-cp/estados/{clave}/municipios
Municipios de un estado, ordenados por nombre.

```bash
curl https://tecmm.mx/api-cp/estados/14/municipios
```

```json
[
  {"clave": "001", "nombre": "Acatic"},
  {"clave": "039", "nombre": "Guadalajara"},
  ...
]
```

**Errores:**
- `404` — clave inexistente: `{"error":"estado no encontrado: 99"}`

### 5. GET /api-cp/estados/{clave}/municipios/{clave_mnpio}/ciudades
Ciudades presentes dentro de un municipio específico (cascada nivel 2). Solo incluye ciudades que realmente tienen colonias en ese municipio; un municipio rural puede devolver `[]`.

```bash
curl https://tecmm.mx/api-cp/estados/14/municipios/039/ciudades
```

```json
[{"clave": "03", "nombre": "Guadalajara"}]
```

**Errores:**
- `404` — estado inexistente: `{"error":"estado no encontrado: ZZ"}`
- `404` — municipio inexistente: `{"error":"municipio no encontrado en estado 14: 999"}`

### 6. GET /api-cp/estados/{clave}/municipios/{clave_mnpio}/ciudades/{clave_ciudad}/colonias
Colonias de una ciudad dentro de un municipio (cascada completa). Usa las claves obtenidas en el endpoint anterior.

```bash
curl https://tecmm.mx/api-cp/estados/14/municipios/039/ciudades/03/colonias
```

```json
{
  "clave_estado": "14",
  "estado": "Jalisco",
  "clave_municipio": "039",
  "nombre_municipio": "Guadalajara",
  "clave_ciudad": "03",
  "nombre_ciudad": "Guadalajara",
  "total": 434,
  "colonias": [
    {
      "codigo_postal": "44100",
      "asenta": "Guadalajara Centro",
      "tipo_asenta": "Colonia",
      "municipio": "Guadalajara",
      "estado": "Jalisco",
      "ciudad": "Guadalajara",
      "c_estado": "14",
      "c_mnpio": "039",
      "id_asenta_cpcons": "0003",
      "zona": "Urbano",
      "c_cve_ciudad": "03"
    },
    ...
  ]
}
```

**Errores:**
- `404` — estado inexistente
- `404` — municipio inexistente en ese estado
- `404` — ciudad ajena a ese municipio: `{"error":"ciudad no encontrada en municipio 120 (14): 03"}`

> [!note] Ciudades que cruzan municipios
> Una misma ciudad puede tener colonias en varios municipios, y las claves de ciudad son locales al contexto. Por eso cada nivel valida que la clave exista *dentro* de su padre.

### 7. GET /api-cp/estados/{clave}/ciudades
Todas las ciudades/zonas metropolitanas de un estado (sin filtro por municipio).

```bash
curl https://tecmm.mx/api-cp/estados/14/ciudades
```

```json
[
  {"clave": "55", "nombre": "Acatlán de Juárez"},
  {"clave": "24", "nombre": "Ahualulco de Mercado"},
  ...
]
```

**Errores:** igual que municipios (`404` si el estado no existe).

### 8. GET /api-cp/estados/{clave}/ciudades/{clave_ciudad}/colonias
Colonias de una ciudad a nivel estado (agrupa todos sus municipios). Ejemplo: Jalisco/Guadalajara(03) devuelve 434 colonias.

**Errores:** igual que el endpoint anterior.

### 9. GET /api-cp/buscar
Busca colonias/asentamientos por nombre (coincidencia parcial, insensible a acentos).

| Parámetro | Requerido | Regla |
|---|---|---|
| `q` | Sí | Texto a buscar |
| `limit` | No | Entero entre 1 y 500. Default: `50` |

```bash
curl "https://tecmm.mx/api-cp/buscar?q=guadalajara&limit=2"
```

**Errores:**
- `400` — falta `q`: `{"error":"parámetro requerido: q"}`
- `400` — límite inválido: `{"error":"limit debe ser un entero entre 1 y 500"}`

#### Notas sobre la búsqueda
- **Ignora acentos y mayúsculas**: buscar `san jose` encuentra `San José`.
- Ignora espacios, puntos, comas, guiones y guiones bajos.
- Es coincidencia *contiene*: `tepic` también matchea `Batepic`.
- Si hay más resultados que `limit`, la respuesta se trunca sin indicar paginación.

## CORS

Middleware global aplicado a todas las rutas.

| Header | Valor |
|---|---|
| `Access-Control-Allow-Origin` | `*` |
| `Access-Control-Allow-Methods` | `GET, OPTIONS` |
| `Access-Control-Allow-Headers` | `*` |

Los preflights `OPTIONS` responden `204 No Content` inmediatamente.

## Formato de error estándar
Todo error responde con el mismo shape:

```json
{"error": "mensaje descriptivo"}
```

| Código | Casos |
|---|---|
| `400` | CP sin 5 dígitos · falta `q` · `limit` fuera de rango |
| `404` | CP inexistente · estado inexistente · municipio inexistente en estado · ciudad inexistente en municipio/estado |
| `405` | Método distinto a GET |

## Despliegue a producción

El servicio vive en el compose de `tecmm-mx-infra` (`image: cp-service:latest`). El script construye, sube y **carga la imagen + recrea el servicio** vía compose.

### Uso
```bash
./to-production.sh -M "Nueva funcionalidad mayor"   # X.0.0
./to-production.sh -U "Feature o mejora"            # x.Y.0
./to-production.sh -F "Corrección de bug"           # x.y.Z
```

### Qué hace paso a paso
1. Incrementa la versión según la flag y lee la última de `docs/versions/version-history.txt`
2. Construye la imagen `cp-service:latest` para **linux/amd64** (CSV incrustado)
3. La guarda comprimida en `Images/cp-service-{version}-{sha}-{timestamp}.tar.gz`
4. La sube vía SCP a `root@tecmm.mx:/home/tecmm-mx-infra/app-images/`
5. Por SSH: `docker load` → `cd /home/tecmm-mx-infra && docker compose --env-file <env autodetectado> up -d --no-deps cp-service` → health-check contra `/health` (15 intentos × 2s)
6. Registra fecha, versión, descripción y archivo en `docs/versions/version-history.txt`

> [!warning] Si falla el health-check
> El release **no** se registra en el historial. Revisa con `docker compose --env-file .env.production logs cp-service`.

### Exposición pública con Caddy
Bloque ya configurado en `/home/tecmm-mx-infra/caddy/Caddyfile` (referencia en `deploy/Caddyfile.api-cp`):
```caddyfile
handle /api-cp/* {
    reverse_proxy cp-service:8050
}
```

Recargar Caddy tras cambios:
```bash
docker exec tecmm-mx-infra-caddy-1 caddy fmt --overwrite /etc/caddy/Caddyfile
docker exec tecmm-mx-infra-caddy-1 caddy validate --config /etc/caddy/Caddyfile
docker exec tecmm-mx-infra-caddy-1 caddy reload --config /etc/caddy/Caddyfile
```

### Verificación post-deploy
```bash
curl https://tecmm.mx/api-cp/health
curl https://tecmm.mx/api-cp/estados/14/municipios/039/ciudades
curl -i -X OPTIONS https://tecmm.mx/api-cp/estados   # 204 con headers CORS
```

### Rollback manual
Las imágenes anteriores quedan en `/home/tecmm-mx-infra/app-images/` del servidor:
```bash
ssh root@tecmm.mx
gunzip -c /home/tecmm-mx-infra/app-images/<archivo-anterior>.tar.gz | docker load
cd /home/tecmm-mx-infra && docker compose --env-file .env.production up -d --no-deps cp-service
```

### Historial de versiones
`docs/versions/version-history.txt` — una línea por release:
```
2026-08-25 | 1.0.0 | M | Servicio de Servicios postales | cp-service-1.0.0-51fab6a-20260825-134606.tar.gz
2026-08-25 | 1.1.0 | U | Cascada municipio->ciudades->colonias | cp-service-1.1.0-51fab6a-20260825-153505.tar.gz
```
