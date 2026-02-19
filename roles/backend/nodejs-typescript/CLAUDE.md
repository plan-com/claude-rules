# Backend Developer Role — Node.js + TypeScript Stack — Claude Rules

You are assisting a **Backend Developer working with Node.js and TypeScript** at Plan.com. Follow the general backend rules in `roles/backend/CLAUDE.md` plus these Node.js+TypeScript-specific rules.

## Stack

- **Runtime**: Node.js 20 LTS+
- **Language**: TypeScript (strict mode)
- **Package Manager**: npm or pnpm (follow project convention)
- **Frameworks**: Express, Fastify, NestJS, or Hono — follow the project's established choice
- **ORM/Query Builder**: Prisma, TypeORM, Drizzle, or Knex — follow the project's established choice
- **Testing**: Jest or Vitest, Supertest for HTTP tests
- **Linting**: ESLint with TypeScript parser
- **Formatting**: Prettier
- **Containerized**: Docker multi-stage builds, deployed to Plan.com swarm clusters

## Rules

### TypeScript Standards

- Enable `strict: true` in `tsconfig.json` — no exceptions.
- Never use `any` — use `unknown` and narrow with type guards if the type is truly dynamic.
- Define explicit types for function parameters and return values.
- Use interfaces for object shapes and types for unions/intersections.
- Prefer `const` assertions and enums for fixed value sets.
- Use generics for reusable, type-safe abstractions.
- Avoid type assertions (`as`) — prefer type guards and narrowing.
- Keep `@ts-ignore` / `@ts-expect-error` count at zero in production code.

### Code Patterns

- Use `async/await` — never use raw callbacks or `.then()` chains for complex flows.
- Handle promise rejections explicitly — never leave unhandled promises.
- Use dependency injection — NestJS has it built-in; for others, use a DI container or constructor injection.
- Separate routing, business logic, and data access into distinct layers.
- Use middleware for cross-cutting concerns (auth, logging, validation, error handling).
- Validate request payloads at the boundary using Zod, Joi, class-validator, or equivalent.
- Prefer immutable data patterns — avoid mutating objects in place.

### Node.js Specifics

- Use the `node:` prefix for built-in modules (`import fs from 'node:fs'`).
- Handle `process` signals (`SIGTERM`, `SIGINT`) for graceful shutdown in Docker Swarm.
- Implement graceful shutdown: stop accepting new requests, finish in-flight work, close DB connections, then exit.
- Use `node --max-old-space-size` to match container memory limits.
- Avoid synchronous file I/O in request handlers.
- Use streams for large data processing.

### Package Management

- Lock file (`package-lock.json` or `pnpm-lock.yaml`) must be committed.
- Use exact versions in `package.json` for critical dependencies.
- Run `npm audit` or `pnpm audit` regularly.
- Minimize dependency count — prefer standard library and small, focused packages.
- Keep `devDependencies` separate from `dependencies`.

### Testing

- Write tests in TypeScript — test files should mirror source structure.
- Use Vitest or Jest with TypeScript support.
- Use Supertest for HTTP endpoint testing.
- Mock external dependencies (databases, APIs) — use dependency injection for testability.
- Test error paths and edge cases, not just happy paths.
- Use test containers for integration tests requiring databases.
- Target meaningful test coverage for business logic.

### Docker & Deployment

- Use `node:20-alpine` or equivalent slim base image from the **self-hosted Registry**.
- Multi-stage build: install all deps + compile TS in build stage, copy only compiled JS and production `node_modules` to runtime stage.
- Use `.dockerignore` to exclude: `node_modules`, `.git`, `*.md`, test files, `.env`.
- Set `NODE_ENV=production` in runtime stage.
- Use `dumb-init` or `tini` as PID 1 to handle signal forwarding properly.
- Health check endpoint: lightweight `GET /health` that returns `200` with minimal processing.

### Logging & Observability

- Use `pino` or `winston` for structured JSON logging.
- Include `traceId`, `spanId`, `service`, and `environment` fields in every log entry.
- Integrate OpenTelemetry Node.js SDK for distributed tracing to SigNoz.
- Expose `/metrics` endpoint with `prom-client` for Prometheus scraping.
- Use correlation IDs from incoming request headers for trace propagation.

### Performance

- Use connection pooling for database connections.
- Implement caching (Redis) for frequently accessed, rarely changing data.
- Avoid blocking the event loop — offload CPU-intensive work to worker threads.
- Use streaming responses for large payloads.
- Profile with `--inspect` on Constellations — never enable the inspector on Galaxies or StarWars.

### Security

- Use `helmet` middleware for HTTP security headers.
- Implement rate limiting with `express-rate-limit` or framework equivalent.
- Never use `eval()`, `new Function()`, or `child_process.exec()` with user input.
- Sanitize user input before database queries.
- Use environment variables for secrets — never import `.env` files in production.
- Enable CORS with explicit origin allowlists — never use `*` in production.

## Response Guidelines

- Write idiomatic TypeScript with full type safety.
- Always include error handling with typed error responses.
- Provide test examples alongside new code.
- Flag potential event loop blocking operations.
- Consider Docker Swarm's multi-replica environment — ensure statelessness.
