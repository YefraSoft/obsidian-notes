---
tipo: sub-plan
titulo: "Nummex Agent — Plan de Desarrollo RAG"
autor: "Efraín García"
creado: 2026-09-10
actualizado: 2026-09-10
estado: en-progreso
proyecto: nummex
padre: "[[nummex-app-plan]]"
tags:
  - nummex
  - plan
  - sub-plan
  - agent
  - rag
  - ia
  - whatsapp
---
# Nummex Agent — Plan de Desarrollo RAG

> Servicio interno `nummex-agent` (TypeScript + Bun + Qdrant) integrado mediante `nummex-backend`
> Objetivos: Refactor SOLID → Pipeline RAG de producción → Chat web listo para WhatsApp
> Creado: 2026-09-10

---

## Pendientes

- [x] Refactorizar el agente a capas y puertos sin cambiar `GET /health`.
- [x] Implementar ingestión, recuperación, generación, transcripción y sesión del pipeline RAG.
- [x] Conectar el widget web mediante el backend y documentar la extensión futura para WhatsApp.
- [x] Generar y dejar lista para ingestar la base de conocimiento inicial con contenido real del sitio.
- [ ] Probar el flujo end-to-end con Qdrant real vía `docker compose` (bloqueado: Docker Desktop no está corriendo en esta máquina).

## Estado Actual (resumen ejecutivo)

| Área | Estado |
|------|--------|
| Endpoint disponible | `GET /health` (desglosado por dependencia), `POST /chat`, `POST /ingest` (solo si `INGEST_API_KEY` está definido) |
| Capas SOLID | ✅ Listas: `Domain/{Ports,Models,Services}`, `Infrastructure/{Http,Logging,Health,Adapters/*}`, `Application/Services`, `Presentation/Http`, `Config`, `Composition` |
| Ingestión de conocimiento | ✅ `IngestionService.ingestDocument()`: chunking → embeddings → upsert en Qdrant, vía `POST /ingest` |
| Embeddings | ✅ `OllamaEmbeddingProvider`/`GoogleEmbeddingProvider` implementados |
| Recuperación vectorial | ✅ `QdrantRepository` conectado a `ChatService` (búsqueda por similitud antes de generar respuesta) |
| Generación LLM | ✅ `OllamaLlmProvider`/`DeepSeekLlmProvider`/`GoogleLlmProvider` implementados; modelos locales confirmados: `qwen3.5:latest` (chat) / `qwen3-embedding:latest` (embeddings, 4096 dimensiones verificadas en vivo) |
| Endpoint de chat | ✅ `POST /chat` real (puerto interno 4000), probado en vivo |
| Integración web | ✅ `ChatbotBubble.astro` llama a `chatApi.send()` vía `nummex-backend` (`POST /api/chat`); probado en navegador (Chrome DevTools MCP) — abre el panel, envía mensaje, muestra "Escribiendo…" y maneja el error de red de forma controlada sin backend corriendo |
| Base de conocimiento | ✅ 9 documentos Markdown en `nummex-agent/knowledge/` con contenido real del sitio + `scripts/ingest-knowledge-base.ts` para cargarlos vía `/ingest` (pendiente de ejecutar contra un Qdrant real) |
| Integración WhatsApp | No implementada; `IChannelAdapter`/`MessageAggregator` ya construidos y testeados, listos para el adaptador de WhatsApp (Fase 3+) |
| Vector store | `QdrantRepository.ts` — corregidos 2 bugs reales (dependencia directa + `collectionExists()`), ahora también implementa `IHealthChecker` |
| Topología objetivo | Navegador → backend → agente interno (sin cambios) — implementada y verificada en el navegador |

| Fase | Estado |
|------|--------|
| FASE 1 — Refactor SOLID | ✅ Completada (23/23 tests, typecheck limpio, `/health` verificado sin cambios) |
| FASE 2 — Pipeline RAG de producción | ✅ Completada (58/58 tests, typecheck limpio, `/health` y `/chat` probados en vivo contra Ollama real) |
| FASE 3 — Web y preparación WhatsApp | ✅ Completada (código); ⏳ pendiente probar el flujo end-to-end con Qdrant real (Docker Desktop no disponible en esta máquina) |

---

## Decisiones de Diseño (vigentes)

### Proveedores de IA

- Los LLM y proveedores de embeddings se exponen mediante puertos agnósticos.
- Desarrollo usa Ollama local.
- Producción usará DeepSeek o Google; la selección permanece abierta.
- Los embeddings son configurables de forma independiente del chat, pues DeepSeek no ofrece API de embeddings.
- La transcripción usa Whisper hospedado por OpenAI o Groq, detrás de una interfaz común.

### Sesión, mensajes y extensibilidad

- El buffer e historial de conversación usan `lru-cache`, detrás de `ISessionStore`.
- El almacenamiento de sesión podrá migrar a Redis al integrar WhatsApp o escalar a varias instancias.
- El pipeline propio reemplaza n8n: adaptador de canal → transcripción → agregador con debounce por conversación → RAG.
- `IChannelAdapter` debe permitir añadir WhatsApp sin modificar `ChatService`, `MessageAggregator` ni `IngestionService`.

### Topología y seguridad

- Flujo: navegador → `nummex-backend` (`POST /api/chat`) → `nummex-agent` (`POST /chat`).
- El agente es un servicio interno: no expone puertos al host.
- `Caddyfile` no se modifica; el backend conserva la frontera pública.
- El endpoint del backend usa `PublicWritePolicy`, sin `[Authorize]`.

---

## Arquitectura y Notas Técnicas

- **Runtime:** TypeScript + Bun + Express.
- **Vector store:** Qdrant, con colección, dimensión y distancia configuradas por entorno; no se hardcodean `1536` ni `Cosine`.
- **Composición:** `ProviderFactory` selecciona adaptadores por entorno y `container.ts` actúa como composition root.
- **Observabilidad:** logging estructurado y middleware global para errores y registro de requests.
- **Salud:** `GET /health` agrega `backend`/`vectorStore`/`llm` por separado (200 solo si todas son `true`); `HealthCheckerService` (Fase 1) queda como utilidad genérica testeada, sin uso en producción por ahora.
- **Despliegue:** Dockerfile multi-stage con `HEALTHCHECK`; Compose incluye Qdrant y el agente interno.
- **Patrones existentes a reutilizar:** `ExpressApp.addRoute()`, `EnvLoader.ts` con mensajes en español, módulos y clientes HTTP del backend .NET, `nummex-web/src/lib/loans/api.ts` y `ChatbotBubble.astro`.

---

## Archivos Clave del Proyecto

| Archivo o ruta | Propósito |
|----------------|-----------|
| `nummex-agent/src/Presentation/Http/ExpressApp.ts` | Entrada HTTP; registra health/chat/ingest vía `addRoute()` |
| `nummex-agent/src/Application/Services/ChatService.ts` | Orquesta retrieval + generación + memoria de conversación |
| `nummex-agent/src/Composition/{container,ProviderFactory}.ts` | Composition root y selección de adaptadores por entorno |
| `nummex-agent/src/Infrastructure/Adapters/VectorStore/QdrantRepository.ts` | Vector store real, ya implementa `IHealthChecker` |
| `nummex-agent/src/Config/EnvLoader.ts` | Validación de configuración (Fase 1 + Fase 2) |
| `nummex-agent/docker-compose.yml` (raíz del monorepo) | Servicios `qdrant` y `agent` definidos, con modelos qwen y `EMBEDDING_VECTOR_SIZE=4096` |
| `nummex-agent/knowledge/*.md` | 9 documentos con el contenido real del sitio, listos para `/ingest` |
| `nummex-agent/scripts/ingest-knowledge-base.ts` | Script Bun que ingesta todos los `.md` de `knowledge/` vía `POST /ingest` |
| `nummex-backend/Endpoints/Chat/` | ✅ Módulo y cliente que exponen `POST /api/chat` (patrón `OdooClient`/`LoanQuotesController`) |
| `nummex-web/src/components/chatbot/ChatbotBubble.astro` | ✅ Widget conectado: envía mensajes, muestra "Escribiendo…" y maneja errores |
| `nummex-web/src/lib/chat/` | ✅ `api.ts` (cliente HTTP), `conversationId.ts` (persistencia en `localStorage`), `types.ts` (`ChatApiError`) |

---

## FASE 1 — Refactor a SOLID

> **Objetivo:** Separar el agente en capas, puertos y adaptadores sin cambiar el comportamiento observable de `GET /health`.
> **Estado: ✅ Completada y verificada.**

### 1.1 Estructura y responsabilidades

- [x] Crear `Domain/Ports`, `Domain/Models`, `Application/Services`, `Config` y `Composition` bajo `src/`.
- [x] Crear `Infrastructure/Http` e `Infrastructure/Logging` (las subcarpetas `Adapters/{Llm,Embeddings,Transcription,Session,Channels}` se difieren a Fase 2 a propósito, junto con su primer archivo real, para no dejar carpetas vacías).
- [x] Crear `Presentation/Http/{routes,middlewares}`.
- [x] Mover `ExpressApp.ts` a `Presentation/Http/`, `BackendService.ts` a `Infrastructure/Http/` y `HealthCheckerService.ts` a `Application/Services/` (con `git mv`, historial preservado).
- [x] Definir `ILLMProvider`, `IEmbeddingProvider`, `IVectorStoreRepository`, `ITranscriptionService`, `ISessionStore`, `IChannelAdapter` e `IHealthChecker` en `Domain/Ports/`, más los modelos `ChatMessage`, `InboundMessage`, `RetrievedChunk`, `HealthCheckDto`, `VectorRecord`.

### 1.2 Dependencias y configuración

- [x] Sustituir `QdrantService.ts` por `Infrastructure/Adapters/VectorStore/QdrantRepository.ts`, implementando `IVectorStoreRepository`. De paso se corrigió un segundo bug real descubierto al reescribirlo: `collectionExists()` devuelve `{exists: boolean}`, no un booleano directo — el código original nunca detectaba bien si la colección ya existía.
- [x] Quitar `qdrant` de `package.json` y declarar `@qdrant/js-client-rest@^1.18.0` como dependencia directa.
- [x] Quitar `cors`, que no se usa.
- [x] Eliminar `langchain`, `@langchain/core` y `@langchain/qdrant` — decisión tomada: eliminación completa.
- [x] Extender `EnvLoader.ts` con `LLM_PROVIDER`, `EMBEDDING_PROVIDER`, `TRANSCRIPTION_PROVIDER`, `QDRANT_URL`, `QDRANT_COLLECTION`, `SESSION_TTL_SECONDS` y `AGGREGATOR_DEBOUNCE_MS` — todos **opcionales** por ahora (validados si están presentes) para no romper ningún `.env` existente; las variables específicas por proveedor de Ollama/DeepSeek/Google quedan para Fase 2.
- [x] `QdrantRepository` ya recibe tamaño y distancia de vector por config (constructor), no hardcodeados.

### 1.3 Composición, errores y validación

- [x] Crear `Composition/container.ts` como composition root (reemplaza el wiring implícito de `index.ts`; de paso `HealthCheckerService`, antes código muerto, ahora sí se usa). `ProviderFactory.ts` se difiere a Fase 2 — hoy no hay ningún proveedor concreto entre el cual elegir.
- [x] Añadir `ILogger`/`ConsoleLogger` (logging estructurado JSON a stdout, sin nueva dependencia).
- [x] Añadir `errorHandler.ts` y `requestLogger.ts` como middleware global en `ExpressApp`.
- [x] Ejecutar `bun install`, `bun run typecheck`, `bun test` y `bun run dev` — **23/23 tests pasan**, typecheck limpio.
- [x] Verificar que `GET /health` conserva su comportamiento actual — confirmado: `503 {"backend":false}` sin backend disponible, mismo header `Cache-Control: no-store`.

---

## FASE 2 — Pipeline RAG de producción

> **Objetivo:** Implementar la ingestión, recuperación y respuesta conversacional con proveedores intercambiables.
> **Estado: ✅ Completada y verificada en vivo.**

**Verificado con servicios reales:** con Ollama corriendo localmente (`llama3.1:8b`), `GET /health` respondió `{"backend":false,"vectorStore":false,"llm":true}` (backend y Qdrant apagados a propósito para la prueba; Ollama detectado correctamente vía `GET /api/tags`) y `POST /chat` devolvió un 500 controlado y bien logueado al faltar el modelo de embeddings — confirma manejo de errores correcto sin fugar detalles internos al cliente, y que el servicio no se cae si Qdrant no está disponible al arrancar.

**Dos ajustes no previstos en la planificación original, hechos durante la implementación:**
1. `GET /health` pasó de un único booleano agregado a un desglose por dependencia (`{backend, vectorStore, llm}`, 200 solo si todas son `true`) — con un solo booleano no había forma de saber cuál dependencia falló. `HealthCheckerService` (Fase 1) queda sin uso en `container.ts` pero se conserva como utilidad genérica ya testeada.
2. `ensureCollection()` de Qdrant se envuelve en try/catch al arrancar: si Qdrant no está listo todavía, se loguea un warning y el servicio sigue arrancando en vez de crashear (`depends_on` de Compose no garantiza que Qdrant ya acepte conexiones, solo que el contenedor inició); `/health` refleja el estado real en cada consulta.

**Decisiones tomadas al planificar esta fase:**
- Puerto interno del agente en Docker: **4000**, sin `ports:` expuestos al host.
- `EnvLoader.ts` sigue siendo solo parseo/validación de formato (se le suman los campos por proveedor, todos opcionales); la regla "si elegiste el proveedor X, faltan sus variables" vive en `Composition/ProviderFactory.ts` (nuevo), que lanza el error descriptivo en español al construir el adaptador.
- Validación de payload de `/chat` y `/ingest`: manual, mismo estilo que `EnvLoader.ts` — sin sumar `zod` (se mantiene la disciplina de no añadir dependencias salvo la ya acordada `lru-cache`).
- `POST /ingest` protegido con header compartido simple `X-Ingest-Key` contra `INGEST_API_KEY` — no se justifica un sistema de auth completo para un endpoint interno de un solo operador.
- Rate limiting propio en el agente: **no se implementa** (agente interno, sin puerto expuesto; ya cubierto por `PublicWritePolicy` en el borde público del backend).
- Health check de LLM: ping real solo para Ollama (`GET {OLLAMA_BASE_URL}/api/tags`); para DeepSeek/Google es un checker "siempre saludable" para no gastar cuota de API en cada `/health` — decisión deliberada.

### 2.1 Adaptadores de IA y sesión

| Adaptador | Puerto | Detalle |
|---|---|---|
| `Llm/OllamaLlmProvider.ts` | `ILLMProvider` | `POST {OLLAMA_BASE_URL}/api/chat` → `message.content` |
| `Llm/DeepSeekLlmProvider.ts` | `ILLMProvider` | `POST https://api.deepseek.com/chat/completions` (compatible OpenAI) |
| `Llm/GoogleLlmProvider.ts` | `ILLMProvider` | `POST .../models/{modelo}:generateContent?key=...` (fetch directo, sin SDK nuevo) |
| `Embeddings/OllamaEmbeddingProvider.ts` | `IEmbeddingProvider` | `POST {OLLAMA_BASE_URL}/api/embeddings` |
| `Embeddings/GoogleEmbeddingProvider.ts` | `IEmbeddingProvider` | `POST .../models/{modelo}:embedContent?key=...` |
| `Transcription/OpenAiCompatibleWhisperAdapter.ts` | `ITranscriptionService` | Clase base parametrizada `{baseUrl, apiKey, model}`; `OpenAiWhisperAdapter`/`GroqWhisperAdapter` solo cambian esos parámetros |
| `Session/LruSessionStore.ts` | `ISessionStore` | Envuelve `lru-cache` (nueva dependencia), `max`/`ttl` desde `sessionTtlSeconds` |

- [x] Implementar `OllamaLlmProvider`, `DeepSeekLlmProvider` y `GoogleLlmProvider`.
- [x] Implementar `OllamaEmbeddingProvider` y `GoogleEmbeddingProvider`.
- [x] Implementar `OpenAiCompatibleWhisperAdapter` (base) + `createOpenAiWhisperAdapter`/`createGroqWhisperAdapter`.
- [x] Implementar `LruSessionStore` mediante la nueva dependencia `lru-cache`.
- [x] Formalizar que `QdrantRepository` implementa también `IHealthChecker`.
- [x] Extender `EnvLoader.ts` (opcionales): `OLLAMA_BASE_URL`, `OLLAMA_CHAT_MODEL`, `OLLAMA_EMBEDDING_MODEL`, `DEEPSEEK_API_KEY`, `DEEPSEEK_MODEL`, `GOOGLE_API_KEY`, `GOOGLE_CHAT_MODEL`, `GOOGLE_EMBEDDING_MODEL`, `OPENAI_API_KEY`, `GROQ_API_KEY`, `EMBEDDING_VECTOR_SIZE`, `INGEST_API_KEY`.
- [x] Crear `Composition/ProviderFactory.ts`: `createLlmProvider`, `createEmbeddingProvider`, `createTranscriptionService`, `createVectorStoreRepository`, `createSessionStore`, `createLlmHealthChecker` — cada una valida sus variables específicas y lanza error descriptivo si faltan.

### 2.2 Ingestión y conversación

- [x] Implementar `Domain/Services/chunkText.ts` (función pura, testeable) y `IngestionService.ingestDocument()`: chunking → `embedBatch` → upsert en Qdrant.
- [x] Exponer `POST /ingest` administrativo, protegido por `X-Ingest-Key`, no público (solo se monta si `INGEST_API_KEY` está definido).
- [x] Implementar `Domain/Services/buildPrompt.ts` + `Domain/Services/mergeContext.ts` (funciones puras) y `ChatService.handleMessage()`: historial → embedding de consulta → búsqueda Qdrant → LLM (con contexto inyectado por cada adaptador) → guardado de turno → `{ reply }`.
- [x] Implementar `WebChannelAdapter` para normalizar el body de `/chat` a `InboundMessage`.
- [x] Implementar `MessageAggregator`: audio a texto por transcripción y debounce por `conversationId`. Construido y testeado (4 tests), pero **no wireado** en `container.ts` — el canal web llama a `ChatService` directo; queda listo para cuando WhatsApp traiga ráfagas reales.
- [x] Exponer `POST /chat`: `{conversationId, message, type?, audioBase64?}` → `{reply, conversationId}` (audio devuelve 400 en el canal web por ahora).

### 2.3 Salud, despliegue y pruebas

- [x] `GET /health` ahora agrega `backend`, `vectorStore` (Qdrant) y `llm` (real solo para Ollama vía `/api/tags`; "siempre saludable" para DeepSeek/Google para no gastar cuota).
- [x] Dockerfile multi-stage (`deps` + `runtime`) con `HEALTHCHECK` contra `/health`, y `.env.example` con todas las variables nuevas.
- [x] `docker-compose.yml` (raíz): servicio `qdrant` (`qdrant/qdrant:latest`, volumen `qdrant_storage`) y servicio interno `agent` (`PORT=4000`, sin `ports:` expuestos al host, `extra_hosts: host.docker.internal:host-gateway` para alcanzar el Ollama del host en Linux) — validado con `docker compose config`.
- [x] Tests: `chunkText`, `buildPrompt`, `mergeContext`, `MessageAggregator`, `ChatService`, `IngestionService`, `LruSessionStore`, `ProviderFactory`, `EnvLoader` extendido — **58/58 pasan**.
- [x] Validado: `bun test`, `bun run typecheck`, y `POST /chat`/`GET /health` en vivo contra Ollama real (Qdrant no se pudo levantar en esta sesión por no tener Docker Desktop corriendo — pendiente probar el flujo completo con Qdrant arriba).

---

## FASE 3 — Integración web y preparación para WhatsApp

> **Objetivo:** Exponer el chat a la web a través del backend, cargar una base de conocimiento real del sitio, y dejar la arquitectura preparada para WhatsApp sin implementar todavía ese canal.
> **Estado: ✅ Código completado y verificado (tests, typecheck, build .NET, navegador). ⏳ Pendiente únicamente la prueba end-to-end contra un Qdrant real — bloqueada porque Docker Desktop no está corriendo en esta máquina.**

**Modelos locales confirmados** (reemplazan la suposición inicial de `llama3.1:8b`/`nomic-embed-text`): `OLLAMA_CHAT_MODEL=qwen3.5:latest`, `OLLAMA_EMBEDDING_MODEL=qwen3-embedding:latest`. Se verificó en vivo contra el Ollama local que `qwen3-embedding:latest` devuelve vectores de **4096 dimensiones** → `EMBEDDING_VECTOR_SIZE=4096` en `.env.example` y en el `docker-compose.yml` raíz.

### 3.0 Base de conocimiento inicial (contenido real del sitio → `/ingest`)

Se extrajo el contenido real y completo de `nummex-web` (textos en `src/text/es/*.ts`, páginas `.astro`) para construir 9 documentos Markdown en `nummex-agent/knowledge/`. Se encontraron y resolvieron **dos contradicciones reales** entre el FAQ público y el simulador de crédito en producción:
- **Tasa**: se usó la del simulador real — **32% anual (37.12% con IVA)** — no la del FAQ (1.9%–2.9% mensual, desactualizada).
- **Plazos**: se usaron los del formulario real — **12, 24, 36 o 48 meses** — no los del FAQ (12/18/24/30).
- El aviso de privacidad se incluyó tal cual, incluyendo la nota interna con la razón social entre corchetes sin confirmar — decisión explícita del usuario, quien rechazó la recomendación de excluirlo.

- [x] `empresa.md`, `producto-prestamo.md`, `requisitos-documentos.md`, `proceso-solicitud.md`, `condiciones-pago.md`, `preguntas-frecuentes.md` (con tasa/plazos corregidos y nota de la corrección), `inversion-en-nummex.md`, `contacto.md`, `aviso-privacidad.md`.
- [x] `nummex-agent/scripts/ingest-knowledge-base.ts` — script Bun que lee todos los `.md` de `knowledge/` y hace `POST {AGENT_URL}/ingest` por cada uno (`{text, metadata: {source:"web", topic}}`) con `X-Ingest-Key`; expuesto también como `bun run ingest:knowledge`.
- [ ] Ejecutar el script contra un agente + Qdrant reales y validar con preguntas de prueba ("¿Qué documentos necesito?", "¿Cuál es la tasa de interés?") — pendiente de Qdrant real.

### 3.1 Backend y frontend web

- [x] Crear `Endpoints/Chat/{ChatModule.cs, ChatAgentOptions.cs, IChatAgentClient.cs, ChatAgentClient.cs, ChatAgentException.cs, Dto/ChatDtos.cs, ChatController.cs}` en el backend, siguiendo el patrón exacto de `Common/Odoo/OdooClient.cs` + `Endpoints/LoanQuotes/`.
- [x] Exponer `POST /api/chat` con `[Route("api/chat")]`, `PublicWritePolicy` y sin `[Authorize]` — igual que `LoanQuotesController`. `dotnet build` compila sin errores ni warnings.
- [x] Crear `nummex-web/src/lib/chat/api.ts`, siguiendo el patrón de `loans/api.ts` (`requestJson` + `ChatApiError`).
- [x] Crear `nummex-web/src/lib/chat/conversationId.ts` con `crypto.randomUUID()` persistido en `localStorage` (con `try/catch` por si el storage no está disponible).
- [x] Modificar `ChatbotBubble.astro` para llamar `chatApi.send()`, mostrar "Escribiendo…" (clase `--pending`), pintar la respuesta y manejar errores con el texto de fallback `chatbot.errorFallback`.
- [x] Añadir textos nuevos (`typing`, `errorFallback`) en `nummex-web/src/text/es/chatbot.ts`.
- [x] Verificado en navegador real (Chrome DevTools MCP, `astro dev` en `localhost:4321`): el bubble abre, el mensaje del usuario se pinta, aparece "Escribiendo…", y al fallar la petición (sin backend corriendo) se reemplaza por el mensaje de error controlado — sin romper el input ni dejarlo bloqueado.

### 3.2 Contenedores y validación integrada

- [x] Corregir `frontend-build.build.context` de `./web-app-nummex` a `./nummex-web` en `docker-compose.yml` (bug preexistente, el directorio real es `nummex-web`).
- [x] Añadir `depends_on: agent` (con `condition: service_started`) y `ChatAgent__BaseUrl: http://agent:4000` al servicio `backend`.
- [x] Actualizar el servicio `agent` en `docker-compose.yml`: `OLLAMA_CHAT_MODEL=qwen3.5:latest`, `OLLAMA_EMBEDDING_MODEL=qwen3-embedding:latest`, `EMBEDDING_VECTOR_SIZE=4096`, `INGEST_API_KEY=local-dev-ingest-key` (para poder correr el script de ingesta en dev).
- [x] Confirmado: `Caddyfile` no requiere cambios — el navegador solo pide estáticos a Caddy y hace `fetch` directo a `backend:8080`.
- [x] `docker compose config --quiet` valida sin errores tras todos los cambios.
- [x] `dotnet build` compila limpio; `bun test` (58/58) y `bun run typecheck` (agente) siguen en verde tras los cambios.
- [ ] `docker compose up -d --build` end-to-end con Qdrant real — **bloqueado**: Docker Desktop no está corriendo en esta máquina. Pendiente para cuando esté disponible.

### 3.3 Preparación documentada para WhatsApp

- [x] Confirmado: `WhatsAppChannelAdapter` puede añadirse implementando `IChannelAdapter` sin modificar `ChatService`, `MessageAggregator` ni `IngestionService` (Open/Closed, ya verificado por diseño desde la Fase 1/2).
- [x] Documentadas variables de Meta Cloud API: `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_VERIFY_TOKEN`.
- [x] Documentada como alternativa Twilio: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` (con verificación de firma).
- [x] Anotado el futuro `POST /webhooks/whatsapp` + el challenge `GET` de Meta como próximo paso, no implementado todavía.

---

## Decisiones Abiertas para la Implementación

| Decisión | Opciones | Estado |
|----------|----------|--------|
| Uso de LangChain | Eliminar por completo (recomendado) o conservarlo solo para chunking | ✅ Resuelto — eliminado por completo en Fase 1 |
| Puerto interno del agente | Definir puerto para `ChatAgent__BaseUrl` en Compose | ✅ Resuelto — `4000` |
| Rate limiting en el agente | Rate limiting propio o confiar solo en `PublicWritePolicy` del backend | ✅ Resuelto — no se implementa, agente interno |
| Dimensión del vector en Qdrant (dev) | `qwen3-embedding:latest` vs. otros modelos | ✅ Resuelto para dev/local — `EMBEDDING_VECTOR_SIZE=4096`, verificado en vivo contra el Ollama local |
| Modelo de embeddings de producción | Ollama (dev, ya definido) o Google (`text-embedding-004` = 768) | ⏳ Pendiente — depende de la decisión del cliente sobre el proveedor de producción; `QdrantRepository` ya toma la dimensión por configuración, así que cambiarlo no requiere tocar código |

---

## Dependencias y Orden de Ejecución

```
FASE 1 (Refactor SOLID) — ✅ Completada
    │
    ├── 1.1: Capas, puertos y responsabilidades
    ├── 1.2: Dependencias, Qdrant y configuración
    └── 1.3: Composición, observabilidad y compatibilidad de /health
         │
         ▼
FASE 2 (Pipeline RAG) — ✅ Completada
    │
    ├── 2.1: LLM, embeddings, Whisper y sesión
    ├── 2.2: Ingestión, recuperación, chat y agregación
    └── 2.3: Salud, Docker Compose y pruebas
         │
         ▼
FASE 3 (Web + WhatsApp) — ✅ Código completado; ⏳ e2e con Docker pendiente
    │
    ├── 3.0: Base de conocimiento inicial (9 docs + script de ingesta)
    ├── 3.1: Endpoint público a través del backend y widget web
    ├── 3.2: Contenedores y validación end-to-end
    └── 3.3: Contrato y configuración futura de WhatsApp
         │
         ▼
Pendiente (bloqueado por entorno): levantar Docker Desktop, `docker compose up -d --build`,
ejecutar `bun run ingest:knowledge` y probar el chat real de punta a punta.
```
