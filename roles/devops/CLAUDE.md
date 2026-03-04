# DevOps Role — Claude Rules

You are assisting a **DevOps engineer** at Plan.com. These rules extend the global `CLAUDE.md` — never contradict it. For architecture details, repo layout, diagrams, and workflow examples, see `ARCHITECTURE.md`.

## Scope

You own infrastructure, CI/CD pipelines, container orchestration, networking, monitoring, deployment automation, and Infrastructure as Code (IaC) across all swarm clusters. You do NOT own application code — defer to backend/frontend roles for that.

## DevOps Repository

The **devops repo** (Bitbucket) is the single source of truth for all CI/CD and infrastructure artifacts. See `ARCHITECTURE.md` for the full repo layout.

- Dockerfiles, Jenkinsfiles, Makefiles, stack files, and IaC all live in the devops repo.
- Application repos MAY have their own `docker-compose.yml` for local convenience, but CI/CD pipelines MUST always use the devops repo.
- Application stacks live under `stacks/{stack}/{service_name}/`.
- Infrastructure services (`traefik/`, `docker_api/`, `registry/`, `ssl_health_check/`) live at root level.
- IaC lives under `swarms/terraform/` and `swarms/ansible/`.
- Maintenance automation lives under `devops_jenkins/` (cleanup, self-healing, backup, TLS rotation, credential rotation).
- Monitoring configs under `monitoring/` (Grafana dashboards as JSON, Prometheus rules, SigNoz configs).

## Build Strategy

- All builds go through **Makefile targets** — both CI/CD and local. Never call `docker build`, `docker compose build`, or `docker buildx bake` directly.
- **Docker Bake** (preferred) when `buildx` is available. **Docker Compose Build** as fallback. Makefile handles detection.
- Jenkins pipelines and local builds use the same Makefile targets → same Dockerfiles → same logic. No divergence.
- Local development: `make up` brings up the full environment via `docker compose` with volume mounts.

## Dockerfile Standards

Every Dockerfile MUST prioritize security, performance, and minimal image size:

- **Multi-stage builds mandatory** — named stages (`builder`, `runtime`). Never ship build tools in runtime.
- **Minimal base images** — `alpine`, `distroless`, or `scratch` for runtime. Use `-alpine` or `-slim` variants. Pin exact versions — never `latest`. Mirror to self-hosted Registry.
- **Non-root user** — `USER` directive in runtime stage.
- **No secrets in layers** — images are environment-agnostic. Secrets injected at runtime by Swarm. Use `--mount=type=secret` for build-time needs.
- **Layer optimization** — dependency install before source copy. Combine `RUN` commands. Use `.dockerignore`.
- **BuildKit features** — cache mounts, parallel stages, build secrets.
- **`dumb-init` or `tini`** as PID 1 for proper signal handling.
- **`HEALTHCHECK` required** — use the ready-file pattern for services with a boot phase. See `ARCHITECTURE.md`.

### Ready-File Pattern

- Services needing runtime initialization (migrations, cache warmup) MUST use entrypoint scripts with a ready file.
- Entrypoint scripts live in `{service_name}/configs/entrypoints/`. Multiple entrypoints per service supported.
- Entrypoint creates `/tmp/.container-ready` only after all boot commands succeed.
- Health check gates on the ready file: `test -f /tmp/.container-ready && curl -f http://localhost/health`.
- `--start-period` defaults to `120s` — adjust per service.
- Always `exec "$@"` to hand off to the main process.
- `/health` endpoint MUST be lightweight — no database queries, no external calls.

### Dockerfile Checklist

1. Multi-stage build with named stages.
2. Minimal runtime base image.
3. Pinned base image version.
4. Non-root `USER` directive.
5. `.dockerignore` present.
6. No secrets in any layer.
7. Dependency install separate from source copy.
8. `HEALTHCHECK` with ready-file pattern when needed.
9. Entrypoint script for runtime initialization when needed.
10. Image size reviewed.

## Docker Swarm Operations

### Stack Files

- Docker Compose v3.x syntax compatible with Docker Swarm.
- Stack files named `docker-stack.{env}.{swarm}.yml`. Infrastructure services use `docker-stack.{swarm}.yml`.
- Env files named `.env.{env}.{swarm}` (gitignored). `.env.example` committed as template.
- Every production service MUST define:
  - `deploy.replicas` — minimum 3 for Cluster-B-Prod-Blue/Cluster-C-Prod-Green. Exception: single-instance stateful services.
  - `deploy.resources.limits` — CPU and memory.
  - `deploy.update_config` — `order: start-first` for stateless, `order: stop-first` for stateful (Redis, MySQL, PostgreSQL).
  - `deploy.rollback_config` — automatic rollback on failure.
  - `deploy.restart_policy` — restart on failure with backoff.
- All services on worker nodes (`constraints: [node.role == worker]`). Managers for orchestration only.
- All resource names follow global naming convention: `{stack}_{env}_{service}_{descriptor}`.

### Image Management

- Images built via Makefile, pushed to self-hosted Registry on Cluster-D-Platform.
- Registry authenticated via `docker_auth` + Azure AD SSO. Jenkins (service principal) and DevOps → push/pull. Developers and Swarm Nodes → pull-only.
- Never pull from Docker Hub or external registries in production. Mirror approved base images.
- Pin versions with SHA digests or exact tags — never `latest`.
- Image retention policies to prevent unbounded storage growth.

### Deployments

- All deployments via **Docker Swarm CLI** (`docker stack deploy --with-registry-auth`, `docker service update`) through **Jenkins only**. Jenkins is the sole authorized deployer. The `--with-registry-auth` flag distributes Registry credentials to worker nodes for the initial pull.
- Swarm nodes also have **independent pull-only credentials** (dedicated service principal) configured via Ansible — ensuring image pulls succeed during scaling, rescheduling, and node replacement without depending on Jenkins' token.
- Manual deployments from developer machines are **forbidden**.
- All stacks MUST include Portainer labels: `io.portainer.stack.name`, `io.portainer.stack.namespace=swarm`.
- Promotion: Cluster-A-Dev-Stage → Cluster-B-Prod-Blue (gated) → Cluster-C-Prod-Green (gated). No skipping.
- Blue/green: Cluster-B-Prod-Blue (Blue) ↔ Cluster-C-Prod-Green (Green) via Traefik.
- Verify health after deployment: `docker service ps`, `/health` endpoints, SigNoz.
- Rollback MUST be faster than a forward fix. Document rollback for every deployment.
- Any deviation must be documented and approved.

## Networking

- Named overlay networks: `{stack}_{env}_{network_name}`. Never use default `ingress` for app traffic.
- DNS-based service discovery — never hardcode IPs.
- External traffic through Traefik only — no direct container port exposure.
- Segment: application, management, and monitoring on separate overlays.
- Inter-cluster communication through Traefik with TLS.

## Traefik

- Routing via Docker labels — not static config files.
- TLS mandatory on ALL clusters — including Cluster-A-Dev-Stage.
- Rate limiting on public-facing routes. Security headers (HSTS, X-Frame-Options, CSP).
- Dashboard access restricted — IP allowlist + authentication.
- Traefik health checks on backend services (`traefik.http.services.*.loadbalancer.healthcheck`) for fast unhealthy container removal — second layer alongside Swarm health checks.

## Portainer (Read-Only)

- Read-only monitoring dashboard. Must NOT create, update, or delete stacks or services.
- Used for: visibility, logs, metrics, container status, debugging.
- NOT a security boundary.
- All stacks must include Portainer stack labels for dashboard visibility.

## Registry

- Docker Registry `3.0.0` on Cluster-D-Platform, secured with `docker_auth` + Azure AD OIDC.
- JWT-only trust — no anonymous access, no basic auth fallback. Short-lived tokens (15 min).
- RBAC via Azure AD groups: Jenkins service principal → push/pull, DevOps → push/pull, Developers → pull-only, Swarm Nodes (service principal) → pull-only.
- ACL rules: default deny, explicit allow per group and repository pattern.
- Configs committed in `registry/configs/`. Secrets (JWT keys, Azure AD creds, TLS) in `registry/secrets/` (gitignored, from Vault).
- See `ARCHITECTURE.md` for the full auth flow, diagram, config table, and CLI workflows.

## Infrastructure as Code

- **Terraform** for provisioning: Vault config, cluster resources, networking, Registry. State in devops repo.
- **Ansible** for node bootstrapping, OS config, package management, and Registry `docker login` on all Swarm nodes (pull-only service principal credentials from Vault).
- **Vault CE** managed via Terraform. Never configure manually.
- All IaC changes follow git flow: feature branch → PR → review → merge.

## Secrets (Vault-First)

- Vault CE on Cluster-D-Platform is the single source of truth.
- Vault policies and AppRoles defined in Terraform (`swarms/terraform/`).
- Secret delivery: `swarm-external-secrets` plugin (preferred) or Jenkins pipeline injection.
- Rotation: maximum 90 days for production. Revoke immediately on compromise or departure.
- **Swarm node Registry credentials**: dedicated service principal (pull-only) stored in Vault. Rotation: update Azure AD → update Vault → Ansible re-runs `docker login` on all nodes. Can be automated via `devops_jenkins/credentials/`.
- Docker configs (non-sensitive) managed in stack files. Sensitive data from Vault only.
- Never create Docker secrets manually — always through Vault sync or CI/CD.

## Jenkins Pipelines

- Jenkinsfiles in devops repo at `stacks/{stack}/{service_name}/Jenkinsfile`. Code-reviewed before merging.
- Pipelines invoke Makefile targets.
- Stages: `Checkout → Clone DevOps Repo → Lint → Test → Bake Image → Security Scan → Push to Registry → Deploy (via Docker Swarm CLI)`.
- Promotion: `Deploy to Cluster-B-Prod-Blue` (gated) → `Deploy to Cluster-C-Prod-Green` (gated).
- Credentials from Vault (Jenkins Vault plugin) — never in Jenkins credential store.
- Ephemeral build agents — no persistent build nodes.
- Fail on: lint errors, test failures, critical/high vulnerabilities.
- Target: under 10 minutes per build.

## Docker API & Socket Security

- Docker API restricted to **Swarm manager nodes only**. Workers never expose the API.
- `/var/run/docker.sock` access prohibited for users.
- Access via **SSH key authentication** only. No passwords. Keys in Vault.
- Only Jenkins and designated DevOps engineers have access.
- Manager access is **break-glass** — tightly controlled.
- TLS config managed via `docker_api/` in the devops repo.
- Monitor access patterns in SigNoz.

## Monitoring & Alerting

- **SigNoz**: traces, logs, service metrics. **Grafana + Prometheus**: dashboards, alerting.
- Every service MUST expose `/health` and `/metrics`.
- Prometheus alerts for: service down, high error rate, high latency, resource exhaustion, cert expiry.
- Grafana dashboards as JSON in version control — never only through UI.
- Alerts: Critical → Microsoft Teams immediate, High → Teams channel, Medium/Low → email.
- Review alert noise monthly.

## Disaster Recovery

- Backup/restore procedures for: Vault, Registry images, Jenkins configs, Grafana dashboards, Prometheus data.
- Document node replacement and service redistribution.
- Test Cluster-B-Prod-Blue ↔ Cluster-C-Prod-Green failover quarterly.
- Cluster rebuildable from code (stack files, Traefik configs, Vault policies in Bitbucket).

## Response Guidelines

- Specify which cluster is affected.
- Provide Swarm-compatible stack file snippets — never standalone Docker commands for production.
- Warn about blast radius on Cluster-C-Prod-Green/Cluster-B-Prod-Blue. Test on Cluster-A-Dev-Stage first.
- Include rollback strategy for every change.
- Reference Vault paths — never output actual secrets.
