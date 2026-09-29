# Handoff técnico — FCV Citas

**Fecha:** 2026-09-24  
**Destino:** Claude, para continuar la implementación y validación de S5.  
**Workspace:** raíz con dos repositorios independientes: `citas-api` y `citas-web`.

## Estado confirmado antes del handoff

| Componente | Estado |
|---|---|
| `citas-api` | Rama `develop`, Swagger y adaptador S5 confirmados en el historial; corrección local de configuración n8n pendiente de validación y commit. |
| `citas-web` | Rama `develop`, commit `5438094`, sin cambios pendientes en este corte. |
| Docker | El usuario reinició los contenedores. Codex no pudo ejecutar `docker compose ps` porque el sandbox no tuvo acceso al socket de Docker; confirmar estado desde el host antes de probar. |
| S2–S4 | Integrados previamente: identidad, modelo 3FN, oferta, disponibilidad, reserva, cancelación, reprogramación, agenda profesional, bandeja ADMIN, auditoría y frontend React/Vite. |
| S5 | Iniciada, no validada. Swagger y el adaptador de webhook fueron añadidos parcialmente. |

## Corrección de configuración n8n — 2026-09-25

- `docker-compose.yml` se restauró exactamente a la versión del repositorio: no transporta URL, Bearer ni variables de n8n.
- Las variables fueron añadidas con valores vacíos y desactivados a `citas-api/.env`; su plantilla versionada está en `citas-api/.env.example`.
- WF-002 usa exclusivamente `N8N_STATUS_WEBHOOK_*`. WF-001 y WF-003 tienen prefijos reservados independientes: `N8N_REMINDERS_WEBHOOK_*` y `N8N_DAILY_SUMMARY_WEBHOOK_*`.
- La API solo consume WF-002 por ahora. Los otros dos grupos no activan llamadas ni workflow alguno.
- Validación pendiente: Docker Desktop no expone el daemon `dockerDesktopLinuxEngine` en este equipo, por lo que `docker compose exec -T citas-api-dev mvn test` no pudo ejecutarse. Cuando Docker esté iniciado, ejecutar la suite indicada en el plan de este documento.

## Cambios S5 ya aplicados en `citas-api`

Estos cambios están sin commit y deben revisarse, compilarse y probarse antes de conservarlos:

- `pom.xml`: añade `org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.13`.
- `OpenApiConfig.java`: declara título `FCV Training Citas API`, tags y esquema `bearerAuth` JWT.
- `SecurityConfig.java`: deja públicos `/swagger-ui/**`, `/swagger-ui.html` y `/v3/api-docs/**`; las rutas de negocio siguen protegidas.
- Controladores de autenticación, agenda y perfil: añaden tags Swagger; agenda y perfil declaran Bearer JWT en OpenAPI.
- `Ports.AppointmentStatusChanged`: incorpora `eventId` UUID.
- `SchedulingService`: genera el UUID y sigue publicando el evento después del commit.
- `NoOpAppointmentEvents`: queda activo únicamente cuando `app.n8n.status-webhook.enabled=false`.
- `N8nWebhookAppointmentEvents.java`: nuevo adaptador HTTP activado solo cuando `app.n8n.status-webhook.enabled=true`.
  - Envía `Authorization: Bearer <token>`.
  - Envía solo `schemaVersion`, `eventId`, `eventType`, `appointmentId`, `status`, `source`, `occurredAt`.
  - Emite `APPROVED` y `REJECTED` provenientes de ADMIN, y `CANCELLED` proveniente de USER.
  - No emite la aprobación automática de medicina general (`SYSTEM`) ni cierres clínicos.
  - Si n8n devuelve error o falla la red, registra un aviso técnico sin token y no revierte la transacción ya confirmada.
- `application.yml`: incorpora propiedades `app.n8n.status-webhook.*`.
- `docker-compose.yml` no contiene variables de n8n. `citas-api/.env.example` define secretos separados para WF-001, WF-002 y WF-003; los valores reales viven únicamente en `citas-api/.env`.
- Pruebas nuevas o modificadas aún sin ejecutar:
  - `N8nWebhookAppointmentEventsTest.java`: Bearer, payload mínimo, filtrado y tolerancia a error HTTP.
  - `AuthIntegrationTest.java`: endpoint `/v3/api-docs` público y esquema Bearer.

## Archivos que no deben mezclarse con S5

`citas-api/docs/FCV Dev/.obsidian/graph.json` ya estaba modificado antes de este corte. No pertenece al trabajo de Swagger o n8n; preservarlo sin revertirlo ni incluirlo en un commit de S5.

La documentación Wiki, la trazabilidad Scrum y la HU-035 **todavía no fueron actualizadas**. No se creó ni configuró un workflow real en n8n Cloud, ni se añadió MCP.

## Plan obligatorio para Claude

1. **Revisar y compilar lo pendiente.** Confirmar que Spring Boot 3.5.0 resuelve springdoc `2.8.13`; corregir solo incompatibilidades reales. Revisar que exista exactamente una implementación de `Ports.AppointmentEvents` para cada valor de `N8N_STATUS_WEBHOOK_ENABLED`.
2. **Ejecutar validación backend.** Desde la raíz, con Docker levantado:

   ```powershell
   docker compose ps
   docker compose exec citas-api-dev mvn test
   ```

   Corregir fallos de compilación o pruebas antes de continuar. Verificar también que `GET http://localhost:8080/v3/api-docs` responda `200` y que `http://localhost:8080/swagger-ui/index.html` cargue.
3. **Arrancar y validar servicios locales.** En terminales separadas:

   ```powershell
   docker compose exec -e SPRING_PROFILES_ACTIVE=local citas-api-dev mvn spring-boot:run
   docker compose exec citas-web-dev npm run dev
   ```

   Verificar frontend en `http://localhost:5173` y API en `http://localhost:8080`. Ejecutar en frontend:

   ```powershell
   docker compose exec citas-web-dev npm run lint
   docker compose exec citas-web-dev npm test
   docker compose exec citas-web-dev npm run build
   ```

4. **Configurar n8n Cloud sin versionar secretos.** El usuario debe crear el flujo con trigger **On Webhook Call**, Header Auth que valide `Authorization: Bearer <token compartido>` y respuesta inmediata HTTP 200. Configurar solamente en `citas-api/.env`:

   ```text
   N8N_STATUS_WEBHOOK_ENABLED=true
   N8N_STATUS_WEBHOOK_URL=<URL única generada por n8n Cloud para WF-002>
   N8N_STATUS_WEBHOOK_BEARER_TOKEN=<Bearer exclusivo de WF-002>
   ```

   Reiniciar la API. WF-001 y WF-003 tienen sus propios prefijos `N8N_REMINDERS_*` y `N8N_DAILY_SUMMARY_*`, pero no están consumidos todavía por la API. No incluir URL privada, token ni credenciales Gmail/n8n en Git, evidencia o JSON.
5. **Prueba funcional n8n.** Con datos sintéticos, ejecutar una aprobación especializada, un rechazo especializado y una cancelación de USER. Confirmar tres ejecuciones en n8n Cloud y que no se modifiquen indebidamente `appointments`, `professional_slots` ni `appointment_status_history`.
6. **Verificar datos en MySQL.** Abrir cliente dentro del contenedor:

   ```powershell
   docker compose exec -it mysql sh -lc 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE"'
   ```

   Consultas de lectura:

   ```sql
   SELECT id, status_id, scheduled_start_at, scheduled_end_at FROM appointments ORDER BY id DESC LIMIT 20;
   SELECT appointment_id, status_id, change_source, changed_at FROM appointment_status_history ORDER BY id DESC LIMIT 20;
   ```

7. **Cerrar evidencia.** Actualizar `citas-api/docs/FCV Dev/llm-wiki/wiki/contracts.md`, `decisions.md`, `traceability.md` y `log.md`; actualizar HU-035 a `En curso` con los resultados reales. HU-035 no puede marcarse completada hasta exportar `automations/n8n/WF-002-status-notifications.json` sin credenciales y cumplir CA-01 a CA-03.
8. **Control de cambios.** Ejecutar `git diff --check` en ambos repositorios, confirmar que no se expusieron secretos y crear commits separados, trazables y solo con archivos de cada repositorio. La raíz no es un tercer repositorio Git.

## Límites de esta sesión

- Se trabaja solo con webhook HTTP saliente hacia n8n Cloud.
- MCP contra n8n Server queda para la próxima sesión.
- No se implementan reintentos persistentes, Gmail, SMS, WhatsApp ni cambios de estado desde n8n.
- n8n recibe eventos posteriores al commit; el núcleo de citas mantiene la autoridad sobre agenda, slots y auditoría.

## Cierre del núcleo de agendamiento (S3) — 2026-09-25

Ejecutado según `PLAN_CIERRE_AGENDAMIENTO_S3.md`. Evidencia completa: `citas-api/docs/FCV Dev/evidence/S3-cierre-agendamiento.md`.

- **Backend** (`SchedulingService`): la reserva rechaza slots retenidos por reprogramación; se corrigió una **doble reserva concurrente** real (lectura de ocupación ahora bloqueante); redecidir una cita devuelve `409` en lugar de `500`. Nueva `SchedulingClosureIntegrationTest` (6 escenarios). `mvn test` vía `compose.test.yml`: **22/22**.
- **Frontend**: modal USER en orden sede/especialidad/fecha → profesional → franja → confirmación, con resultado `APPROVED`/`REQUESTED` y manejo de `409`; ADMIN usa `/admin/appointments/pending-specialized` con motivo de rechazo visible; hora local sin depender de la zona del navegador (`bogotaTime.ts`); IDs de catálogo normalizados. `lint`, **21/21** Vitest y `build` en verde (en el host).
- **Recorrido REST + MySQL** con los tres roles: pasa (estados, slots, historial `SYSTEM`/`USER`/`ADMIN`, 403/400/409).
- **Entorno**: `citas-api-dev` no tiene socket Docker (usar `compose.test.yml`); `citas-web-dev` tiene `node_modules` de Windows (Vite no arranca en el contenedor). Vite se ejecutó en el host en `:5174` con la API iniciada con `FRONTEND_ORIGIN=http://localhost:5174`.
- **Pendiente**: recorrido web visual con los tres roles (hacerlo una persona), HU-019 sin reverificar, cambios sin commit en ambos repositorios (separar de los cambios S5 existentes y de `.obsidian/graph.json`).

## Actualización 2026-09-29 — cierre S3 completado

- Recorrido web USER/PROFESSIONAL/ADMIN ejecutado por el agente con cuentas sintéticas (autorizado en Wiki `preferences.md`). Evidencia: `citas-api/docs/FCV Dev/evidence/S3-cierre-agendamiento.md`.
- HU-018 a HU-024: `Completada`. HU-033: `En progreso` (resto de pantallas fuera del corte).
- Defectos UI corregidos: formato de fecha en confirmación, refresco de "Mis citas", franjas obsoletas al reabrir el modal, edición de bloques (HU-019).
- Suites: Maven 22/22, Vitest 23/23, lint y build OK.
- Servicios en ejecución: API `http://localhost:8080` (perfil `local`), web `http://localhost:5173` (Vite en el contenedor desde `/opt/web`). Comandos de arranque en `PLAN_TRABAJO_CIERRE_S3.md`.
- Pendiente: commits separados en `develop` (los cambios S5 previos y `.obsidian/graph.json` no deben mezclarse); luego continuar S5 (n8n) según este documento.

## Actualización 2026-09-29 — calendario USER

- Nuevo endpoint `GET /api/v1/availability/days` y modal de reserva en 3 pasos con calendario (evidencia `citas-api/docs/FCV Dev/evidence/UX-calendario-user.md`, plan `PLAN_TRABAJO_CALENDARIO_USER.md`).
- Semilla dev: `.\citas-api\scripts\seed-dev-agenda.ps1` (agenda 29-sep a 15-oct-2026, idempotente).
- Maven 23/23, Vitest 27/27. Cambios sin commit en ambos repos.
