
# Plan — nummex-agent: SOLID, producción e integración con la web (listo para WhatsApp)

## Contexto

`nummex-agent` conectará un RAG con la web (y luego WhatsApp) del monorepo `monolito-numex`. Hoy solo existe `GET /health`; no hay pipeline de ingestión, embeddings, recuperación, generación, ni endpoint de chat. `QdrantService.ts` es código muerto con un bug real de dependencias (`@qdrant/js-client-rest` no declarado, solo llega transitivo vía `@langchain/qdrant`).

El plan original usaba **n8n** para transcribir audios y concatenar ráfagas de mensajes (`["hola","buenas","tardes"]` → `"hola buenas tardes"`). Se reemplaza por un **pipeline propio dentro de `nummex-agent`**: adaptador de canal → transcripción (Whisper hospedado) → agregador con debounce por conversación → RAG. Sin infraestructura externa, menos saltos de red, versionado junto al código.

**Decisiones confirmadas:**
- LLM/embeddings: puertos agnósticos. Dev = Ollama (local). Prod = DeepSeek o Google (sin decidir). Embeddings configurables independientes del chat (DeepSeek no tiene API de embeddings).
- Transcripción: Whisper hospedado (OpenAI o Groq) detrás de interfaz.
- Memoria/buffer de conversación: `lru-cache` (equivalente TS a Caffeine) detrás de `ISessionStore`, migrable a Redis cuando entre WhatsApp/escale a varias instancias.
- Topología: navegador → `nummex-backend` (`POST /api/chat`) → `nummex-agent` (`POST /chat`, interno, sin puerto expuesto). `Caddyfile` no se toca.

**Patrones a reutilizar:** `ExpressApp.addRoute()`, `EnvLoader.ts` (validación en español), patrón `Endpoints/<Feature>/Module.cs+Controller.cs` con `IHttpClientFactory`+`IOptions<T>` (ver `Common/Odoo/OdooClient.cs`), patrón `nummex-web/src/lib/loans/api.ts`, widget `ChatbotBubble.astro` (script líneas 71-84, aún sin llamar API).

---

## Fase 1 — Refactor a SOLID (sin cambiar comportamiento observable)

- [ ] Crear estructura de capas en `src/`: `Domain/Ports`, `Domain/Models`, `Infrastructure/Adapters/{Llm,Embeddings,VectorStore,Transcription,Session,Channels}`, `Infrastructure/Http`, `Infrastructure/Logging`, `Application/Services`, `Presentation/Http/{routes,middlewares}`, `Config`, `Composition`
- [ ] Mover `ExpressApp.ts` → `Presentation/Http/`; `BackendService.ts` → `Infrastructure/Http/`; `HealthCheckerService.ts` → `Application/Services/`
- [ ] Reescribir `QdrantService.ts` → `Infrastructure/Adapters/VectorStore/QdrantRepository.ts` implementando `IVectorStoreRepository`, parametrizando tamaño de vector/distancia desde env (no hardcodear 1536/Cosine)
- [ ] Quitar `"qdrant"` de `package.json`; declarar `@qdrant/js-client-rest` como dependencia directa
- [ ] Quitar `cors` (nunca usado)
- [ ] Decidir y (recomendado) eliminar `langchain`, `@langchain/core`, `@langchain/qdrant`
- [ ] Definir puertos en `Domain/Ports/`: `ILLMProvider`, `IEmbeddingProvider`, `IVectorStoreRepository`, `ITranscriptionService`, `ISessionStore`, `IChannelAdapter`, `IHealthChecker`
- [ ] Extender `EnvLoader.ts` (mismo estilo, mensajes en español): `LLM_PROVIDER`, `EMBEDDING_PROVIDER`, vars de Ollama/DeepSeek/Google, `QDRANT_URL`, `QDRANT_COLLECTION`, `TRANSCRIPTION_PROVIDER` + API key, `SESSION_TTL_SECONDS`, `AGGREGATOR_DEBOUNCE_MS`
- [ ] Crear `Composition/ProviderFactory.ts` (switch por env → adaptador) y `Composition/container.ts` (composition root)
- [ ] Añadir `ILogger` + adaptador de logging estructurado
- [ ] Añadir middleware global de errores (`errorHandler.ts`) y de request logging (`requestLogger.ts`) en `ExpressApp`
- [ ] Verificar: `bun install && bun run typecheck && bun test && bun run dev` — `GET /health` debe comportarse igual que antes

## Fase 2 — Completar para producción (pipeline RAG real)

- [ ] Implementar `OllamaLlmProvider`, `DeepSeekLlmProvider`, `GoogleLlmProvider`
- [ ] Implementar `OllamaEmbeddingProvider`, `GoogleEmbeddingProvider`
- [ ] Implementar `OpenAiWhisperAdapter`/`GroqWhisperAdapter` (base común OpenAI-compatible)
- [ ] Implementar `LruSessionStore` (nueva dep `lru-cache`)
- [ ] Implementar `IngestionService.ingestDocument()` (chunking testeable → embedBatch → upsert Qdrant)
- [ ] Endpoint admin `POST /ingest` (no público) para cargar base de conocimiento
- [ ] Implementar `ChatService.handleMessage()`: historial → embed consulta → búsqueda Qdrant → `buildPrompt` → LLM → guardar turno → `{ reply }`
- [ ] Implementar `WebChannelAdapter` (normaliza body de `/chat` a `InboundMessage`)
- [ ] Implementar `MessageAggregator` (audio→texto vía transcripción + debounce por `conversationId`, listo para WhatsApp aunque el canal web lo salte)
- [ ] Endpoint `POST /chat` vía `ExpressApp.addRoute()`: `{conversationId, message, type?, audioBase64?}` → `{reply, conversationId}`
- [ ] `QdrantHealthChecker` y `LlmProviderHealthChecker` añadidos a `HealthCheckerService`
- [ ] Dockerfile multi-stage con `HEALTHCHECK`
- [ ] Crear `.env.example` con todas las variables nuevas
- [ ] Tests: `chunkText`, `buildPrompt`, `MessageAggregator`, `ChatService` (mocks manuales), `LruSessionStore`, extender `EnvLoader.test.ts`
- [ ] `docker-compose.yml`: añadir servicio `qdrant` (+ volumen `qdrant_storage`) y servicio `agent` (sin `ports:` expuestos al host)
- [ ] Verificar: `bun test`, `curl POST /chat`, `curl /health` (con Qdrant + LLM), `docker compose up -d agent`

## Fase 3 — Conectar con la web y dejar listo para WhatsApp

- [ ] Backend: crear `Endpoints/Chat/ChatModule.cs`, `ChatAgentOptions.cs`, `IChatAgentClient`/`ChatAgentClient.cs`, `Dto/ChatDtos.cs`, `ChatController.cs` (`[Route("api/chat")]`, `PublicWritePolicy`, sin `[Authorize]`)
- [ ] Frontend: crear `src/lib/chat/api.ts` (patrón `loans/api.ts`)
- [ ] Frontend: crear `src/lib/chat/conversationId.ts` (`crypto.randomUUID()` en `localStorage`)
- [ ] Modificar `ChatbotBubble.astro` (líneas 71-84): llamar `chatApi.send()`, estado "escribiendo…", pintar respuesta, manejar errores
- [ ] Añadir textos nuevos a `src/text/es/chatbot.ts`
- [ ] Fix `docker-compose.yml`: `frontend-build.build.context` de `./web-app-nummex` → `./nummex-web`
- [ ] `docker-compose.yml`: `depends_on: agent` + `ChatAgent__BaseUrl` en `backend`
- [ ] Confirmar `Caddyfile` sin cambios (decisión documentada, no olvido)
- [ ] Verificar: `dotnet build`, `curl POST /api/chat`, `docker compose up -d --build` + `docker compose ps`, prueba manual del widget en el navegador (Network tab)

### Preparación para WhatsApp (documentar, no implementar aún)
- [ ] Confirmar que `IChannelAdapter` permite añadir `WhatsAppChannelAdapter` sin tocar `ChatService`/`MessageAggregator`/`IngestionService`
- [ ] Documentar variables necesarias: Meta Cloud API (`WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_VERIFY_TOKEN`) o Twilio (`TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`)
- [ ] Dejar anotado el futuro `POST /webhooks/whatsapp` + `GET` challenge de Meta

## Decisiones abiertas para el momento de implementar

- [ ] ¿Eliminar LangChain por completo o conservar parcialmente para chunking?
- [ ] Dimensión real del vector en Qdrant según modelo de embeddings de producción (Ollama/Google = 768)
- [ ] Puerto interno de `nummex-agent` en `docker-compose.yml` (para `ChatAgent__BaseUrl`)
- [ ] ¿Rate limiting propio dentro del agente o solo confiar en `PublicWritePolicy` del backend?
