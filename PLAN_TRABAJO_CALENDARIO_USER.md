# Plan de trabajo — calendario de agendamiento USER (opción A)

Marcar `[x]` al completar. Contrato nuevo: `GET /api/v1/availability/days?locationId&specialtyId&from&to` (USER) → `[{date, slots}]`.

- [x] 1. Backend: `SchedulingService.availableDays` + endpoint + validaciones (400 rango, 403 rol).
- [x] 2. Backend: pruebas Testcontainers.
- [x] 3. Semilla dev `citas-api/scripts/seed-dev-agenda.sql` + `.ps1` (29-sep a 15-oct-2026), ejecutada.
- [x] 4. Frontend: `appointmentsApi.availableDays` + prueba.
- [x] 5. Frontend: `AvailabilityCalendar.tsx` y modal en 3 pasos (especialidad/sede → calendario+horas → confirmación) + pruebas.
- [x] 6. Suites: Maven, lint, Vitest, build.
- [x] 7. Recorrido web USER con datos semilla + verificación MySQL.
- [x] 8. Documentación: contracts.md, log.md, evidencia, HU-021/033, current-state.md; `git diff --check`.
- [x] 9. API y web ejecutándose.
