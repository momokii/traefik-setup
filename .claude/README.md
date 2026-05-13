# Traefik Setup — Agent Orientation

## What This Project Is

A collection of practical Traefik reverse proxy configuration examples for Docker environments. Demonstrates SSL/TLS termination via Let's Encrypt, weighted round-robin load balancing for canary deployments, security middleware (rate limiting, IP allowlisting), and zero-scale resource optimization using the Sablier plugin.

## Tech Stack

- **Infrastructure**: Docker Compose, Traefik v3.1.5
- **Plugins**: Sablier v1.10.1
- **Images**: traefik:v3.1.5, traefik/whoami:latest, nginx:alpine, sablierapp/sablier:1.10.1
- **Configuration**: YAML only — no application code, no build step, no test framework
- **Network**: External Docker network `traefik-networks`

## Repository Structure

```
traefik-setup/
├── README.md                    # Comprehensive project documentation
├── traefik/                     # Core Traefik reverse proxy setup
│   ├── compose.yaml             # Traefik container definition
│   ├── traefik-config.yaml      # Static configuration (entrypoints, providers, certs)
│   └── dynamic-config.yaml      # Dynamic config (routers, services, middlewares)
├── ssl-setup-test/              # Example: SSL/TLS with rate limiting
│   └── compose.yaml
├── canary-deployment-test/      # Example: A/B testing with weighted routing
│   ├── compose.yaml
│   ├── html1/index.html         # v1 test content
│   └── html2/index.html         # v2 test content
├── sablier-test-zero-scale/     # Example: Auto-scaling to zero
│   └── compose.yaml
└── .claude/                     # Agent infrastructure (this directory)
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
docker network create traefik-networks

# 2. Start Traefik core
cd traefik && docker-compose up -d

# 3. Verify Traefik is running
docker logs traefik

# 4. (Optional) Start any example service
cd ../ssl-setup-test && docker-compose up -d
```

**Health check**: `curl http://localhost:8080/ping` — returns `OK` when Traefik is healthy.

## Security Note

See `.claude/SECURITY_STANDARDS.md` for full findings. Key points:
- No `.gitignore` exists — one must be created before committing sensitive files
- All placeholder values (`YOUR-DOMAIN`, `YOUR_EMAIL_HERE`, etc.) must be replaced before production use
- Traefik dashboard runs in insecure mode (`api.insecure: true`) — disable for production
- Docker socket is mounted — standard for Traefik but limit container access

## Current State

- **Status**: `.claude/state/CURRENT_STATUS.md`
- **Task queue**: `.claude/state/TASK_QUEUE.md`
- **Decisions log**: `.claude/state/DECISIONS_LOG.md`

## Key Docs Index

- `README.md` — full project documentation with setup guides and troubleshooting
- `traefik/traefik-config.yaml` — static Traefik configuration (entrypoints, cert resolvers, providers)
- `traefik/dynamic-config.yaml` — dynamic routing, services, and middleware definitions
