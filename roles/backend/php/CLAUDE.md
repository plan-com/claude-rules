# Backend Developer Role — PHP Stack — Claude Rules

You are assisting a **Backend Developer working with PHP** at Plan.com. Follow the general backend rules in `roles/backend/CLAUDE.md` plus these PHP-specific rules.

## Stack

- **Language**: PHP 8.2+
- **Framework**: Laravel (preferred) or Symfony where applicable
- **Package Manager**: Composer
- **Testing**: PHPUnit, Pest
- **Static Analysis**: PHPStan / Larastan
- **Code Style**: PHP-CS-Fixer with PSR-12
- **Containerized**: Docker multi-stage builds, deployed to Plan.com swarm clusters

## Rules

### PHP Standards

- Use strict types in every file: `declare(strict_types=1);`
- Follow PSR-12 coding standard.
- Use type declarations for all function parameters, return types, and class properties.
- Leverage PHP 8.2+ features: enums, readonly properties, fibers, named arguments, match expressions, intersection types.
- Avoid `mixed` type unless absolutely necessary.
- Never use `@` error suppression operator.
- Never use `eval()`, `exec()`, `shell_exec()`, or `system()` with user input.

### Laravel-Specific

- Use Eloquent ORM with proper relationships — avoid raw queries unless performance-critical.
- Define database schema through migrations — never modify the database directly.
- Use Form Requests for validation — never validate in controllers.
- Use API Resources for response transformation.
- Use Policies and Gates for authorization.
- Queue long-running tasks — use Laravel Queues with a proper driver (Redis, database).
- Use Events and Listeners for decoupled side effects.
- Cache aggressively with proper invalidation strategies.
- Use Laravel's built-in rate limiting for API endpoints.
- Store configuration in `config/` files, read from environment variables.

### Composer & Dependencies

- Lock Composer dependencies (`composer.lock` must be committed).
- Specify exact version constraints in `composer.json` where stability matters.
- Run `composer audit` regularly to check for vulnerability advisories.
- Avoid abandoned or unmaintained packages.

### Testing

- Use Pest or PHPUnit — follow the project's established convention.
- Use Laravel's testing helpers: `assertDatabaseHas`, `assertJson`, HTTP test methods.
- Use Factories and Seeders for test data — never fixtures against production data.
- Test feature/integration flows through HTTP tests.
- Test business logic in unit tests isolated from the framework.
- Run `php artisan test --parallel` in CI when possible.

### Docker & Deployment

- Use `php:8.2-fpm-alpine` or equivalent slim base image from the **self-hosted Registry**.
- Multi-stage build: Composer install in build stage, copy vendor to runtime stage.
- Use OPcache in production for performance.
- Configure PHP-FPM pool settings appropriate for the container's resource limits.
- Run artisan commands (migrations, cache clear, config cache) as part of the deployment entrypoint.
- Health check endpoint must bypass middleware stacks for reliability.

### Logging & Observability

- Use Laravel's `Log` facade with structured context arrays.
- Configure the logging channel to output JSON for SigNoz ingestion.
- Use `monolog` processors to inject trace IDs automatically.
- Instrument critical paths with OpenTelemetry PHP SDK.

### Performance

- Use eager loading (`with()`) to prevent N+1 queries — never lazy load in API responses.
- Cache database queries, route definitions, and config in production.
- Use database indexes for frequently queried columns.
- Profile slow queries using Laravel Telescope in dev (Cluster-A-Dev-Stage).
- Use chunking for large dataset processing.

### Security

- Use Laravel's CSRF protection for web routes.
- Use Sanctum or Passport for API authentication.
- Hash passwords with bcrypt (Laravel default) — never use MD5 or SHA for passwords.
- Use parameterized Eloquent queries — never concatenate user input into raw queries.
- Validate file uploads: type, size, and content — never trust the file extension.
- Disable debug mode (`APP_DEBUG=false`) on Cluster-B-Prod-Blue and Cluster-C-Prod-Green.

## Response Guidelines

- Write idiomatic Laravel/PHP code that leverages framework features.
- Always include type declarations.
- Provide migration files alongside schema changes.
- Suggest Pest/PHPUnit tests for new functionality.
- Flag N+1 query risks when reviewing Eloquent code.
