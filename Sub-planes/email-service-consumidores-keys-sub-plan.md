---
tipo: sub-plan
titulo: Slice 2 — Consumidores y keys
autor: Efraín García
creado: 2026-10-27
actualizado: 2026-10-27
estado: en-progreso
proyecto: email-service
padre: "[[email-service-plan]]"
tags:
  - email-service
  - sub-plan
  - consumidores
  - keys
  - slice-2
---

# Slice 2 — Consumidores y keys

> **Plan padre:** [[email-service-plan]]
> Que el administrador cree consumidores con sus roles, genere y revoque keys y los gestione desde el panel.

## Pendientes

- [ ] Activar y desactivar un consumidor (`active`).
- [ ] Editar los roles (`capabilities`) de un consumidor.
- [ ] `GET /consumers/{id}`.
- [ ] Vencimiento de la key elegible desde la pantalla.
- [ ] Mostrar los errores de validación por campo.
- [ ] Política de rotación: nueva key y vencimiento de la anterior.
- [ ] Tope de keys activas por consumidor.

## Estado Actual

| Área | Estado |
|------|--------|
| `GET /consumers` (con keys, sin hashes) | ✅ |
| `POST /consumers` (nombre + roles, primera key en claro) | ✅ |
| `POST /consumers/{id}/keys` (con `expireAt` opcional) | ✅ |
| `POST /keys/{id}/revoke` (idempotente) | ✅ |
| Validación nativa del nombre y los roles | ✅ |
| Panel: alta de consumidor, nueva key y revocar | ✅ |
| Activar/desactivar, editar roles y detalle | ⏳ |

## Alcance

Cubre consumidores y keys y su pantalla de administración. No cubre la asignación de plantillas (Slice 3) ni de grupos (Slice 6); esas pantallas se agregan en sus slices.

## Forma de trabajo

Sigue las **Reglas de Trabajo** de [[email-service-plan]] y se construye sobre `ConsumerService`, `KeyService`, `ConsumerQueries` y sus repositorios. La **lista de consumidores va por GraphQL**; `GET /consumers/{id}` y las escrituras, por REST.

## Implementación

### Paso 1 — Base de datos
- [x] Tablas `consumer` y `keys` ya definidas.
- [ ] Nada que migrar para activar/desactivar (la columna `active` existe).

### Paso 2 — API
- [x] `ConsumerController`, `KeyController`, `Contracts` y `Builders` en `Api/Endpoints/Admin`.
- [ ] `PATCH /consumers/{id}` para `active` y roles (validados con `IsEnumList`).
- [ ] `GET /consumers/{id}` en REST y la lista de consumidores por GraphQL (paginada, con filtro por nombre y orden).
- [ ] Regla de rotación y tope de keys activas.
- [ ] Un consumidor con mensajes o lotes no se borra: se desactiva.

### Paso 3 — Panel
- [x] Formulario de consumidor con los 4 roles, listado con sus keys y revocación.
- [ ] Interruptor activar/desactivar, edición de roles y selector de vencimiento.
- [ ] Mostrar los `errors` por campo de los 400.

### Paso 4 — Documentación
- [ ] README: tabla de endpoints al día.

## Criterios de Aceptación

- [ ] Crear un consumidor devuelve la key una sola vez; el listado nunca trae hashes ni keys en claro.
- [ ] Un consumidor desactivado deja de autenticarse (401) sin tocar sus keys.
- [ ] Nombre o roles inválidos → 400 con `errors` por campo; nombre repetido → 409.
- [ ] Revocar dos veces es idempotente; una key revocada responde 401.
- [ ] El panel refleja los cambios sin recargar a mano y la key nueva se muestra una sola vez.
- [ ] `dotnet build` y `bun run build` correctos.

## Preguntas abiertas

- ¿Cuántas keys activas por consumidor como máximo?
- ¿La rotación vence la key anterior de inmediato o con un período de gracia?

## Relacionado

- [[email-service-plan]]
- [[email-service-auth-sub-plan]]
- [[email-service-plantillas-sub-plan]]
