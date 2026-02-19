# Plan.com Organization - Claude Rules

## Organization Overview

This repository contains role-based CLAUDE.md rules for the **Plan.com** organization. Each role has its own set of guidelines that Claude must follow when assisting team members in that capacity.

## Infrastructure Map

### Docker Swarm Clusters

| Cluster Name       | Environment            | Purpose                            |
|-------------------|------------------------|------------------------------------|
| **Constellations** | Dev / Stage            | Development and staging workloads  |
| **Galaxies**       | Beta / Prod (Blue)     | Blue production + beta testing     |
| **StarWars**       | Prod (Green)           | Green production (live traffic)    |
| **MythicalCreatures** | Platform Services   | Registry, CI/CD, Monitoring        |

### Shared Services per Swarm (Constellations, Galaxies, StarWars)

- **Portainer** — Container management UI per swarm
- **Traefik** — Reverse proxy / load balancer / SSL termination per swarm
- **Docker API Service** — Docker Engine API exposed on port `2375` per swarm

### Platform Services (MythicalCreatures Swarm)

- **Self-Hosted Docker Registry** — Private container image storage
- **Jenkins** — CI/CD pipeline builder
- **SigNoz** — Observability and monitoring (traces, metrics, logs)
- **Grafana + Prometheus** — Metrics dashboards and alerting (integrated with SigNoz)

## Deployment Flow

```
Code Push → Jenkins (MythicalCreatures)
  → Build & Push Image → Registry (MythicalCreatures)
    → Deploy to Constellations (Dev/Stage)
      → Promote to Galaxies (Beta/Prod Blue)
        → Promote to StarWars (Prod Green)
```

## Role-Based Rules

Each subdirectory under `roles/` contains a `CLAUDE.md` tailored for a specific role:

| Role | Path | Description |
|------|------|-------------|
| DevOps | `roles/devops/CLAUDE.md` | Infrastructure, swarm management, CI/CD pipelines |
| DevSecOps | `roles/devsecops/CLAUDE.md` | Security scanning, compliance, secrets management |
| Backend (General) | `roles/backend/CLAUDE.md` | Shared backend conventions |
| Backend (PHP) | `roles/backend/php/CLAUDE.md` | PHP stack rules |
| Backend (Node.js+TS) | `roles/backend/nodejs-typescript/CLAUDE.md` | Node.js + TypeScript stack rules |
| Backend (Python) | `roles/backend/python/CLAUDE.md` | Python stack rules |
| Backend (Golang) | `roles/backend/golang/CLAUDE.md` | Golang stack rules |
| Frontend | `roles/frontend/CLAUDE.md` | Frontend development rules |
| Product Owner | `roles/product-owner/CLAUDE.md` | Product management and story writing |
| Design UX | `roles/design-ux/CLAUDE.md` | UX/UI design guidelines |
| QA Tester | `roles/qa-tester/CLAUDE.md` | Testing strategy and quality assurance |

## How to Use

1. **Copy the relevant `CLAUDE.md`** into your project repository root, or
2. **Reference it** in your project's own `CLAUDE.md` using an import/include pattern, or
3. **Compose multiple roles** by combining relevant sections for cross-functional work

## Naming Conventions

- Swarm clusters are always referred to by their codenames: **Constellations**, **Galaxies**, **StarWars**, **MythicalCreatures**
- Never use generic names like "dev cluster" or "prod cluster" — always use the codenames
- Environment labels: `dev`, `stage`, `beta`, `prod-blue`, `prod-green`
