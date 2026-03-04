# DevOps — Architecture Reference

This document describes **how** Plan.com's infrastructure is organized. For behavioral rules, see `CLAUDE.md`.

## DevOps Repository Layout

The devops repo is organized into top-level concerns. Application stack build/deploy logic is isolated under `stacks/` while infrastructure services, cluster management, maintenance automation, and observability each have their own top-level folder.

```
devops/
├── .gitignore                           ← Global git ignore rules
├── .dockerignore                        ← Global Docker build context exclusions
├── README.md
│
├── stacks/                              ← Application stacks — build, deploy, and runtime artifacts
│   └── {stack}/                         ← One folder per stack (e.g., auth, billing, gateway)
│       ├── Makefile                     ← Build/run/deploy targets for this stack
│       ├── docker-compose.yml           ← Local dev + fallback build definitions
│       ├── docker-bake.hcl              ← Bake build definitions (when buildx available)
│       ├── docker-stack.dev.cluster-a-dev-stage.yml   ← Swarm stack — Dev
│       ├── docker-stack.stg.cluster-a-dev-stage.yml   ← Swarm stack — Staging
│       ├── docker-stack.uat.cluster-b-prod-blue.yml         ← Swarm stack — UAT
│       ├── docker-stack.demo.cluster-b-prod-blue.yml        ← Swarm stack — Demo
│       ├── docker-stack.prod.cluster-b-prod-blue.yml        ← Swarm stack — Production Blue
│       ├── docker-stack.prod.cluster-c-prod-green.yml        ← Swarm stack — Production Green
│       ├── .env.example                 ← Template with all required vars (committed)
│       ├── .env.dev.cluster-a-dev-stage       ← Dev values (gitignored)
│       ├── .env.stg.cluster-a-dev-stage       ← Staging values (gitignored)
│       ├── .env.uat.cluster-b-prod-blue             ← UAT values (gitignored)
│       ├── .env.demo.cluster-b-prod-blue            ← Demo values (gitignored)
│       ├── .env.prod.cluster-b-prod-blue            ← Production Blue values (gitignored)
│       ├── .env.prod.cluster-c-prod-green            ← Production Green values (gitignored)
│       └── {service_name}/              ← One folder per service in the stack
│           ├── Jenkinsfile              ← CI/CD pipeline for this service (committed)
│           ├── Dockerfile               ← Official Dockerfile (committed)
│           ├── .dockerignore            ← Service-level build context exclusions (committed)
│           ├── configs/                 ← Runtime configs (committed)
│           │   └── entrypoints/         ← Boot phase scripts (committed)
│           ├── src/                     ← Cloned application repo (gitignored)
│           ├── secrets/                 ← Local dev secrets (gitignored)
│           ├── logs/                    ← Build log history (gitignored)
│           └── build_info/
│               └── build-info.json      ← Last build metadata (committed)
│
├── traefik/                             ← Traefik reverse proxy — per-swarm setup
│   ├── configs/                         ← Static/dynamic config, middleware definitions
│   ├── secrets/                         ← TLS certificates, private keys (gitignored)
│   ├── docker-stack.cluster-a-dev-stage.yml  ← Traefik stack — Cluster-A-Dev-Stage
│   ├── docker-stack.cluster-b-prod-blue.yml        ← Traefik stack — Cluster-B-Prod-Blue
│   └── docker-stack.cluster-c-prod-green.yml        ← Traefik stack — Cluster-C-Prod-Green
│
├── docker_api/                          ← Docker API security and TLS configuration
│   ├── configs/                         ← daemon.json templates, TLS config
│   ├── secrets/                         ← Client/server certificates (gitignored)
│   ├── docker-stack.cluster-a-dev-stage.yml  ← Docker API stack — Cluster-A-Dev-Stage
│   ├── docker-stack.cluster-b-prod-blue.yml        ← Docker API stack — Cluster-B-Prod-Blue
│   └── docker-stack.cluster-c-prod-green.yml        ← Docker API stack — Cluster-C-Prod-Green
│
├── registry/                            ← Docker Registry + docker_auth (Cluster-D-Platform)
│   ├── configs/                         ← Registry config, docker_auth config, ACL rules
│   ├── secrets/                         ← TLS certs, JWT keys, Azure AD credentials (gitignored)
│   └── docker-stack.cluster-d-platform.yml ← Registry stack — Cluster-D-Platform
│
├── ssl_health_check/                    ← SSL expiration monitoring per swarm
│   ├── configs/                         ← Domain lists, health check config per cluster
│   ├── secrets/                         ← Credentials if needed (gitignored)
│   ├── docker-stack.cluster-a-dev-stage.yml  ← Health check stack — Cluster-A-Dev-Stage
│   ├── docker-stack.cluster-b-prod-blue.yml        ← Health check stack — Cluster-B-Prod-Blue
│   └── docker-stack.cluster-c-prod-green.yml        ← Health check stack — Cluster-C-Prod-Green
│
├── devops_jenkins/                      ← Maintenance cronjobs and automation
│   ├── cleanup/                         ← Disk space cleanup on swarm nodes
│   ├── self_healing/                    ← Auto-scale services below intended replication
│   ├── backup/                          ← Database backups, folder backups
│   ├── tls/                             ← TLS certificate rotation
│   └── credentials/                     ← Service credential rotation (e.g., Swarm node Registry login)
│
├── swarms/                              ← Cluster infrastructure management
│   ├── terraform/                       ← Provisioning: Vault, Registry, networking, nodes
│   └── ansible/                         ← Node bootstrapping, OS config, package management
│
├── monitoring/                          ← Observability as code
│   ├── grafana/                         ← Dashboard JSON exports (version-controlled)
│   ├── prometheus/                      ← Alert rules, recording rules, scrape configs
│   └── signoz/                          ← SigNoz dashboards and alert configurations
│
└── scripts/                             ← Global helper scripts and utilities
    ├── rotate-secrets.sh                ← Vault secret rotation helpers
    ├── registry-cleanup.sh              ← Image retention / garbage collection
    └── ...                              ← Migration tools, health checks, etc.
```

### Top-Level Folders

| Folder | Purpose |
|--------|---------|
| `stacks/` | Application stack build/deploy — one subfolder per stack |
| `traefik/` | Traefik reverse proxy setup — deployed on every cluster before any app stack |
| `docker_api/` | Docker API TLS configuration — securing the Docker socket per cluster |
| `registry/` | Docker Registry + docker_auth with Azure AD SSO and RBAC — runs on Cluster-D-Platform |
| `ssl_health_check/` | SSL expiration monitoring — health check DNS per swarm for certificate rotation visibility |
| `devops_jenkins/` | Maintenance cronjobs — cleanup, self-healing, backups, TLS rotation |
| `swarms/` | Cluster infrastructure — Terraform provisioning and Ansible configuration |
| `monitoring/` | Grafana dashboards (JSON), Prometheus alert/recording rules, SigNoz configs |
| `scripts/` | Global helper scripts — secret rotation, registry cleanup, migration tools |

### Infrastructure Services (Pre-Stack)

`traefik/`, `docker_api/`, `registry/`, and `ssl_health_check/` are infrastructure services that must be deployed before any application stack. Setup order:

1. **Docker API** — Secure the Docker socket with TLS (migrate from port 2375 to 2376).
2. **Registry** — Deploy the Docker Registry with `docker_auth` on Cluster-D-Platform. Configure Azure AD SSO, RBAC, and JWT token auth.
3. **Traefik** — Deploy the reverse proxy with TLS certificates (stored as Docker secrets via Vault), middleware, and routing rules on each swarm.
4. **SSL Health Check** — Deploy the health check service with DNS entries for the cluster (e.g., `cluster-a-dev-stage-health-check.company.com`). Monitors SSL certificate expiration across all domains configured in Traefik.

### Maintenance Automation (`devops_jenkins/`)

| Category | Purpose | Examples |
|----------|---------|----------|
| `cleanup/` | Disk space management on swarm nodes | Prune unused images, volumes, build cache |
| `self_healing/` | Auto-scale recovery for services below intended replication | Detect 3/5 running containers → restore to 5/5 |
| `backup/` | Scheduled backups | Database dumps, config folder snapshots |
| `tls/` | TLS certificate lifecycle | Rotate certificates with new ones from Vault or ACME |
| `credentials/` | Service credential rotation | Rotate Swarm node Registry credentials via Ansible |

### Stack-Level Files (`stacks/{stack}/`)

- **`Makefile`** — All build, run, deploy, and utility commands. Engineers and Jenkins interact through `make` targets.
- **`docker-compose.yml`** — Local development services and fallback build strategy when `buildx` is not available.
- **`docker-bake.hcl`** — Preferred build definition when `docker buildx` is available.
- **`docker-stack.{env}.{swarm}.yml`** — Swarm stack deployment file per environment and cluster. Pattern: `docker-stack.dev.cluster-a-dev-stage.yml`, `docker-stack.prod.cluster-b-prod-blue.yml`, `docker-stack.prod.cluster-c-prod-green.yml`.
- **`.env.example`** — Template with all required variables. **Committed**. Copy to `.env.{env}.{swarm}` and populate from Vault.
- **`.env.{env}.{swarm}`** — Actual values. **Gitignored**. Pattern: `.env.dev.cluster-a-dev-stage`, `.env.prod.cluster-b-prod-blue`, `.env.prod.cluster-c-prod-green`.

### Service-Level Files (`stacks/{stack}/{service_name}/`)

- **`Jenkinsfile`** — CI/CD pipeline. Jenkins points to this file in the devops repo — never application repos.
- **`Dockerfile`** — Official Dockerfile. Must follow Dockerfile Standards in CLAUDE.md.
- **`.dockerignore`** — Build context exclusions. Extends the global `.dockerignore`.
- **`configs/`** — Runtime configs: nginx, PHP-FPM, cron, etc. Part of the build or mounted as Docker configs.
  - **`configs/entrypoints/`** — Boot phase scripts (e.g., `entrypoint.sh`, `worker-entrypoint.sh`). Multiple entrypoints per service supported.
- **`src/`** — Cloned application repo. Mounted as volume locally, copied during builds. **Gitignored**.
- **`secrets/`** — Local dev secrets. **Gitignored**. Production secrets from Vault.
- **`build_info/build-info.json`** — Last build metadata. **Committed**.
- **`logs/`** — Build logs. **Gitignored**. Naming: `{service_name}-YYYY-MM-DD-HH-mm-ss.log`.

### Global `.gitignore`

```gitignore
# Application source (cloned repos)
**/src/

# Local secrets
**/secrets/

# Environment files (keep example)
.env*
!.env.example

# Build logs
**/logs/

# OS / IDE
.DS_Store
.idea/
.vscode/
*.swp
```

### Global `.dockerignore`

```dockerignore
# Infrastructure files — not needed inside application containers
Dockerfile
Jenkinsfile
Makefile
docker-compose*.yml
docker-stack*.yml
docker-bake*.hcl
docker-bake*.json

# Git and CI/CD
.git
.gitignore

# Environment files
.env*

# Build logs
**/logs/

# Documentation
README.md
*.md
LICENSE

# OS / IDE
.DS_Store
.idea/
.vscode/
*.swp
```

## Entrypoint & Health Check — Ready-File Pattern

Since images are built without secrets or environment variables (injected at runtime by Swarm), services that need a boot phase use the ready-file pattern.

**Entrypoint script** (`configs/entrypoints/entrypoint.sh`):

```bash
#!/bin/sh
set -e

# ─── Boot phase (secrets and envs are available at runtime) ───
# Run migrations, cache warmup, config compilation, etc.

# ─── Signal ready ───
touch /tmp/.container-ready

# ─── Hand off to main process (PID 1) ───
exec "$@"
```

**Dockerfile HEALTHCHECK**:

```dockerfile
COPY configs/entrypoints/ /entrypoints/
RUN chmod +x /entrypoints/*.sh

ENTRYPOINT ["/entrypoints/entrypoint.sh"]

HEALTHCHECK --interval=30s --timeout=5s --start-period=120s --retries=3 \
  CMD test -f /tmp/.container-ready && curl -f http://localhost/health || exit 1
```

**Flow:**

1. Container starts → Swarm injects secrets and env vars → entrypoint runs.
2. Entrypoint executes boot commands — health check failures during `--start-period` do NOT count toward retries.
3. Entrypoint creates `/tmp/.container-ready` — signals boot phase complete.
4. Entrypoint `exec`s the main process.
5. Health check passes: ready file exists AND `/health` responds → Swarm marks healthy → Traefik routes traffic.

## Registry — Authentication Architecture

The self-hosted Docker Registry (`registry:3.0.0`) on Cluster-D-Platform is secured using `docker_auth` as a token service integrated with Azure AD (Microsoft SSO) via OIDC.

### Architecture Diagram

```
┌──────────────┐     ┌───────────────┐     ┌──────────────────┐     ┌──────────────┐
│  User /      │────▶│  docker_auth  │────▶│  Azure AD (OIDC) │     │   Docker     │
│  Jenkins /   │     │  Token Service│◀────│  Microsoft SSO   │     │   Registry   │
│  Swarm Node  │     └───────┬───────┘     └──────────────────┘     │   3.0.0      │
│              │             │ JWT token                             │              │
│              │─────────────┼─────────────────────────────────────▶│              │
│              │             │ (Bearer token in request)             │              │
└──────────────┘     ┌───────┴───────┐                              └──────────────┘
                     │  ACL Rules    │
                     │  (RBAC)       │
                     └───────────────┘
```

### Authentication Flow

1. **`docker login`** — User, Jenkins, or Swarm node sends credentials to the Registry.
2. **Registry redirects** — Registry returns a `401` with a `Www-Authenticate` header pointing to `docker_auth`.
3. **Token request** — Docker client requests a token from `docker_auth` with the requested scope (e.g., `repository:auth_prod_api:pull,push`).
4. **OIDC authentication** — `docker_auth` authenticates the user via Azure AD using OIDC. For service principals (Jenkins, Swarm Nodes), it validates client credentials.
5. **ACL evaluation** — `docker_auth` checks ACL rules to determine if the identity has the requested permissions.
6. **JWT token issued** — If authorized, `docker_auth` signs and returns a JWT token with the granted scopes.
7. **Registry validates** — Docker client sends the JWT to the Registry. The Registry validates the signature against `docker_auth`'s public key and serves the request.

### RBAC Model

| Role | Azure AD Group | Permissions | Use Case |
|------|---------------|-------------|----------|
| CI/CD (Jenkins) | Service Principal | `push` + `pull` | Build and push images in pipelines |
| DevOps Team | DevOps AD Group | `push` + `pull` | Manual image builds (hotfixes, debugging) |
| Developers | Developers AD Group | `pull` only | Pull images for local development |
| Swarm Nodes | Service Principal | `pull` only | Image pulls during deploy, scaling, rescheduling, and node replacement |

### Required Configurations

| File | Location | Purpose |
|------|----------|---------|
| `docker_auth` config | `registry/configs/auth_config.yml` | OIDC settings, ACL rules, JWT signing, server config |
| Registry config | `registry/configs/registry_config.yml` | Storage backend, token auth endpoint, TLS settings |
| JWT signing key | `registry/secrets/` | Private key for `docker_auth` to sign tokens (gitignored) |
| JWT public key | `registry/configs/` | Public key for Registry to verify tokens (committed) |
| Azure AD credentials | Vault | Client ID and client secret for the OIDC application |
| TLS certificates | `registry/secrets/` | HTTPS certs for both Registry and `docker_auth` (gitignored) |
| Stack file | `registry/docker-stack.cluster-d-platform.yml` | Swarm deployment: Registry + `docker_auth` services |

### Jenkins Workflow

```bash
# Jenkins pipeline authenticates using service principal credentials from Vault
docker login registry.cluster-d-platform.company.com \
  --username "$AZURE_CLIENT_ID" \
  --password "$AZURE_CLIENT_SECRET"

# Build and push (authorized by ACL: service principal → push + pull)
make build-api
docker push registry.cluster-d-platform.company.com/auth_prod_api:1.2.3
```

### Developer Workflow

```bash
# Developer authenticates via Azure AD SSO (browser opens for login)
docker login registry.cluster-d-platform.company.com

# Pull only (authorized by ACL: developers group → pull only)
docker pull registry.cluster-d-platform.company.com/auth_dev_api:latest

# Push attempt → denied by docker_auth ACL
docker push ... # 403 Forbidden
```

### Swarm Node Authentication

Swarm nodes need to pull images independently — during scaling, rescheduling, self-healing, and node replacement. Since JWT tokens are short-lived (15 min), the `--with-registry-auth` token from Jenkins expires long before these events occur.

**Solution:** Each Swarm node has a dedicated pull-only service principal configured via Ansible. The node stores credentials (client ID + secret) in `~/.docker/config.json`. On every pull, the Docker client contacts `docker_auth` with these credentials, gets a fresh JWT, and pulls the image. JWT stays short-lived because a new one is requested on each pull.

**Node provisioning (Ansible):**

```bash
# Ansible runs docker login on each Swarm node during provisioning
# Credentials come from Vault (pull-only service principal)
docker login registry.cluster-d-platform.company.com \
  --username "$SWARM_NODE_CLIENT_ID" \
  --password "$SWARM_NODE_CLIENT_SECRET"
```

**Credential rotation flow:**

1. Rotate the service principal secret in Azure AD.
2. Update the new secret in Vault.
3. Ansible re-runs `docker login` on all Swarm nodes (automated via `devops_jenkins/credentials/`).
4. Nodes get fresh JWTs on next pull using the new credentials.

**Why `--with-registry-auth` alone is not enough:**

| Scenario | `--with-registry-auth` | Node-level `docker login` |
|----------|----------------------|--------------------------|
| Initial deployment | Works (token is fresh) | Works |
| Scale up (hours later) | Fails (JWT expired) | Works (fresh JWT on pull) |
| Node rescheduling | Fails (JWT expired) | Works |
| New node joins cluster | No token available | Works (Ansible provisions) |
| Self-healing (container restart) | Fails (JWT expired) | Works |
