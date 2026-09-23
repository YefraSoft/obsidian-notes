---
tipo: plan
titulo: "Plan por fases - App RH Cotla"
autor: "Efraín García"
creado: 2026-08-25
actualizado: 2026-09-11
estado: draft
proyecto: rh-cotla
tags:
  - rh-cotla
  - plan
  - mvp
---
# Plan por fases - App RH Cotla

## Objetivo
Entregar un MVP de RH para Ferretería Cotla que cubra expediente de colaboradores, asistencia con selfie y GPS, solicitudes de ausencia, horas extra, alertas y reportes para RH y gerentes.

## Fuente funcional
- Documento original: `docs/01-especificacion-app-rh.md`
- Documento aprobado: versión consolidada recibida el 2026-08-25
- Decisión operativa: usar la versión aprobada como fuente de verdad cuando difiera de la original.

## Fase 0 - Aterrizaje funcional y técnico
### Entregables
- Especificación consolidada y diferencias entre versión original y aprobada.
- Modelo de datos PostgreSQL.
- Catálogo de roles, estados y reglas base del MVP.

### Decisiones cerradas
- 1 tienda en MVP, con modelo preparado para crecer.
- 3 roles: colaborador, gerente y RH.
- Gerente con visibilidad por departamento.
- Incapacidad con adjunto permitido, no obligatorio.
- Vacaciones con política configurable: anual al aniversario o proporcional mensual, con ajustes manuales por RH.

## Fase 1 - Núcleo de identidad y expediente
### Objetivo
Habilitar registro, activación y administración del colaborador.

### Alcance
- Usuario, contraseña y estado de cuenta.
- Auto-registro de colaborador.
- Activación manual por RH.
- Expediente personal y laboral.
- Departamentos, horarios y ubicación principal de tienda.
- Asignación de roles.

### Criterios de aceptación
- Un colaborador puede registrarse pero no operar hasta ser activado.
- RH puede editar expediente, horario, departamento y zona asignada.
- Un gerente solo queda asociado a uno o más departamentos.

## Fase 2 - Asistencia y checador móvil
### Objetivo
Operar marcaciones con evidencia y revisión.

### Alcance
- Entrada y salida obligatorias.
- Salida a comer y regreso opcionales.
- Captura de selfie, GPS y hora servidor.
- Validación de radio permitido.
- Etiqueta automática `Fuera de zona`.
- Estado diario de asistencia e incidencias.

### Criterios de aceptación
- No se duplican tipos de marca en el mismo día.
- La secuencia inválida de marcación se rechaza.
- RH puede consultar detalle por colaborador y día.

## Fase 3 - Ausencias y vacaciones
### Objetivo
Controlar solicitudes de ausencia con validación de saldo y aprobación RH.

### Alcance
- Solicitudes: vacaciones, permiso con goce, permiso sin goce, incapacidad.
- Historial de estados y motivo de rechazo.
- Adjuntos de soporte para incapacidad cuando existan.
- Libro mayor de vacaciones con saldo consultable.
- Políticas configurables de devengo.
- Ajustes manuales por RH.

### Criterios de aceptación
- Vacaciones no se envían sin saldo suficiente.
- RH puede aprobar o rechazar con trazabilidad.
- El saldo refleja devengos, ajustes y consumos.

## Fase 4 - Horas extra y reportes
### Objetivo
Permitir captura, revisión y consulta operacional.

### Alcance
- Captura de horas extra con evidencia fotográfica.
- Aprobación o rechazo por RH.
- Reportes en pantalla:
  - marcaciones,
  - incidencias,
  - horas extra,
  - saldos y solicitudes.

### Criterios de aceptación
- Las horas aprobadas aparecen en reportes del colaborador y RH.
- Gerentes solo consultan datos de su departamento.

## Fase 5 - Alertas y notificaciones
### Objetivo
Cerrar el circuito de comunicación del MVP.

### Alcance
- Alertas de cumpleaños y aniversarios.
- Notificaciones in-app y push por cambios de estado.
- Lectura de notificaciones por usuario.

### Criterios de aceptación
- El colaborador recibe aviso al aprobarse o rechazarse una solicitud.
- RH y gerentes pueden consultar alertas del equipo según permisos.

## Riesgos y pendientes para fase 2+
- Cálculo legal completo de vacaciones por tablas multi-anuales de México puede requerir afinar reglas si el cliente desea cubrir escalas más allá del valor inicial de 12 días.
- La tolerancia offline de la app requiere estrategia de sincronización en backend y móvil.
- Multi-sucursal, exportaciones y nómina quedan fuera de este MVP.

## Orden recomendado de implementación
1. Modelo de datos y catálogos.
2. Autenticación, roles y expediente.
3. Asistencia.
4. Ausencias y vacaciones.
5. Horas extra.
6. Reportes.
7. Alertas y notificaciones.
