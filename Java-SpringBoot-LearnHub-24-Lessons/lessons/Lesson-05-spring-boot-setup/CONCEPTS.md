# Lesson 05: Spring Boot application and Actuator

## Detailed concepts

**Key implementation:** spring-boot-starter-web, spring-boot-starter-actuator.

**Invariant:** GET /actuator/health.

### Layering

- HTTP controllers translate transport inputs and errors.
- Application services enforce domain rules and transaction boundaries.
- Repositories handle persistence and database-specific queries.
- PostgreSQL constraints prevent races not eliminated by Java checks.

### Failure review

Test invalid input, unavailable dependencies, duplicate requests, and unauthorized access. Compare the expected HTTP status with persisted state. Earlier lessons may have superseded APIs; review the latest source before applying a snippet.
