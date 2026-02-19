# Backend Developer Role — Golang Stack — Claude Rules

You are assisting a **Backend Developer working with Go** at Plan.com. Follow the general backend rules in `roles/backend/CLAUDE.md` plus these Go-specific rules.

## Stack

- **Language**: Go 1.22+
- **Frameworks**: Standard library `net/http`, Chi, Gin, Echo, or Fiber — follow the project's established choice
- **Module System**: Go Modules
- **Testing**: Standard `testing` package, testify
- **Linting**: golangci-lint
- **Containerized**: Docker multi-stage builds, deployed to Plan.com swarm clusters

## Rules

### Go Standards

- Follow Effective Go and the Go Code Review Comments guidelines.
- Use `gofmt` / `goimports` — code must be formatted before committing.
- Use meaningful variable names — avoid single-letter names except in short loops or well-known conventions (`i`, `ctx`, `err`).
- Keep packages focused and small — one package per concern.
- Exported functions and types must have doc comments.
- Use `context.Context` as the first parameter for functions that perform I/O or may be cancelled.
- Return errors — never panic in library or application code (except truly unrecoverable states).
- Wrap errors with `fmt.Errorf("context: %w", err)` for stack context.

### Project Layout

- Follow the standard Go project layout conventions.
- Separate `cmd/`, `internal/`, and `pkg/` directories.
- `cmd/` — Application entry points (main packages).
- `internal/` — Private application code not importable by other modules.
- `pkg/` — Public library code if the project exports packages.
- Keep `main.go` thin — delegate to internal packages.

### Error Handling

- Always check errors — never use `_` to discard error returns (except documented cases).
- Use custom error types for domain errors.
- Use `errors.Is()` and `errors.As()` for error comparison.
- Sentinel errors (`var ErrNotFound = errors.New("not found")`) for expected error conditions.
- Return errors up the call stack — let the caller decide how to handle them.
- Log errors at the boundary (HTTP handler, main), not deep in business logic.

### Concurrency

- Use goroutines and channels appropriately — prefer channels for communication, mutexes for shared state.
- Always use `sync.WaitGroup` or `errgroup.Group` to manage goroutine lifecycles.
- Use `context.Context` for cancellation propagation.
- Avoid goroutine leaks — ensure every goroutine has a clear exit path.
- Use `sync.Pool` for frequently allocated objects when profiling shows GC pressure.
- Never share mutable state between goroutines without synchronization.

### HTTP Services

- Use standard `net/http` as the foundation — add a router (Chi, Gin) for convenience.
- Implement structured middleware: logging, recovery, auth, request ID, CORS.
- Use proper HTTP status codes consistently.
- Implement graceful shutdown: listen for `SIGTERM`/`SIGINT`, drain connections with `http.Server.Shutdown(ctx)`.
- Set appropriate timeouts: `ReadTimeout`, `WriteTimeout`, `IdleTimeout` on `http.Server`.
- Use `encoding/json` with struct tags — never build JSON strings manually.

### Database

- Use `database/sql` with a driver, or `sqlx` / `pgx` for PostgreSQL, `go-sql-driver/mysql` for MySQL.
- Use connection pooling — configure `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`.
- Use prepared statements or parameterized queries — never concatenate user input into SQL.
- Use `sqlc` or `go-migrate` for type-safe queries and migrations respectively.
- Close rows and statements — use `defer rows.Close()` immediately after query.

### Testing

- Use table-driven tests for testing multiple input/output scenarios.
- Use `testify/assert` and `testify/require` for assertions.
- Use `testify/mock` or interfaces for mocking dependencies.
- Use `httptest` for HTTP handler testing.
- Use `testcontainers-go` for integration tests with real databases.
- Benchmark performance-critical code with `testing.B`.
- Use `t.Parallel()` where tests are independent.

### Docker & Deployment

- Multi-stage build: build with `golang:1.22-alpine`, run with `alpine:3.x` or `scratch` from the **self-hosted Registry**.
- Static binary compilation: `CGO_ENABLED=0 GOOS=linux go build -o /app`.
- Final image should be minimal — `scratch` or `distroless` for smallest attack surface.
- Run as non-root user.
- Health check endpoint: `GET /health` returning `200 OK` with minimal logic.
- Handle `SIGTERM` for graceful shutdown in Docker Swarm.

### Logging & Observability

- Use `slog` (standard library, Go 1.21+) or `zerolog` / `zap` for structured JSON logging.
- Include `trace_id`, `span_id`, `service`, and `environment` in log context.
- Integrate OpenTelemetry Go SDK for distributed tracing to SigNoz.
- Expose `/metrics` endpoint with `prometheus/client_golang` for Prometheus scraping.
- Use middleware to inject trace context into request handlers.

### Performance

- Profile with `pprof` on Constellations — expose `/debug/pprof/` only in dev.
- Use `sync.Pool` for high-frequency allocations.
- Prefer stack allocation — avoid unnecessary heap escapes (use `go build -gcflags='-m'` to check).
- Use buffered I/O (`bufio`) for file and network operations.
- Benchmark before optimizing — `go test -bench`.

### Security

- Validate all input — use a validation library or write explicit checks.
- Never use `unsafe` package without documented justification.
- Use `crypto/rand` for random values — never `math/rand` for security purposes.
- Set request body size limits: `http.MaxBytesReader`.
- Use TLS for all external communication.
- Regularly run `govulncheck` to check for known vulnerabilities.

## Response Guidelines

- Write idiomatic Go — simple, explicit, and readable.
- Always handle errors explicitly.
- Provide table-driven test examples alongside new code.
- Include graceful shutdown logic for new services.
- Flag potential goroutine leaks or race conditions.
