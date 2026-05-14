# Traefik Setup — Agent Orientation

## What This Project Is

A production-ready Traefik reverse proxy setup for Docker. Features automatic SSL via Let's Encrypt, secure dashboard with basic auth, and a drop-in app template for easy deployment.

## Tech Stack

- **Infrastructure**: Docker Compose, Traefik v3.1.5
- **Configuration**: YAML only — no application code, no build step
- **Network**: External Docker network `traefik-network`
- **SSL**: Let's Encrypt with HTTP-01 challenge

## Repository Structure

```
traefik-setup/
├── compose.yaml              # Traefik container (hardened)
├── .env.example              # Config template
├── .gitignore                # Protects .env, acme.json, logs
├── traefik/
│   ├── traefik.yaml          # Static config (entrypoints, providers, cert resolver)
│   └── dynamic.yaml          # Dashboard auth, TLS options, security headers
├── templates/
│   └── app-compose.yaml      # Copy-paste template for new apps
├── examples/                 # Reference configs (not part of core setup)
│   ├── canary-deployment/
│   ├── ssl-setup/
│   └── sablier-zero-scale/
└── .claude/                  # Agent infrastructure (this directory)
```

## Orientation Sequence

Read these files in order before starting any work:

1. **This file** (`.claude/README.md`) — project overview and structure
2. `.claude/AGENT_RULES.md` — behavioral rules for this session
3. `.claude/CODING_STANDARDS.md` — YAML and Docker Compose conventions used here
4. `.claude/SECURITY_STANDARDS.md` — security posture and requirements
5. `.claude/ENVIRONMENT_GUIDE.md` — how to start/stop environments
6. `.claude/state/CURRENT_STATUS.md` — what is done, in progress, and blocked
7. `.claude/state/TASK_QUEUE.md` — next tasks to work on

## Environment Bootstrap

```bash
# 1. Create the external network (once)
docker network create traefik-network

# 2. Configure environment
cp .env.example .env
# Edit .env with your email

# 3. Start Traefik
docker compose up -d

# 4. Verify
docker logs traefik
```

## Key Docs Index

- `README.md` — full project documentation with setup guides and troubleshooting
- `traefik/traefik.yaml` — static Traefik configuration
- `traefik/dynamic.yaml` — dashboard auth, TLS options, security headers
- `templates/app-compose.yaml` — template for adding new apps
