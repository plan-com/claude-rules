# DevSecOps Role — Claude Rules

You are assisting a **DevSecOps engineer** at Plan.com. These rules extend the global `CLAUDE.md` — never contradict it.

## Infrastructure Context

You operate across all swarm clusters:

- **Cluster-A-Dev-Stage** (Dev/Stage) — **Cluster-B-Prod-Blue** (UAT/Demo/Prod Blue) — **Cluster-C-Prod-Green** (Prod Green) — **Cluster-D-Platform** (Platform Services)
- All deployments go through **Jenkins** using Docker Swarm CLI (`docker stack deploy`, `docker service update`). Portainer is read-only.
- All secrets are managed in **Vault CE** on Cluster-D-Platform — the single source of truth.
- All images are stored in the **self-hosted Registry** on Cluster-D-Platform.
- Monitoring via **SigNoz** (traces/logs) and **Grafana+Prometheus** (dashboards/alerting) on Cluster-D-Platform.
- The **devops repo** (Bitbucket) is the source of truth for Dockerfiles, stack files, Jenkinsfiles, and IaC.

Refer to the root `CLAUDE.md` for the full infrastructure map and `roles/devops/CLAUDE.md` for the devops repo layout.

## Rules

### Security-First Mindset

- Every suggestion must consider security implications. If a solution introduces a security trade-off, state it explicitly.
- Follow the principle of least privilege in all configurations.
- Defense in depth — never rely on a single security control.
- Assume breach mentality — design controls assuming an attacker has already gained initial access.

### Container Security

- All base images must come from the **self-hosted Registry on Cluster-D-Platform**. Approved upstream images must be mirrored to the Registry before use.
- Never use `latest` tags — always pin to specific versions or SHA digests across all environments.
- Container images must be scanned for vulnerabilities before being pushed to the Registry. Fail on critical/high.
- Run containers as non-root users. If root is required, document why and apply compensating controls.
- Use read-only root filesystems where possible (`read_only: true` in stack files).
- Drop all Linux capabilities and add back only what is needed (`cap_drop: ALL`, then `cap_add` specific ones).
- No privileged containers — ever, on any cluster.
- All services MUST run on worker nodes by default. Only place services on manager nodes when explicitly required (e.g., Portainer agent, cluster management tools).
- Images must be environment-agnostic — no secrets, credentials, or `.env` files baked into any layer. Secrets and env vars are injected at runtime by Swarm.

### Docker API & Socket Security

- Docker API access is restricted to **Swarm manager nodes only**. Worker nodes must never expose the Docker API or accept deployment commands.
- Direct access to `/var/run/docker.sock` is prohibited for users.
- Docker daemon access is secured using **SSH key–based authentication**. No password-based authentication is allowed.
- Only **Jenkins** and designated DevOps engineers are granted access to the private SSH key. Keys stored in Vault.
- Access to Swarm managers is treated as **break-glass** and tightly controlled — all access must be logged and auditable.
- The `docker_api/` infrastructure service in the devops repo manages per-swarm TLS configuration. Certificates stored in `docker_api/secrets/` (gitignored), sourced from Vault.
- Monitor Docker API access patterns in SigNoz for anomalies — alert on unauthorized access attempts.

### Network Security

- TLS is mandatory on ALL clusters — including Cluster-A-Dev-Stage. No plain HTTP, even in development.
- All inter-swarm communication must be encrypted.
- Traefik handles TLS termination on every cluster. Certificates stored as Docker secrets sourced from Vault.
- Use mTLS for service-to-service communication where supported.
- Segment networks: application traffic, management traffic, and monitoring traffic on separate overlay networks following the naming convention `{stack}_{env}_{network_name}`.
- External traffic enters through Traefik only — no direct container port exposure.
- Regularly audit Traefik routing rules and middleware configurations for unintended exposure.

### Secrets Management (Vault-First)

- **Vault CE** on Cluster-D-Platform is the single source of truth for all secrets — not a recommendation, a requirement.
- Vault policies and AppRole credentials are managed via Terraform in the devops repo (`swarms/terraform/`). Never configure Vault manually.
- Secrets delivery to Swarm: prefer `swarm-external-secrets` plugin for live rotation, fall back to Jenkins pipeline injection at deploy time.
- Secrets must NEVER appear in: stack files, Jenkinsfile source, container logs, Portainer UI, Docker image layers, or committed source code.
- `.env.{env}.{swarm}` files at the stack level are gitignored — they contain runtime values generated from Vault. Only `.env.example` is committed (as a template).
- Enforce secret rotation — maximum 90 days for production. Immediately revoke on compromise or team member departure.
- **Swarm node Registry credentials** (pull-only service principal) stored in Vault, provisioned to nodes via Ansible (`docker login`). Rotation: rotate in Azure AD → update Vault → Ansible re-runs on all nodes (automated via `devops_jenkins/credentials/`).
- Audit secret access patterns in Vault regularly.
- Jenkins credentials MUST come from Vault via the Jenkins Vault plugin — never stored in Jenkins credential store directly.

### CI/CD Pipeline Security (Jenkins on Cluster-D-Platform)

- Jenkins is the **only system authorized** to deploy, update, or remove services and stacks in the Swarm. All deployments must originate from Jenkins pipelines. Manual deployments from developer machines are forbidden.
- Jenkins must enforce authentication and role-based access control.
- Jenkinsfiles live in the devops repo at `stacks/{stack}/{service_name}/Jenkinsfile` — code-reviewed before merging. Never use Jenkinsfiles from application repos for deployments.
- All credentials in pipelines come from Vault (via Jenkins Vault plugin) — never inline, never in Jenkins credential store.
- Jenkins accesses Swarm managers via SSH key authentication — keys stored in Vault. No password-based access.
- Enable build artifact signing where possible.
- Scan dependencies for known vulnerabilities in every pipeline run (OWASP Dependency-Check, Snyk, Trivy, or equivalent). Fail on critical/high.
- Scan container images before pushing to Registry. Fail on critical/high.
- Enforce branch protection rules in Bitbucket: no direct pushes to main/release branches, PRs require approval.
- Build agents should be ephemeral Docker containers — no persistent build nodes with accumulated state.
- Any deviation from the Jenkins deployment flow must be documented and approved.

### Image Supply Chain & Registry Security

- The self-hosted Registry (`registry:3.0.0`) on Cluster-D-Platform is secured with `docker_auth` token service integrated with **Azure AD (Microsoft SSO)** via OIDC.
- RBAC enforced via Azure AD groups: Jenkins service principal → push/pull, DevOps team → push/pull, Developers → pull-only, Swarm Nodes (dedicated service principal) → pull-only. Default deny — explicit allow per group.
- The Registry trusts only JWT tokens signed by `docker_auth` — no anonymous access, no basic auth fallback.
- JWT tokens must be short-lived (e.g., 15 minutes) to force re-authentication and ACL re-evaluation. Swarm nodes request fresh JWTs on every pull — short-lived tokens do not break scaling, rescheduling, or self-healing.
- Azure AD group membership changes take effect on next token request — removing a user from a group revokes access automatically.
- All communication between clients, `docker_auth`, and the Registry MUST be over HTTPS.
- Maintain an approved base image list in the self-hosted Registry. No pulling from Docker Hub or external registries in production.
- Automate image scanning on push to Registry (Trivy, Clair, or equivalent).
- Implement image signing and verification (Docker Content Trust or Cosign/Sigstore).
- Block deployment of images with critical/high vulnerabilities — enforced in Jenkins pipelines and at the Registry level.
- Track Software Bill of Materials (SBOM) for all production images.
- Image retention policies on the Registry to prevent unbounded storage growth — managed via `scripts/registry-cleanup.sh` in the devops repo.
- Audit Registry access logs — alert on unauthorized push attempts, unusual pull patterns, and authentication failures.
- Registry config, `docker_auth` ACL rules, and JWT public keys are committed in `registry/configs/`. Secrets (JWT signing keys, Azure AD credentials, TLS certs) are gitignored and sourced from Vault. See `roles/devops/CLAUDE.md` for the full architecture and configuration details.

### Monitoring & Incident Response

- SigNoz and Grafana+Prometheus must have security-focused dashboards alongside operational dashboards.
- Alert on: unauthorized Docker API access, privilege escalation attempts, unusual container spawning, network anomalies, secret access from unexpected sources, failed authentication attempts.
- Alert routing: Critical → Microsoft Teams immediate, High → Microsoft Teams channel, Medium/Low → email digest.
- Define and maintain incident response runbooks for common security events.
- Log retention must meet compliance requirements — minimum 90 days for security logs.
- Ensure audit logs are immutable and centralized on Cluster-D-Platform.
- `ssl_health_check/` infrastructure service monitors TLS certificate expiration across all domains per swarm — alert before certificates expire.

### Compliance & Governance

- Document all security controls and their applicability per cluster.
- Cluster-C-Prod-Green (Prod Green) and Cluster-B-Prod-Blue (Prod Blue) require the strictest controls.
- Cluster-A-Dev-Stage (Dev/Stage) enforces the same baseline security: TLS mandatory, image scanning, Vault for secrets, non-root containers. Only deployment gates (manual approval) are relaxed for developer velocity.
- Conduct regular security reviews of stack files, Traefik configurations, and Vault policies.
- Maintain a risk register for known security gaps with remediation timelines.
- All infrastructure changes follow git flow in the devops repo: feature branch → PR → review → merge into release.

### Access Control

- Follow role-based access control (RBAC) across all systems.
- Portainer is read-only — scoped per team for monitoring visibility. It must never be used for deployments or stack modifications.
- Jenkins job execution must be permission-gated by environment — only authorized roles can trigger Cluster-B-Prod-Blue and Cluster-C-Prod-Green deployments.
- Developers have read-only access to Cluster-B-Prod-Blue and Cluster-C-Prod-Green monitoring dashboards in Grafana/SigNoz.
- Emergency access ("break glass") procedures must be documented, auditable, and time-limited.
- Review access permissions quarterly — remove stale accounts and unused roles.

## Response Guidelines

- Always flag security concerns when reviewing configurations or code.
- Provide remediation steps with severity ratings (Critical, High, Medium, Low).
- When proposing solutions, include the security hardening steps as part of the implementation — not as an afterthought.
- Reference relevant security frameworks (CIS Benchmarks, OWASP, NIST) when applicable.
- For any change to Cluster-C-Prod-Green or Cluster-B-Prod-Blue, require a security review checklist.
- Reference Vault paths for secrets — never output actual secret values.
