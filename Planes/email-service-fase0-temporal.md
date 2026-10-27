---
tipo: plan
titulo: Fase 0 temporal — Dominios Consumer y Manager
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: en-progreso
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - plan
  - email-service
  - temporal
  - auth
  - consumer
  - manager
---

# Fase 0 temporal — Dominios Consumer y Manager

> **Plan padre:** [[email-service-plan]]. Plan **temporal**: separa la identidad en dos dominios (Consumer y Manager), cifra la key con su dominio y deja una sola key vigente por dueño. Al terminar se borra de la bóveda y queda en el historial de git. Después se sigue con el panel y el cierre de la Fase 1.

## Decisiones

- **Tres identidades:** *Admin* (key de OpenBao, sin cambios), *Consumer* (app, backend a backend) y *Manager* (persona con email que administra contactos; sin contraseña, usa su key).
- **Dominios separados:** tablas `consumer` y `manager`, cada una con su tabla de keys (`consumer_keys`, `manager_keys`, FK propia con `ON DELETE CASCADE`).
- **Roles por dominio (sin mezcla):** Consumer → `Sender`, `GroupSender`. Manager → `GroupManager`, `TemplateManager`. Se validan en el servicio (400 con `errors`) y el handler solo emite los roles válidos del dominio.
- **Una sola key vigente por dueño.** Rotar es reemplazar: la anterior se revoca en el acto y el consumidor se actualiza por su cuenta. `expire_at` se mantiene opcional (key temporal para una campaña). Sin tope de keys ni período de gracia. Índice único parcial `WHERE revoked_at IS NULL`.
- **La key viaja cifrada con su dominio:** token opaco con cifrado autenticado (AES-256-GCM, nonce aleatorio, tag que detecta alteraciones, prefijo de versión). Lleva dominio, id de la key e id del dueño: se resuelve la tabla con **una sola consulta**. Después de descifrar siempre se consulta la base (revocada, vencida, dueño inactivo) y se compara el hash. La clave de cifrado es un secreto nuevo en OpenBao (detalle solo en la wiki).
- **Rutas:** `POST /consumers` y `POST /managers` crean al dueño y generan su primera key (se muestra una vez). `POST /consumers/{id}/key` y `POST /managers/{id}/key` solo rotan; `POST …/key/revoke` revoca. GET por id y `PATCH` (`active`, roles) por dominio. Las listas van por GraphQL (`consumers`, `managers`) con `filter` y `orderBy`.
- El administrador entrega la key; no hay login por email ni correos en esta fase.
- Las asignaciones de grupos y plantillas (Slices 3 y 6) se enlazarán al dueño que corresponda.

## Orden

1. [x] **Documentación:** este plan, plan maestro, sub-planes, `AGENTS.md`, README y wiki al día con el nuevo enfoque.
2. [x] **`infra/db.sql`:** cambios directos (sin `ALTER`).
3. [x] **Base:** `make down-volumes`, `make up` (recarga el esquema) y entidades regeneradas con el scaffold de EF. Pendiente: el secreto de cifrado con `make init` (paso A2).
4. [ ] **Implementación**, por bloques CREAR → MODIFICAR → ENLAZAR con sub-agentes.

## Implementación

### A. CREAR (aditivo)
- [ ] A1 — Core y contratos: `KeyOwnerKind`, `AllowedRoles`, tipo de `Actor`, `IKeyProtector`, contratos de Manager y de rotación, validación de roles por dominio.
- [ ] A2 — `AesGcmKeyProtector` y el secreto de cifrado en las opciones (`make init` lo genera la primera vez).
- [ ] A3 — Puertos y repositorios por dominio; `ManagerService`, `ManagerQueries`, builder y servicios de keys con `RotateAsync`.
- [ ] A4 — Tipos GraphQL de managers, `filter` y `orderBy`.

### B. MODIFICAR
- [ ] B1 — Entidades regeneradas, `EmailDbContext`, `ApiKeyRepository` y `Actor.From` con dominio.
- [ ] B2 — Handler con la key cifrada y claim de dominio; `ConsumerService` valida roles contra el dominio.

### C. ENLAZAR
- [ ] C1 — DI y rutas: `ManagerController`, `PATCH`, rotar y revocar por dominio; retiro de `POST consumers/{id}/keys` y `POST keys/{id}/revoke`.
- [ ] C2 — GraphQL: `consumers` y `managers` con `filter` y `orderBy`.
- [ ] C3 — Verificación con curl: key alterada o de otra clave → 401 sin tocar la base; rotar (la vieja → 401 al instante); vencida, inactivo, rol de otro dominio.
- [ ] C4 — Documentación final (wiki, README, `AGENTS.md`).

## Criterios de Aceptación

- [ ] Cada dominio tiene sus tablas y sus keys; no se mezclan roles entre dominios.
- [ ] Una key alterada se rechaza con 401 sin consultar la base; una rotada deja de funcionar al instante.
- [ ] `dotnet build` con 0 errores y 0 advertencias en cada paso.

## Relacionado

- [[email-service-plan]]
- [[email-service-auth-sub-plan]]
- [[email-service-consumidores-keys-sub-plan]]
