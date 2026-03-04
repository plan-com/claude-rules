# Backend Developer Role — Claude Rules (General)

You are assisting a **Backend Developer** at Plan.com. Follow these rules strictly.

## Infrastructure Context

All backend services are deployed as Docker containers across Plan.com's swarm clusters:

- **Cluster-A-Dev-Stage** (Dev/Stage) — **Cluster-B-Prod-Blue** (UAT/Demo/Prod Blue) — **Cluster-C-Prod-Green** (Prod Green)
- Images are built by **Jenkins** on **Cluster-D-Platform** and pushed to the **self-hosted Registry**
- Monitoring via **SigNoz** (traces/logs) and **Grafana+Prometheus** (metrics) on **Cluster-D-Platform**

For stack-specific rules, see the subdirectories: `php/`, `nodejs-typescript/`, `python/`, `golang/`.

## Rules

### General Backend Principles

- Write clean, maintainable, and testable code.
- Follow SOLID principles and clean architecture patterns.
- Every service must have a clear separation of concerns: routing, business logic, data access.
- Prefer composition over inheritance.
- Keep functions/methods small and focused — single responsibility.
- Handle errors explicitly. Never silently swallow exceptions.

### API Design

- RESTful APIs must follow consistent naming conventions: plural nouns for resources, HTTP verbs for actions.
- Use proper HTTP status codes: `200` success, `201` created, `400` bad request, `401` unauthorized, `403` forbidden, `404` not found, `422` validation error, `500` server error.
- Version APIs in the URL path (`/api/v1/`) or via headers — be consistent within a project.
- All API responses must follow a consistent envelope structure.
- Document APIs with OpenAPI/Swagger specifications.
- Implement pagination for list endpoints.
- Use request validation at the API boundary — never trust client input.

### Database

- Use migrations for all schema changes — never modify databases manually.
- Write queries that are index-aware. Avoid full table scans.
- Use connection pooling appropriate to the language/framework.
- Implement soft deletes for important business entities.
- Separate read and write operations where performance demands it.
- Never store secrets or passwords in plain text — always hash with proper algorithms (bcrypt, argon2).

### Docker & Deployment

- Every service must have a `Dockerfile` following best practices:
  - Use multi-stage builds to reduce image size.
  - Pin base image versions — never use `latest`.
  - Pull base images from the **self-hosted Registry on Cluster-D-Platform**.
  - Run as non-root user.
  - Use `.dockerignore` to exclude unnecessary files.
- Include a `docker-compose.yml` for local development that mirrors the swarm stack structure.
- Expose health check endpoints (`/health` or `/healthz`) for Swarm health checks and Traefik.
- Expose metrics endpoints (`/metrics`) for Prometheus scraping where applicable.

### Configuration

- Use environment variables for configuration — follow 12-factor app principles.
- Provide sensible defaults for local development.
- Never hardcode connection strings, API keys, or secrets.
- Use Docker secrets in swarm deployments for sensitive configuration — secrets sourced from Vault.
- Separate configuration by environment using the standard suffixes: `dev`, `stg`, `uat`, `demo`, `prod`.

### Logging & Observability

- Use structured logging (JSON format) so SigNoz can parse and index fields.
- Log levels: `DEBUG` (dev only), `INFO` (normal operations), `WARN` (recoverable issues), `ERROR` (failures requiring attention).
- Include correlation/trace IDs in all log entries for distributed tracing.
- Never log sensitive data (passwords, tokens, PII).
- Instrument code with OpenTelemetry for distributed tracing — SigNoz is the collection backend.

### Testing

- Write unit tests for business logic — aim for meaningful coverage, not arbitrary percentages.
- Write integration tests for API endpoints and database interactions.
- Use test fixtures and factories — never depend on production data.
- Tests must be runnable in CI (Jenkins) without external dependencies — use test containers or mocks.
- Include contract tests when services communicate with other backend services.

### Security

- Validate and sanitize all input at service boundaries.
- Use parameterized queries — never concatenate user input into SQL.
- Implement authentication and authorization checks at the middleware/interceptor level.
- Set proper CORS headers for APIs consumed by frontend applications.
- Rate limit public-facing endpoints.
- Keep dependencies up to date — regularly audit for known vulnerabilities.

### Git & CI/CD

- Follow conventional commits or the team's agreed commit message format.
- Feature branches must pass all tests in Jenkins before merging.
- Code must be reviewed before deployment to Cluster-B-Prod-Blue or Cluster-C-Prod-Green.
- Deployments follow the promotion path: `Cluster-A-Dev-Stage → Cluster-B-Prod-Blue → Cluster-C-Prod-Green`.

## Response Guidelines

- When writing code, always include error handling and input validation.
- Suggest appropriate design patterns for the problem at hand.
- When proposing database changes, include the migration.
- Always consider how the code will behave in a containerized, multi-replica swarm environment.
- Flag potential performance issues proactively.
