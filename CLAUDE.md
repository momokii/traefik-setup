# Traefik Reverse Proxy Setup

Production-ready Traefik reverse proxy for Docker. Drop-in app deployment with automatic SSL.

## Commands

```bash
docker network create traefik-network           # First time — required
docker-compose up -d                            # Start Traefik
docker logs traefik                             # Check status
docker-compose config                           # Validate compose syntax
docker-compose restart                          # Apply static config changes
```

## Structure

```
compose.yaml              → Traefik container (ports 80/443, HTTP/3)
.env                      → Secrets: domain, email, dashboard creds (gitignored)
.env.example              → Template for .env
traefik/
  traefik.yaml            → Static config (entrypoints, providers, cert resolver)
  dynamic.yaml            → Dashboard auth, TLS options, security headers
templates/
  app-compose.yaml        → Copy-paste template for adding new apps
examples/                 → Reference configs (canary, ssl, sablier)
```

## Adding a New App

1. Add DNS A record in Cloudflare (subdomain → VM IP, proxy OFF)
2. Copy `templates/app-compose.yaml` to app's folder
3. Replace the 4 `CHANGE ME` values (name, image, domain, port)
4. `docker-compose up -d`
5. SSL is automatic

## Conventions

- All services use external network `traefik-network`
- Sensitive values in `.env` (never committed)
- Placeholder in `.env.example`: `yourdomain.com`, `email@example.com`
- Docker socket always mounted `:ro`
- Traefik routing via Docker labels on each app's compose
- `dynamic.yaml` auto-reloads — no restart needed
- `traefik.yaml` changes require `docker-compose restart`

## Security

- Dashboard behind basic auth on `traefik.${DOMAIN}` (no insecure port)
- Docker socket read-only + `no-new-privileges`
- TLS 1.2 minimum, modern ciphers, HSTS
- Security headers middleware (nosniff, server hidden)
- Access logging enabled

## Gotchas

- `traefik-network` must exist before starting anything
- Traefik must start before any apps
- Port 80 must be reachable for Let's Encrypt HTTP-01 challenge
- Dynamic config errors apply immediately on save
- ACME email passed via env var, not in YAML (Traefik doesn't support `${VAR}` in static config)

## Agent Infrastructure (`.claude/`)

Read `.claude/README.md` first, then use these references as needed:

| File | When to Read |
|---|---|
| `.claude/AGENT_RULES.md` | Before any work — behavioral rules for this project |
| `.claude/CODING_STANDARDS.md` | Before writing YAML — naming, patterns, file placement conventions |
| `.claude/SECURITY_STANDARDS.md` | Before TLS/auth/network changes — current security measures |
| `.claude/ENVIRONMENT_GUIDE.md` | Before running Docker commands — start/stop/validate procedures |
| `.claude/HOW_TO_RESUME.md` | At session start — step-by-step resume protocol |
| `.claude/state/CURRENT_STATUS.md` | At session start — what's done, in progress, blocked |
| `.claude/state/TASK_QUEUE.md` | At session start — task list (all previous tasks completed) |
| `.claude/state/DECISIONS_LOG.md` | Before architectural changes — documented decisions |
