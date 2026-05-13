# Traefik Setup Examples

Configuration-only repo demonstrating Traefik reverse proxy patterns with Docker Compose (SSL/TLS, canary deployments, middleware, zero-scale via Sablier).

## Commands

```bash
docker network create traefik-networks          # First time — required by all services
cd traefik && docker-compose up -d              # Start Traefik core (always first)
docker logs traefik                              # Check Traefik status
curl http://localhost:8080/ping                  # Health check → "OK"
docker-compose -f <path>/compose.yaml config     # Validate YAML syntax
```

## Structure

```
traefik/          → Core Traefik setup (static + dynamic config)
ssl-setup-test/   → SSL/TLS with Let's Encrypt + rate limiting
canary-deployment-test/ → Weighted round-robin load balancing (90/10 split)
sablier-test-zero-scale/ → Auto-scale to zero with Sablier plugin
```

## Conventions

- All services use external network `traefik-networks`
- Sensitive values are placeholders: `YOUR-DOMAIN`, `YOUR_EMAIL_HERE`, `YOUR_IP_ADDRESS`
- Traefik routing via Docker labels (single services) or `dynamic-config.yaml` (multi-service)
- `dynamic-config.yaml` auto-reloads on change — no restart needed (but errors apply immediately)
- Static config changes (`traefik-config.yaml`) require container restart
- Image versions pinned (avoid `latest` for non-test images)

## Gotchas

- **No .gitignore exists** — do not commit `letsencrypt_data/`, `acme.json`, or `.env` files
- **Line 55 of dynamic-config.yaml** has an unclosed backtick: `Host(\`YOUR-DOMAIN-HERE)` → needs closing backtick
- Traefik must start before any example services
- Port 8080 is the insecure dashboard — disable for production
- All services need `traefik.enable=true` label to be routed

## Agent Infrastructure (`.claude/`)

Read `.claude/README.md` first, then use these references as needed:

| File | When to Read |
|---|---|
| `.claude/AGENT_RULES.md` | Before any work — behavioral rules for this project |
| `.claude/CODING_STANDARDS.md` | Before writing YAML — naming, patterns, file placement conventions |
| `.claude/SECURITY_STANDARDS.md` | Before TLS/auth/network changes — audit findings (SEC-001 to SEC-005) and requirements |
| `.claude/ENVIRONMENT_GUIDE.md` | Before running Docker commands — start/stop/validate procedures per environment |
| `.claude/HOW_TO_RESUME.md` | At session start — step-by-step resume protocol |
| `.claude/state/CURRENT_STATUS.md` | At session start — what's done, in progress, blocked |
| `.claude/state/TASK_QUEUE.md` | At session start — prioritized task list (TASK-001 to TASK-005) |
| `.claude/state/DECISIONS_LOG.md` | Before architectural changes — 6 pre-existing decisions documented |
| `.claude/templates/new_feature.md` | When adding a new example setup |
| `.claude/templates/new_endpoint.md` | When adding a new Traefik route/service |
| `.claude/templates/new_test.md` | When validating configuration changes |
| `.claude/templates/bug_fix.md` | When fixing a configuration bug |
