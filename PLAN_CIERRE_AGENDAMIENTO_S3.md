# Plan de cierre — integración del núcleo de agendamiento

**Fecha:** 2026-09-25  
**Destino:** Claude, para implementar, probar y documentar el cierre funcional antes de n8n.  
**Repositorios:** `citas-api` y `citas-web` son repositorios Git independientes; la raíz solo los orquesta.

## Objetivo y criterio de cierre

Dejar verificado de extremo a extremo, desde el frontend hasta MySQL, este recorrido con datos sintéticos:

1. PROFESSIONAL publica disponibilidad futura en una sede asignada.
2. USER consulta fechas, profesionales y franjas disponibles.
3. USER reserva Medicina General y recibe `APPROVED` sin intervención ADMIN.
4. USER solicita una cita especializada y recibe `REQUESTED` con sus slots retenidos.
5. ADMIN aprueba una solicitud y rechaza otra con motivo; se comprueban estado, slots e historial.

El corte termina cuando el recorrido web, REST, base de datos y pruebas automatizadas tienen evidencia verificable. No declarar completas las HU solo por existir código o por pruebas anteriores de S4.

## Estado encontrado

- `current-state.md` conserva referencias y pendientes de un handoff anterior; actualizarlo con resultados reales al cerrar este trabajo.
- La Wiki registra implementación S3/S4, pero HU-018 a HU-024 mantienen CA y DoD pendientes. HU-033 sigue en progreso.
- `npm run lint` pasó durante la inspección del 2026-09-25. Vitest no pudo iniciar por `spawn EPERM` en el entorno de Codex. Docker no permitió acceso al socket y los servicios HTTP locales no respondieron; se requiere volver a ejecutar estas verificaciones en un entorno funcional.
- Preservar los cambios locales existentes, especialmente `citas-api/docs/FCV Dev/.obsidian/graph.json` y los archivos de configuración n8n. No modificar `docker-compose.yml` para este cierre.

### Defectos o riesgos concretos a resolver

| Prioridad | Hallazgo | Resultado esperado |
|---|---|---|
| 1 | `SchedulingService.reserve` comprueba `appointment_id`, pero no `reschedule_request_id` antes de asignar slots. | Una franja retenida por reprogramación devuelve `409` y ninguna cita nueva la ocupa. |
| 2 | `BookAppointmentModal` exige elegir profesional antes de permitir cambiar la fecha; empieza consultando hoy. | USER puede seleccionar una fecha futura y luego elegir un profesional con disponibilidad en esa fecha. |
| 3 | La pantalla ADMIN consume `/admin/inbox`, cuya respuesta tiene `appointmentId`, no `id`, e incluye reprogramaciones. | La decisión especializada utiliza `/admin/appointments/pending-specialized` y envía un identificador válido al endpoint de decisión. Las reprogramaciones conservan su tratamiento propio de S4. |
| 4 | El cliente de bloques profesionales espera `date`, `start` y `end`. | Confirmar con JSON real que coincide con `SchedulingService.Block`; corregir el mapeo solo si la respuesta difiere y verificar publicación y visualización web. |
| 5 | El modal convierte fechas y horas con `new Date(startAt)`. | La selección y el envío preservan la fecha `YYYY-MM-DD` y la hora local `HH:mm` de `America/Bogota` sin depender de la zona horaria del navegador. |

## Orden de implementación

1. **Backend e integridad.** Corregir la comprobación de slots retenidos en la transacción de reserva. Cubrir reserva general, solicitud especializada, retención, concurrencia y decisiones ADMIN con pruebas de integración MySQL. Mantener intactas las migraciones aplicadas; este cambio no requiere migración nueva.
2. **Frontend USER.** Reordenar la selección del modal a sede, especialidad y fecha; profesional; franja; confirmación. Reconsultar disponibilidad al cambiar filtros y limpiar selecciones incompatibles. Mantener los componentes, colores, tipografía y jerarquía visual aprobados de Stitch. Mostrar la respuesta real `APPROVED` o `REQUESTED` y el error `409` cuando la franja deje de estar libre.
3. **Frontend PROFESSIONAL.** Verificar que crear y listar bloques muestre sede, fecha, inicio y fin reales. Ajustar `schedulingApi.ts` solo según el JSON observado y cubrir la conversión con una prueba de cliente.
4. **Frontend ADMIN.** Obtener las solicitudes especializadas desde `/admin/appointments/pending-specialized`, aprobar o rechazar usando el `id` de la cita y refrescar la bandeja. Exigir motivo visible al rechazar. Mantener `/admin/inbox` para el flujo S4 que distingue solicitudes especializadas de reprogramaciones.
5. **Validación integral.** Levantar infraestructura, API y web; ejecutar suites y recorrido manual con usuarios sintéticos de los tres roles. Inspeccionar estado, slots e historial en MySQL sin imprimir secretos ni datos personales en evidencias.
6. **Documentación y trazabilidad.** Registrar resultados y fallos corregidos en `current-state.md`, Wiki, evidencia de sesión y HU-018 a HU-024 y HU-033. Marcar cada CA/DoD como completado solo con prueba o recorrido trazable. Ejecutar LINT de la Wiki y `git diff --check` en ambos repositorios.

## Contrato y compatibilidad

- `citas-web` sigue consumiendo directamente `citas-api` por REST bajo `/api/v1`; no agregar BFF.
- Conservar los endpoints aprobados: `GET|POST /professional/availability-blocks`, `GET /availability`, `POST /appointments`, `GET /admin/appointments/pending-specialized` y `POST /admin/appointments/{id}/decision`.
- Conservar formatos `YYYY-MM-DD` y `HH:mm` en `America/Bogota` y estados `APPROVED`, `REQUESTED`, `REJECTED`.
- Errores relevantes: `400` por entrada inválida o rechazo sin motivo; `401/403` por autenticación y rol; `409` por franja ocupada, retenida o transición inválida.
- Antes de cualquier cambio contractual imprevisto, documentar impacto en ambos repositorios, compatibilidad, migración y pruebas según `AGENTS.md`.

## Pruebas y evidencia obligatorias

### Automatizadas

- Backend: ejecutar la suite Maven con MySQL Testcontainers. Añadir escenarios de 30 y 60 minutos, slots consecutivos, profesional/sede/especialidad no habilitados, doble reserva concurrente, slot retenido por reprogramación, general `APPROVED`, especializada `REQUESTED`, aprobación y rechazo con liberación de slots e historial `SYSTEM`/`USER`/`ADMIN`.
- Frontend: ejecutar `npm run lint`, `npm test` y `npm run build`. Cubrir fecha futura, cambio de filtros, mapeo de bloques, identificador de decisión ADMIN, error `409` y confirmaciones de ambos tipos de cita.
- Si Docker o Vitest fallan por el entorno, registrar el error exacto, resolver la condición y repetir la prueba; no registrar `PASS` basándose en resultados históricos.

### Recorrido manual entre repositorios y MySQL

1. Confirmar `docker compose ps`, API y Vite; iniciar los procesos dentro de los contenedores de desarrollo si solo están en espera.
2. Crear o utilizar cuentas **sintéticas** USER, PROFESSIONAL y ADMIN por el flujo autorizado del proyecto; asignar al profesional una sede, Medicina General y una especialidad de 60 minutos.
3. Desde la web PROFESSIONAL, publicar dos bloques futuros separados y confirmar que el intervalo intermedio no se ofrece.
4. Desde la web USER, filtrar por sede, especialidad y fecha futura; reservar una franja general. Confirmar `APPROVED` en UI, REST, `appointments`, `professional_slots` y `appointment_status_history` con fuente `SYSTEM`.
5. Solicitar dos citas especializadas en franjas distintas. Confirmar `REQUESTED`, dos slots consecutivos retenidos por cita y fuente `USER`; intentar una segunda reserva sobre la misma franja y confirmar `409` sin duplicado.
6. Desde la web ADMIN, aprobar una y rechazar la otra con motivo. Confirmar `APPROVED` con slots conservados y `REJECTED` con slots liberados, fuentes `ADMIN` e historial correspondiente. La franja rechazada debe volver a aparecer en disponibilidad.
7. Verificar que USER no pueda ejecutar decisiones ADMIN y que PROFESSIONAL solo gestione sus bloques.

Guardar en la evidencia: fecha, ramas y commits, comandos y resultados, identificadores sintéticos de citas/bloques, CA relacionados y consultas SQL de lectura. No copiar credenciales, tokens ni PII a la Wiki o al repositorio.

## Alcance de este corte

La meta es el núcleo de agendamiento previo a S5. La integración con n8n Cloud, Gmail, MCP y los flujos de automatización quedan fuera de esta verificación. `main` se reserva para un incremento estable; trabajar en `develop` sin reescribir historial.
