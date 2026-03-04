# Backend Developer Role — Python Stack — Claude Rules

You are assisting a **Backend Developer working with Python** at Plan.com. Follow the general backend rules in `roles/backend/CLAUDE.md` plus these Python-specific rules.

## Stack

- **Language**: Python 3.11+
- **Frameworks**: FastAPI (preferred), Django, Flask — follow the project's established choice
- **Package Manager**: pip with `requirements.txt` / `pyproject.toml`, or Poetry
- **Testing**: pytest
- **Linting**: Ruff (preferred), flake8, pylint
- **Formatting**: Ruff formatter or Black
- **Type Checking**: mypy or pyright
- **Containerized**: Docker multi-stage builds, deployed to Plan.com swarm clusters

## Rules

### Python Standards

- Use type hints for all function signatures and class attributes.
- Follow PEP 8 style guidelines — enforced by Ruff or Black.
- Use `dataclasses` or Pydantic models for data structures — avoid raw dicts for structured data.
- Prefer f-strings for string formatting.
- Use `pathlib.Path` instead of `os.path` for file system operations.
- Use context managers (`with` statements) for resource management.
- Avoid mutable default arguments.
- Use `__all__` to define public module APIs.

### FastAPI-Specific

- Define request/response models with Pydantic — full type validation at the API boundary.
- Use dependency injection for shared resources (database sessions, auth, configs).
- Use `async def` for endpoints that perform I/O operations.
- Define path operations with explicit response models and status codes.
- Use `BackgroundTasks` for fire-and-forget operations.
- Use APIRouter for organizing endpoints by domain.
- Enable automatic OpenAPI documentation — keep schemas accurate and descriptive.
- Use lifespan context managers for startup/shutdown logic.

### Django-Specific (where applicable)

- Use Django REST Framework for API endpoints.
- Define models with proper field types, validators, and indexes.
- Use Django migrations for all schema changes.
- Use Django's ORM querysets — avoid raw SQL unless performance-critical.
- Use serializers for input validation and output transformation.
- Use class-based views for CRUD operations, function-based for custom logic.
- Cache with Django's cache framework — use Redis backend in production.

### Package Management

- Pin all dependencies with exact versions in lock files.
- Use `pip-audit` or `safety` for vulnerability scanning.
- Separate dev and production dependencies.
- Use virtual environments — never install packages globally in containers.
- If using Poetry, commit `poetry.lock`.

### Testing

- Use pytest as the test runner — never unittest directly.
- Use fixtures for test setup and shared resources.
- Use `pytest-asyncio` for testing async code.
- Use `httpx.AsyncClient` (FastAPI) or Django's test client for endpoint tests.
- Use factories (`factory_boy`) for generating test data.
- Mock external services — use `pytest-mock` or `unittest.mock`.
- Use `testcontainers` for database integration tests.

### Docker & Deployment

- Use `python:3.11-slim` or equivalent from the **self-hosted Registry** as base image.
- Multi-stage build: install dependencies in build stage, copy only necessary packages to runtime.
- Use `gunicorn` with `uvicorn` workers for FastAPI in production.
- Set `PYTHONDONTWRITEBYTECODE=1` and `PYTHONUNBUFFERED=1` in Dockerfile.
- Create a non-root user to run the application.
- Use pip's `--no-cache-dir` in Docker builds.
- Health check endpoint must be lightweight and dependency-aware (check DB connection, etc.).

### Logging & Observability

- Use `structlog` or `python-json-logger` for structured JSON logging.
- Configure logging to output to stdout/stderr for Docker log collection.
- Integrate OpenTelemetry Python SDK for tracing to SigNoz.
- Include `trace_id`, `span_id`, and `service_name` in log output.
- Use Prometheus client (`prometheus_client`) for `/metrics` endpoint.
- Use `logging` module — never use `print()` for application logs.

### Performance

- Use async frameworks (FastAPI + async ORMs) for I/O-bound services.
- Use connection pooling (SQLAlchemy pool, asyncpg pool).
- Profile with `cProfile` or `py-spy` on Cluster-A-Dev-Stage — never in production.
- Use Redis for caching hot data.
- Process large datasets with generators/iterators — avoid loading everything into memory.
- Use Celery or `arq` for background job processing.

### Security

- Use Pydantic for input validation — never trust raw user input.
- Use parameterized queries with SQLAlchemy or Django ORM — never use f-strings in SQL.
- Use `python-jose` or `PyJWT` for JWT handling.
- Never use `pickle` to deserialize untrusted data.
- Set `DEBUG=False` on Cluster-B-Prod-Blue and Cluster-C-Prod-Green.
- Use `secrets` module for generating tokens — never `random`.

## Response Guidelines

- Write idiomatic Python with full type annotations.
- Use Pydantic models for data validation and serialization.
- Include pytest tests alongside new functionality.
- Provide Dockerfile snippets for new services.
- Flag potential async pitfalls (blocking calls in async context).
