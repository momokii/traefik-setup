# Security Standards

## Current Security Posture: GREEN

All previous audit findings (SEC-001 through SEC-005) resolved in the production revamp.

## Resolved Issues

| Issue | Resolution |
|-------|------------|
| SEC-001: Missing `.gitignore` | Created — excludes `.env`, `acme.json`, `letsencrypt_data/`, `*.log` |
| SEC-002: Insecure dashboard | Disabled — dashboard behind basic auth on `traefik.${DOMAIN}` |
| SEC-003: Docker socket read-write | Fixed — mounted `:ro` (read-only) |
| SEC-004: Unclosed backtick | Fixed — old `dynamic-config.yaml` replaced with clean `dynamic.yaml` |
| SEC-005: No container restrictions | Fixed — `security_opt: no-new-privileges:true` added |

## Current Security Measures

### TLS/HTTPS
- TLS 1.2 minimum enforced (`tls.options.default.minVersion`)
- Modern cipher suites only (no weak ciphers)
- SNI strict mode enabled
- HTTP to HTTPS redirect on port 80
- Let's Encrypt auto-provisioning and renewal

### Dashboard
- `api.insecure: false` — no public port
- Basic auth middleware on `traefik.${DOMAIN}`
- Credentials generated via `htpasswd`, stored in `dynamic.yaml`

### Headers
- HSTS enabled (63072000 seconds, includeSubDomains, preload)
- Content-Type nosniff
- Browser XSS filter
- Server header removed
- Referrer policy: same-origin

### Container Security
- Docker socket mounted read-only
- `no-new-privileges:true` on Traefik container
- `exposedByDefault: false` — explicit opt-in required

### Secrets Management
- All secrets in `.env` (gitignored)
- `.env.example` contains only placeholder values
- No real domains, emails, or credentials in committed files

### Logging
- Access logging enabled (`/var/log/traefik/access.log`)
- Traefik logs at INFO level (`/var/log/traefik/traefik.log`)
- Log buffering for performance
- View via: `docker exec traefik tail -f /var/log/traefik/traefik.log`

## Standing Security Requirements

- Never commit `.env` or `acme.json`
- Never set `api.insecure: true`
- Never mount Docker socket without `:ro`
- Always validate configs before deploying (`docker compose config`)
- Pin all Docker image versions — no `latest` in production
