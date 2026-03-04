# Plan.com — Global Rules

These rules apply to ALL agents and roles. Role-specific rules extend but NEVER override these.

## Git Flow

### Branch Naming

Every branch MUST include a Jira ticket ID:

```
feature/TICKET-123-optional-description
chore/TICKET-123-optional-description
bugfix/TICKET-123-optional-description
hotfix/TICKET-123-optional-description
release/vX.Y.Z
```

### Merge Rules

- `main` is protected. Only `release/*` and `hotfix/*` branches merge into `main`.
- `feature/*`, `chore/*`, and `bugfix/*` branches merge into a `release/*` branch.
- Every release follows the deployment pipeline: Dev/Staging → UAT/Demo/Prod Blue → Prod Green.
- Hotfixes bypass the release cycle but MUST be backported into the current release branch.

### Commits

- Every commit message MUST reference the Jira ticket: `[TICKET-123] Short description`.
- Use imperative mood: "Add feature", not "Added feature".
- Keep the first line under 72 characters.

## Infrastructure Context

### Docker Swarm Clusters

| Cluster | Stacks | Purpose |
|---------|--------|---------|
| **Cluster-A-Dev-Stage** | Dev, Staging | Development and staging workloads |
| **Cluster-B-Prod-Blue** | UAT, Demo, Production (Blue) | User acceptance testing, demos, and production blue |
| **Cluster-C-Prod-Green** | Production (Green) | Blue/green deployment target for live traffic |
| **Cluster-D-Platform** | Platform Services | Registry, CI/CD, monitoring, Vault |

### Core Services

- **Orchestration**: Docker Swarm across all clusters with Traefik (reverse proxy, TLS) and Portainer (read-only monitoring). Services run on worker nodes — managers handle orchestration only.
- **Registry**: Self-hosted Docker Registry on Cluster-D-Platform with `docker_auth` + Azure AD SSO. Swarm nodes authenticate independently (pull-only) for image pulls during scaling and rescheduling. All images stored here — no external registries.
- **Version Control**: Bitbucket. PRs require approval before merging.
- **CI/CD**: Jenkins (Cluster-D-Platform) is the **sole authorized deployer**. Builds images via Makefile targets from the devops repo, pushes to Registry, deploys via Docker Swarm CLI. Manual deployments forbidden. Pipelines triggered by Bitbucket webhooks.
- **DevOps Repo**: Single source of truth for Dockerfiles, Jenkinsfiles, stack files, Makefiles, and IaC. See `roles/devops/CLAUDE.md` for the full layout.
- **Secrets**: Vault CE (Cluster-D-Platform) — single source of truth. No secrets in source control, `.env` files, or image layers. Delivered via `swarm-external-secrets` or Jenkins pipeline injection.
- **Monitoring**: SigNoz (traces/logs) + Grafana/Prometheus (dashboards/alerting) on Cluster-D-Platform. All services MUST export telemetry. Alerts: Critical/High → Microsoft Teams, Medium/Low → email.

## Naming Conventions — Docker Resources

All Docker resources (stacks, services, configs, secrets, volumes, networks) MUST include the environment suffix to prevent collisions and ensure clarity across clusters.

### Environment Suffixes

| Suffix | Cluster | Notes |
|--------|---------|-------|
| `dev` | Cluster-A-Dev-Stage | Development |
| `stg` | Cluster-A-Dev-Stage | Staging |
| `uat` | Cluster-B-Prod-Blue | User acceptance testing |
| `demo` | Cluster-B-Prod-Blue | Demo / showcase |
| `prod` | Cluster-B-Prod-Blue (Blue) / Cluster-C-Prod-Green (Green) | Production (blue/green) |

### Pattern

```
Format: {stack}_{env}_{service}_{descriptor}

Stack:    {stack}_{env}                            → auth_dev
Service:  {stack}_{env}_{service}                  → auth_dev_api
Config:   {stack}_{env}_{service}_{config_name}    → auth_dev_api_nginx_conf
Secret:   {stack}_{env}_{service}_{secret_name}    → auth_dev_api_db_password
Volume:   {stack}_{env}_{service}_{volume_name}    → auth_dev_api_uploaded_files
Network:  {stack}_{env}_{network_name}             → auth_dev_backend_network
```

### Rules

- Use **lowercase** and **underscores** only — no hyphens, no camelCase.
- The environment suffix is MANDATORY — even on Cluster-A-Dev-Stage where dev and stg share a cluster.
- Descriptors must be meaningful: `db_password` not `secret1`, `nginx_conf` not `config`.
- When a stack runs on multiple environments in the same cluster, the suffix is the only way to tell them apart — never omit it.

### File Naming Conventions

Stack files and environment files include both the environment and the swarm cluster for full clarity:

```
Stack files:  docker-stack.{env}.{swarm}.yml     → docker-stack.dev.cluster-a-dev-stage.yml
                                                  → docker-stack.prod.cluster-b-prod-blue.yml
                                                  → docker-stack.prod.cluster-c-prod-green.yml

Env files:    .env.{env}.{swarm}                  → .env.dev.cluster-a-dev-stage
                                                  → .env.prod.cluster-b-prod-blue
                                                  → .env.prod.cluster-c-prod-green

Template:     .env.example                        → Committed, lists all required vars
```

Infrastructure services (`traefik/`, `docker_api/`, `ssl_health_check/`) use per-swarm stack files since they run a single version per cluster: `docker-stack.{swarm}.yml` (e.g., `docker-stack.cluster-a-dev-stage.yml`).

## Security — Non-Negotiable

1. **Never output sensitive data**: No secrets, tokens, passwords, API keys, private IPs, or credentials in logs, responses, comments, or code output. Mask or reference Vault paths instead.
2. **Never commit secrets**: No `.env` files, hardcoded credentials, or API keys in source control. Use `.gitignore` to enforce this.
3. **Docker image scanning**: Every build MUST include a container security scan (e.g., Trivy, Grype, or Snyk). Fail the build on critical/high vulnerabilities.
4. **Dependency scanning**: Run dependency vulnerability checks (e.g., `npm audit`, `pip audit`, `govulncheck`, Composer audit) in CI. Fail on critical/high.
5. **Input validation**: Always validate and sanitize user input at system boundaries. Use parameterized queries — never string concatenation for SQL/NoSQL.
6. **Least privilege**: Containers run as non-root. Services get minimal permissions. Network access is restricted by default.
7. **TLS everywhere**: TLS is mandatory on ALL clusters — including Cluster-A-Dev-Stage. No plain HTTP for any communication, internal or external.

## Performance — Universal Standards

1. **No premature optimization** — but never ignore obvious inefficiencies (N+1 queries, unbounded loops, missing indexes, loading entire datasets into memory).
2. **Pagination**: Every list endpoint MUST support pagination. Never return unbounded result sets.
3. **Caching**: Consider caching at appropriate layers (application, CDN, reverse proxy). Invalidation strategy MUST be defined before implementing cache.
4. **Async by default**: Use async/non-blocking patterns for I/O-bound operations. Offload heavy processing to background jobs/queues.
5. **Resource limits**: Every container MUST define CPU and memory limits. Every database query MUST have a timeout.

## Code Quality — Universal

1. **Linting and formatting**: Every project MUST have a linter and formatter configured. CI MUST enforce them — no exceptions.
2. **Tests are required**: Every change MUST include relevant tests. Minimum: unit tests. Integration tests for API endpoints and critical flows.
3. **No dead code**: Remove unused imports, variables, functions, and files. Do not comment out code — delete it.
4. **Meaningful naming**: Variables, functions, and classes must be self-descriptive. No single-letter names outside of loop counters.
5. **Error handling**: Never swallow errors silently. Log with structured format (JSON preferred) and appropriate severity levels. Propagate errors to callers with context.
6. **Documentation**: Public APIs MUST have documentation. Complex business logic MUST have inline comments explaining *why*, not *what*.

## Role-Specific Rules

Each project MUST include the relevant role-specific `CLAUDE.md` from `roles/`. Stack-specific conventions (framework patterns, dependency choices, Dockerfile templates) live there — not here.
