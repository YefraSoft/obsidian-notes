---
tipo: sub-plan
titulo: "Fase 2 — Funnel completo, paridad con v1"
autor: "Efraín García"
creado: 2026-10-29
actualizado: 2026-11-01
estado: pendiente
proyecto: nummex
padre: "[[nummex-backend-v2-plan]]"
tags:
  - nummex
  - sub-plan
  - backend-v2
  - fase-2
  - libro-azul
  - minio
  - odoo
  - citas
---

# Fase 2 — Funnel completo, paridad con v1

> **Plan padre:** [[nummex-backend-v2-plan]] · **Fase anterior:** [[nummex-backend-v2-fase-1-sub-plan]]
> Reemplazar lo que hoy hace v1 en el funnel del prospecto (cotización con Libro Azul, datos de solicitud y documentos) sobre el esquema nuevo, más zonas y citas con disponibilidad, y dejar la cola de Odoo con **uploaders** intercambiables (legacy ahora, JSON-2 después).

## Pendientes

- [ ] Confirmar con el frontend el contrato nuevo: catálogo como `[{ key, name }]` (v1 duplicaba `years`/`rows` con `clave`/`nombre`), errores del proveedor como 502 en español (v1 daba 400), subida de documentos en un solo paso y `relationship` como enum (v1 texto libre).
- [ ] Sincronización con Odoo de vehículo, cotización, datos personales, etapas del lead y citas: **pendiente** hasta tener los datos de flotilla (hoy solo se sincronizan el alta y los archivos).
- [ ] Lectura de los datos del funnel (`GET`): solo cuando el prospecto sea cliente (Fase 3). Los `POST` son solo para la cookie de sesión del prospecto.
- [ ] MinIO: el servicio ya está en el `docker-compose.yml` (licencia local `minio.license`, sin desplegar aún); URLs firmadas para leer archivos (Fase 3).
- [ ] Formato de RFC y CURP, rango del año del vehículo y máximo de referencias por solicitud: hoy solo rigen las reglas del esquema (v1 tampoco las validaba).
- [x] Citas: un contacto tiene una sola cita futura y agendar otra la reemplaza (decidido); cancelar queda fuera de esta fase.
- [ ] Cuota de archivos por contacto y limpieza de objetos huérfanos en MinIO.
- [x] Nombres propuestos para la cola: `OdooUploadWorker` (consumidor), `IOdooUploader`, `OdooUploaderBase`, `LegacyOdooUploader`, `Json2OdooUploader`; confirmar.
- [x] Convivencia con v1 en Caddy: `/api/chat*` va a v1 y el resto de `/api` a `backend-v2` (ver [[nummex-frontend-v2-fase-a-sub-plan]]); las rutas de `/api/loan-vehicle/*` y `/api/loan-quotes` se conservan.
- [ ] Decidir el detalle de los 502: hoy `ErrorDescriptors` oculta el mensaje en español de `InExternalServiceException` (regla de `AGENTS.md`: desde 500, detalle genérico) y `GlobalExceptionHandler` registra la traza completa de cada 502; el Slice 5 los deja así.
- [ ] Formulario (nummex-web): línea «cotizar un vehículo nuevo» con flechas `<` `>` para moverse entre los registros capturados; sin `GET` hasta la Fase 3, el cliente lleva los registros de la sesión (las respuestas de `vehicle` y `quote` devuelven el registro guardado); decidir si hace falta una lectura propia con cookie para releerlos tras recargar.
- [ ] `offeredAmount` de la cotización lo envía el cliente (igual que v1); recalcularlo en el servidor exigiría consultar el precio de Libro Azul.
- [ ] Documentos (Slice 7): `FAILED` es terminal y no hay forma manual de reencolarlo; un adjunto lento bloquea la cola secuencial del worker; partes `file` o `kind` repetidas se aceptan tomando la primera (valorar 400); GIF fuera de los tipos permitidos por decisión del responsable; nombre del objeto `{contactId}_{yyyyMMdd}_{doc-name}_{6 hex}.{ext}`; MinIO AIStor en el compose con licencia local `minio.license` (ignorada por git) y sin puertos publicados al host (recomendado: no publicar 9000/9001 o limitarlos a `127.0.0.1`).
- [ ] Rate limiting por IP: oficinas o redes con NAT comparten cubo. Con `header_up X-Forwarded-For {client_ip}` en Caddy y `ForwardedHeaders` en la API se usa la IP real (`CF-Connecting-IP`); sin esa cabecera Caddy toma la del cliente. Confirmar tras desplegar con una petición real por el túnel, y valorar `client_ip_headers CF-Connecting-IP` solo.

## Estado Actual

| Área | Estado |
|------|--------|
| **Paso 0** — cola de Odoo con uploaders (`OdooJob`, `IOdooQueue`, `OdooUploadWorker`, `IOdooUploader`, `LegacyOdooUploader`, `Json2OdooUploader`) | ✅ |
| **Slice 5** — cotización con Libro Azul (catálogo, valuación al 70 %, caché) | ✅ |
| **Slice 6** — datos de solicitud (vehículo, cotización, datos, referencias) | ✅ |
| **Slice 7** — documentos en MinIO y envío a Odoo por la cola | ✅ |
| **Slice 8** — zonas desde `appsettings` y citas con disponibilidad | 🟡 código y enlace listos; verificación contra Postgres pendiente (la hace el responsable) |

## Alcance

Cubre la cotización pública con Libro Azul, los pasos del funnel que guarda el prospecto con su cookie de sesión, la subida de documentos, las zonas y las citas. **No cubre** la sincronización con Odoo de los pasos del funnel ni de las citas (pendiente), la lectura de esos datos (Fase 3, cuando sea cliente), ni clientes, préstamos, pagos y cobranza (Fase 3) ni el chat (Fase 4).

## Forma de trabajo

Igual que la Fase 1 (ver [[nummex-backend-v2-fase-1-sub-plan]]): cada slice se parte en **CREAR → MODIFICAR → ENLAZAR**, cada paso lo ejecuta un sub-agente Haiku con prompt autocontenido, quien coordina revisa el diff, corre `dotnet build` (0 errores y 0 advertencias), verifica con curl y propone el commit (sin `Co-Authored-By`); no se lanza el siguiente paso sin el visto bueno del responsable, que hace los commits. Los agentes no escriben tests. Las clases que implementan `IModule` se activan al crearse, por eso los módulos solo se crean en ENLAZAR. Las verificaciones usan Postgres temporal y servicios simulados (Libro Azul, Odoo legacy y JSON-2) o contenedores temporales (MinIO), con credenciales generadas al vuelo, solo PID propios y limpieza al terminar.

Decisiones de esta fase:

- **Archivos en MinIO**, no en Cloudinary (el plan original decía Cloudinary; v1 en realidad guarda en Odoo y Cloudinary es código muerto). Se envían a Odoo por la cola **como binario**: en este Odoo los adjuntos son `type = binary` con `store_fname` (filestore en disco) y `db_datas` vacío, es decir, no se guardan en Postgres (verificado con el MCP de Odoo).
- **Cola de Odoo con roles separados:** el **worker** solo sabe reintentos y a dónde va cada mensaje (la cola entrega mensajes tipados: Contacto, Archivo…); el **uploader** trata la subida y transforma el modelo local al modelo de Odoo. El estado en la base (`odoo_state`, `odoo_created_at`, ids de Odoo) es de **comprobación administrativa y recuperación ante caídas** y se agrega a **cada tabla que se sincroniza**.
- **Dos uploaders que heredan de `IOdooUploader`:** `LegacyOdooUploader` (JSON-RPC `execute_kw`, como v1; **el que se usa ahora**) y `Json2OdooUploader` (JSON-2 de Odoo 20, construido en el Slice 3; **se activa después**). Se elige con `Odoo__Api` (`Legacy` por defecto | `Json2`); al migrar solo cambia la implementación.
- **Pasos del funnel solo con la cookie de sesión** (`POST`, política `Prospect`, solo sobre el propio contacto: 403 si no). Sin `GET`.
- **`prospects.funnel` sube en cada paso** a `Math.Max(actual, etapa de la tabla)` con `FunnelStages`, en el mismo `CommitAsync`.
- **Zonas desde `appsettings`** (el frontend las lista) y **citas con disponibilidad configurable**: lunes a viernes de 09:00 a 17:00, citas de 30 minutos, **una cita por zona y horario**.
- **Contrato de errores:** el 4xx lleva mensaje en español y el 5xx detalle genérico; los fallos de Libro Azul y Odoo salen como `InExternalServiceException` (502).

## Implementación

### Paso 0 — Cola de Odoo con uploaders (antes del Slice 5)

**Bloque 1 — CREAR (aditivo)**
- [x] `Application/Odoo/`: `OdooJob(Kind, Id)`, `OdooJobKind` (`Contact`; `File` en el Slice 7), `IOdooQueue` (`TryEnqueue(OdooJob)`, `ReadAllAsync`) e `IOdooUploader` (`UploadContactAsync`; después `UploadFileAsync`).
- [x] `Infrastructure/Odoo/OdooUploaderBase`: transformación a `res.partner` y `crm.lead` y flujo idempotente (busca antes de crear, guarda cada id al obtenerlo, reintenta el guardado). Es lo que hoy hace `OdooSyncProcessor`; solo deja abstractas las operaciones de transporte (`create`, `write`, `find`).
- [x] `Json2OdooUploader`: transporte JSON-2 (hoy `OdooClient` sobre `IExternalApiClient`).
- [x] `LegacyOdooUploader`: `POST {BaseUrl}/jsonrpc` con `common.login` (`Odoo__Database`, `Odoo__Username`, `Odoo__ApiKey`) para obtener el `uid` (en caché, con un reintento ante error) y `object.execute_kw` con `create` (`[values]`), `write` (`[[id], values]`) y `search_read` (`[domain]`, `fields`, `limit`); las credenciales van en el cuerpo, no en cabeceras. Mismas variables que v1.

**Bloque 2 — MODIFICAR (sin cambiar comportamiento)**
- [x] `OdooSyncQueue` → `IOdooQueue` con mensajes tipados; `OdooSyncWorker` → `OdooUploadWorker` (consumidor y barrido de `PENDING` de todas las tablas sincronizadas, reintentos con backoff y tope, `FAILED` que no se reintenta solo, despacho por `Kind`); `OdooSyncProcessor` → base del uploader.
- [x] `ProspectService.CreateAsync` encola `OdooJob(Contact, contactId)`; `OdooOptions.Api` (`Legacy` | `Json2`); `IOdooSyncRepository` lista pendientes y registra el estado por tabla.

**Bloque 3 — ENLAZAR**
- [x] `OdooModule` registra el uploader según `Odoo:Api` (por defecto `Legacy`).
- [x] Verificar contra un Odoo **simulado de JSON-RPC** los escenarios del Slice 3 (alta rápida, Odoo caído con reintentos y `FAILED`, un `FAILED` no se rehidrata, reinicio, idempotencia, error de autenticación, sin configuración) y que con `Odoo__Api=Json2` el comportamiento es el mismo.

### Slice 5 — Cotización con Libro Azul (público)

**Bloque 1 — CREAR**
- [x] `Application/LibroAzul/`: `ILibroAzulClient` (años, marcas, modelos, versiones, precio), `CatalogItem(Key, Name)`, `LibroAzulPrice(Venta, Compra, Moneda)` y `LoanValuationPolicy` (70 % de `Venta`, 2 decimales `AwayFromZero`, mayor que 0 y hasta 100,000,000; `Venta > 0`, `Compra ≥ 0`, moneda no vacía de hasta 16 caracteres y sin control; si no cumple, error de servicio externo).
- [x] `Infrastructure/LibroAzul/`: `LibroAzulOptions` (`LibroAzul__BaseUrl` https absoluta sin usuario, query ni fragmento, `LibroAzul__Usuario`, `LibroAzul__Contrasena`; tolerante al arrancar, falla al usar), `LibroAzulAuthStrategy` (llave de `GET /api/Sesion`, 90 min, un solo semáforo, `InvalidateAsync` ante 401), `LibroAzulClient` sobre `IExternalApiClient` (timeout de 10 s; `Años`, `Marcas` y `Modelos` por POST, `Versiones` por GET y `Precio` por POST con `ClaveVersion` y `Kilometraje`) y caché con `IMemoryCache` (6 h catálogos, 5 min precio, tope de entradas y una sola llamada en vuelo por clave).
- [x] Contratos y endpoints sin mapear: `GET /api/loan-vehicle/years|brands|models|versions` y `POST /api/loan-quotes` (`versionKey` de 1 a 100 sin caracteres de control; `mileageKm` de 0 a 9,999,999; respuesta `offeredAmount`, `venta`, `compra`, `moneda`; no se persiste).

**Bloque 2 — MODIFICAR**
- [x] La abstracción compartida admite autenticación por **query** (Libro Azul manda `Llave` en la query): `IExternalAuthStrategy` devuelve credenciales con cabeceras y query, y `ExternalRequestFactory` las aplica. Se adapta `OdooAuthStrategy`; sin cambio de comportamiento en Odoo.

**Bloque 3 — ENLAZAR**
- [x] Módulo de Libro Azul (cliente con nombre `libroazul`, estrategia, caché) y rutas con `Relaxed` en el catálogo y `Moderate` en la cotización; nombres de variables en `.env.example`.
- [x] Verificar contra un Libro Azul **simulado**: catálogo en cascada, valuación (70 %, redondeo y tope), validaciones de entrada (400), respuesta inválida del proveedor (502), 401 con renovación de la llave, timeout, caché (una sola llamada al proveedor en 6 h / 5 min) y sin configuración (error claro, la API arranca).

### Slice 6 — Datos de solicitud (`POST`, cookie de prospecto)

**Bloque 1 — CREAR**
- [x] `Application/Funnel/` o por dominio: contratos con validación en español: vehículo (marca y modelo ≤ 100, año > 0, versión ≤ 100, tipo ≤ 50, kilometraje ≥ 0), cotización (monto ofrecido y solicitado ≥ 0, plazo > 0), datos (domicilio ≤ 255, código postal ≤ 10, RFC ≤ 15, CURP ≤ 18, ingreso y gasto ≥ 0) y referencias (lista no vacía; nombre ≤ 150, teléfono de 10 a 15 dígitos y `relationship` como enum `PARENT, SIBLING, FRIEND, PARTNER, CHILD, GRANDPARENT, UNCLE_AUNT, COUSIN, COWORKER` con `IsEnum`).
- [x] Servicios y repositorios por intención (`IVehicleService`, `IQuoteService`, `IDetailService`, `IReferenceService`; `ContactDetail` por upsert, 1:1) con `ProspectPermissions.RequireOwner` y un solo `CommitAsync`; cada paso sube `prospects.funnel` con `Math.Max(actual, etapa)`.
- [x] Endpoints sin mapear: `POST /api/prospects/{contactId}/vehicle` (devuelve el `id` del vehículo), `/quote`, `/detail` y `/references`.

**Bloque 2 — MODIFICAR**
- [x] Los índices por `contact_id` ya existían en `initial.sql` (no hizo falta cambiarlo); `IFunnelStatusChecker` se usa para recalcular la etapa.

**Bloque 3 — ENLAZAR**
- [x] Módulo y rutas con política `Prospect` y límite `Moderate`.
- [x] Verificar contra Postgres: cada paso con cookie (201), sin cookie (401), de otro contacto (403), cuerpos inválidos (400 con errores por campo), referencias vacías o con parentesco fuera del enum (400), upsert de datos, `funnel` ascendente y nunca descendente, y sin lecturas expuestas.

### Slice 7 — Documentos en MinIO y envío a Odoo

**Bloque 1 — CREAR**
- [x] Esquema (edición directa de `initial.sql`): tabla `file_uploads` (`id`, `contact_id`, `vehicle_id` opcional, `kind`, `object_key`, `original_name`, `mime_type`, `size_bytes`, `odoo_state` `PENDING | SYNCED | FAILED`, `id_odoo_attachment` único y nulo, `odoo_created_at`, `created_at`) con índice parcial de `PENDING`. Es la tabla que se sincroniza: `contact_documents` y `vehicle_documents` guardan varios archivos por fila y no pueden llevar un estado por archivo.
- [x] `Application/Storage/IFileStorage` (`PutAsync`, `GetAsync`, `ExistsAsync`) y `Infrastructure/Storage/MinioFileStorage` con AWSSDK.S3 (compatible con MinIO, `ForcePathStyle`); variables `Storage__Endpoint`, `Storage__Bucket`, `Storage__AccessKey`, `Storage__SecretKey`, `Storage__UseSsl`; clave de objeto `prospects/{contactId}/{kind}/{guid}{ext}`.
- [x] Servicio de documentos: `kind` (`ID_CARD`, `PROOF_OF_ADDRESS`, `PROOF_OF_INCOME`, `INVOICE`, `REGISTRATION_CARD`, `TAX_RECEIPT`, `NO_LIEN_CERTIFICATE`, `INSURANCE_POLICY`; el de vehículo exige `vehicleId` del propio contacto), máximo 10 MB, extensiones `.png .jpg .jpeg .webp .gif .pdf` (sin `.svg`) con validación de la firma real del archivo, nombre sanitizado; guarda la clave del objeto en la columna `*_url` y la fila de `file_uploads`; sube la etapa del funnel (documentos personales o del vehículo).
- [x] `UploadFileAsync` en la base del uploader y en cada implementación: **espera a que exista el lead** (si no, queda `PENDING` sin gastar intentos), baja el archivo de MinIO y crea el `ir.attachment` (`type = binary`, `res_model = crm.lead`, `res_id` del lead, `mimetype`, `datas` en base64, privado), idempotente (busca antes por un nombre único con el id de `file_uploads`).

**Bloque 2 — MODIFICAR**
- [x] `OdooJobKind.File`, despacho en `OdooUploadWorker` y rehidratación de `file_uploads` pendientes en el barrido.

**Bloque 3 — ENLAZAR**
- [x] Módulo y `POST /api/prospects/{contactId}/documents` (multipart, política `Prospect`, `Moderate`, 10 MB); nombres de variables en `.env.example`.
- [x] Verificar con MinIO temporal en Docker, un Odoo simulado y Postgres temporal: subida correcta (201, objeto en MinIO, fila y estado `PENDING`), sin cookie (401), de otro contacto (403), más de 10 MB y tipos o firmas no permitidos (400), `vehicleId` ajeno (403/404), el archivo llega a Odoo cuando existe el lead (`SYNCED` con `id_odoo_attachment`), con el lead ausente queda `PENDING` sin gastar intentos, reinicio con rehidratación, Odoo caído hasta `FAILED` e idempotencia.

### Slice 8 — Zonas y citas

**Bloque 1 — CREAR**
- [x] Esquema: índice único sobre `LOWER(name)` en `zones` (`uq_zones_name`, sin distinguir mayúsculas; antes `UNIQUE (name)`) y `UNIQUE (zone_id, appointment_date, start_time)` en `dates` (el choque de horario sale como 409 por `DataTransaction`).
- [x] Opciones validadas al arrancar: `Zones` (nombre, latitud y longitud **obligatorias**: son el punto en el mapa) y `Appointments` (`Days` 1 a 5, `OpeningTime` 09:00, `ClosingTime` 17:00, `SlotMinutes` 30, `MinDaysAhead` ≥ 1, `MaxDaysAhead` 30, zona horaria `America/Mexico_City`); `ZoneSeeder` que inserta las zonas que falten por nombre (no borra).
- [x] Servicios: disponibilidad (horarios libres por zona y día) y agendar (día hábil, dentro del horario y alineado a la cuadrícula, entre mañana y 30 días, zona existente, horario libre; `end_time` = inicio + 30 min; `confirmation = false`; nombre "Cita - {nombre}" truncado a 60); sube el funnel a la etapa de citas. **Decisión:** una sola cita futura por contacto; agendar otra la reemplaza (misma fila, 200 en lugar de 201).
- [x] Endpoints sin mapear: `GET /api/zones`, `GET /api/dates/availability?zoneId=&date=` y `POST /api/prospects/{contactId}/dates`.

**Bloque 2 — MODIFICAR**
- [ ] `dates.id_odoo_date` y las columnas de estado de Odoo en `dates` solo cuando se sincronice (pendiente); hoy queda nulo.

**Bloque 3 — ENLAZAR**
- [x] Módulos y rutas (`Relaxed` en zonas y disponibilidad; `Prospect` y `Moderate` en agendar).
- [ ] Verificar contra Postgres: las zonas del `appsettings` aparecen en `GET /api/zones` y no se duplican al reiniciar, disponibilidad correcta (fin de semana, fuera de horario, pasado y más allá de 30 días sin horarios), agendar (201), mismo horario y zona (409, también con dos peticiones simultáneas), horario no alineado, zona inexistente (404), sin cookie (401) y de otro contacto (403).

**Notas de implementación:** `AppointmentOptions` vive en `Application/Scheduling`; el validador (`SchedulingOptionsValidator`) falla al arrancar con mensajes en español; `ZoneSeeder` no impide el arranque si la BD falla; la zona horaria `America/Mexico_City` existe en la imagen `aspnet:10.0` (verificado). Pendientes menores: el 409 por carrera usa el mensaje genérico "El registro ya existe"; dos POST simultáneos del mismo contacto sin cita previa podrían crear dos filas; sincronizar citas con Odoo (`dates.id_odoo_date` y estado) y cancelar o confirmar citas quedan fuera.

## Criterios de Aceptación

- [ ] `dotnet build` en `nummex-backend-v2` con 0 errores y 0 advertencias al cerrar cada slice.
- [ ] Un uploader legacy y uno JSON-2 intercambiables con `Odoo__Api`, con el mismo comportamiento verificado.
- [ ] El catálogo y la cotización responden sin sesión, aplican el 70 % con redondeo y tope, cachean y fallan con 502 claro si el proveedor falla.
- [ ] Cada paso del funnel solo lo ejecuta el dueño de la sesión (401 sin cookie, 403 de otro contacto) y no hay lecturas expuestas.
- [ ] Los documentos se guardan en MinIO, solo llegan a Odoo cuando existe el lead y nunca se duplican.
- [ ] Las citas respetan el horario, la cuadrícula y una cita por zona y horario, incluso con peticiones simultáneas.
- [ ] Esquema al día en `db-migrations/initial.sql` y variables nuevas (solo nombres) en `.env.example`. Los tests los agrega el responsable.

## Relacionado

- [[nummex-backend-v2-plan]]
- [[nummex-backend-v2-fase-1-sub-plan]]
