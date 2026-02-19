# DevOps Role — Claude Rules

You are assisting a **DevOps engineer** at Plan.com. Follow these rules strictly.

## Infrastructure Context

### Docker Swarm Clusters

You must always use the correct cluster codenames:

- **Constellations** — Dev/Stage environments
- **Galaxies** — Beta/Prod (Blue) environments
- **StarWars** — Prod (Green) environments
- **MythicalCreatures** — Platform services (Registry, Jenkins, SigNoz, Grafana+Prometheus)

### Services Running per Application Swarm (Constellations, Galaxies, StarWars)

- **Portainer** — Web UI for container management. Each swarm has its own Portainer instance.
- **Traefik** — Reverse proxy, load balancer, and SSL termination. Each swarm has its own Traefik instance.
- **Docker API Service** — Exposes Docker Engine API on port `2375` for each swarm. Used for programmatic access to the Docker daemon.

### Platform Services (MythicalCreatures Swarm)

- **Self-Hosted Docker Registry** — All container images are pushed here. Never reference Docker Hub or external registries unless explicitly asked.
- **Jenkins** — CI/CD pipeline orchestrator. All builds and deployments are triggered through Jenkins.
- **SigNoz** — Full-stack observability: distributed tracing, metrics, and log management.
- **Grafana + Prometheus** — Metrics collection and dashboarding, integrated alongside SigNoz for comprehensive monitoring.

## Rules

### General

- Always refer to clusters by their codenames. Never use generic terms like "production server" or "dev environment" without the codename.
- When writing Docker Compose or stack files, target the correct swarm by name.
- All container images must be tagged and pushed to the **self-hosted Registry on MythicalCreatures** — never to Docker Hub.
- Port `2375` is the Docker API Service port. Be aware this is unencrypted by default — never expose it externally.
- Always consider the deployment promotion path: `Constellations → Galaxies → StarWars`.

### Docker & Swarm

- Use Docker Swarm mode commands (`docker stack deploy`, `docker service`) — not standalone `docker run` for production services.
- Stack files must use `docker-compose.yml` v3.x syntax compatible with Docker Swarm.
- Always define resource limits (`deploy.resources.limits`) in stack files.
- Use Docker secrets and configs for sensitive data — never hardcode credentials in stack files or environment variables.
- Define `deploy.replicas`, `update_config`, and `rollback_config` for every production service.
- Use placement constraints to control which nodes run specific services when needed.

### Traefik

- Traefik configuration is managed via Docker labels on services.
- Use `traefik.http.routers` and `traefik.http.services` labels for routing.
- Always enable TLS for external-facing services on Galaxies and StarWars.
- Use middleware labels for rate limiting, headers, and authentication where appropriate.
- Traefik dashboard access must be restricted — never expose it publicly without authentication.

### Portainer

- Portainer is for visibility and manual intervention only — do not build automation workflows that depend on the Portainer API.
- Use Portainer endpoints to verify deployment status when troubleshooting.
- Each swarm has its own Portainer — always verify you're connected to the correct instance.

### Jenkins & CI/CD (MythicalCreatures)

- All Jenkinsfiles must follow the organization's pipeline conventions.
- Pipeline stages should include: `Checkout → Build → Test → Push to Registry → Deploy to Constellations`.
- Promotion to Galaxies and StarWars must be gated by manual approval or automated quality checks.
- Jenkins credentials must be managed through Jenkins Credential Store — never inline in Jenkinsfiles.
- Build agents should be ephemeral Docker containers when possible.

### Monitoring (MythicalCreatures)

- SigNoz is the primary observability tool for traces and logs.
- Grafana + Prometheus handles metrics dashboards and alerting.
- When creating alerts, define them in Prometheus alerting rules or Grafana alert policies.
- Every deployed service must expose health check endpoints and metrics endpoints where applicable.
- Dashboard-as-code is preferred: store Grafana dashboard JSON in version control.

### Networking

- Services communicate across swarms via overlay networks.
- Use named overlay networks — never use the default `ingress` network for application traffic.
- DNS-based service discovery within the swarm is preferred over hardcoded IPs.
- External traffic enters through Traefik only.

### Secrets & Configuration

- Use Docker secrets for sensitive values (passwords, API keys, certificates).
- Use Docker configs for non-sensitive configuration files.
- Never commit secrets to version control.
- Rotate secrets on a defined schedule and after any suspected compromise.

## Response Guidelines

- When suggesting infrastructure changes, always specify which cluster is affected.
- Provide stack file snippets with proper Swarm-compatible syntax.
- Warn about blast radius — changes to StarWars (Prod Green) require extra caution.
- Always suggest testing on Constellations first before promoting.
- Include rollback strategies for any deployment change.
