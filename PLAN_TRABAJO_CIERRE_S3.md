# Plan de trabajo — cierre S3 (seguimiento vivo)

Fuente: `PLAN_CIERRE_AGENDAMIENTO_S3.md`, `current-state.md`, HU-018 a HU-024 y HU-033.
Evidencia: `citas-api/docs/FCV Dev/evidence/S3-cierre-agendamiento.md`. Marcar `[x]` al completar.

## Ya hecho (2026-09-25)
- [x] Correcciones backend (retención reprogramación, concurrencia, 409 redecisión) + `SchedulingClosureIntegrationTest`; Maven 22/22.
- [x] Correcciones frontend (modal, ADMIN pending-specialized, hora local, ids); lint, Vitest 21/21, build.
- [x] Recorrido REST + MySQL con tres roles.
- [x] Evidencia, HU (CA), Wiki, `current-state.md`.

## Pendiente (2026-09-29)
- [x] 1. Levantar API (`citas-api-dev`, perfil local) y web (Vite) y confirmar HTTP 200.
- [x] 2. Recorrido web PROFESSIONAL: publicar 2 bloques futuros, verificar listado (sede, fecha, inicio, fin).
- [x] 3. HU-019: editar y eliminar bloque futuro; bloque con cita protegido (409).
- [x] 4. Recorrido web USER: cita general → `APPROVED`; dos especializadas → `REQUESTED`.
- [x] 5. Recorrido web ADMIN: aprobar una, rechazar otra con motivo; verificar en MySQL estado/slots/historial.
- [x] 6. Verificar HU-018 CA-02 (restricciones) y HU-020 CA-03 (aislamiento) por REST.
- [x] 7. Re-ejecutar suites: Maven (`compose.test.yml`), web lint/test/build.
- [x] 8. Actualizar evidencia, HU (CA/DoD/estado), Wiki (traceability, log), `current-state.md`.
- [x] 9. `git diff --check` en ambos repos; sin secretos.
- [x] 10. Dejar API y web ejecutándose; anotar URLs y cómo arrancarlas.

- Defectos UI hallados en el recorrido y corregidos: fecha sin formato en diálogo de confirmación (`App.tsx`), "Mis citas" no se refrescaba tras reservar, modal reabierto mostraba franjas obsoletas.

## Notas para continuar
- Tests backend: `cd citas-api && docker compose -f compose.test.yml run --rm api-test mvn test` (en Git Bash anteponer `MSYS_NO_PATHCONV=1`).
- API: `docker compose exec -d -e SPRING_PROFILES_ACTIVE=local -e FRONTEND_ORIGIN=<origen web> citas-api-dev sh -c 'mvn -q spring-boot:run > /tmp/api.log 2>&1'`.
- Web (en contenedor): dependencias Linux en `/opt/web` del contenedor `citas-web-dev` (fuente enlazada a `/workspace/src`, config `vite.config.mjs` propia). Arranque: `docker compose exec -d citas-web-dev sh -c "cd /opt/web && npx vite --port=5173 --host=0.0.0.0 > /tmp/web.log 2>&1"`. Si el contenedor se recrea, repetir `npm ci` en `/opt/web` (ver evidencia).
- Cuentas sintéticas: crear con `/auth/register`; ADMIN inicial por `INSERT` en `user_roles` (no hay endpoint de bootstrap). No escribir contraseñas en Wiki/evidencia.
