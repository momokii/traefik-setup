# Traefik Reverse Proxy — Production Revamp Design

**Date:** 2026-05-14
**Status:** Approved

## Goal

Transform this repo from a "reference examples" project into a proper, production-ready Traefik reverse proxy setup for a personal Ubuntu/Debian VM. The setup must be easy to use — adding a new app should be copy-paste-edit-deploy.

## User Context

- Single VM running Ubuntu/Debian
- One main domain with subdomains (apps on `app.mydomain.com`)
- Cloudflare DNS with proxy OFF (grey cloud / DNS-only)
- Let's Encrypt for SSL (HTTP-01 challenge)
- Traefik dashboard with basic auth on a subdomain
- This repo = Traefik only. Apps live in separate repos/folders on the VM.

## Architecture

```
Internet → Cloudflare DNS (proxy off) → VM
  → Traefik (80/443)
    → routes to app containers via labels
    → auto-provisions Let's Encrypt SSL per domain
    → dashboard on traefik.mydomain.com (basic auth)
```

All containers share `traefik-network` (external Docker network). Traefik auto-discovers containers via Docker provider with `exposedByDefault: false`.

## Repo Structure

```
traefik-setup/
├── .env.example              # Config: domain, email, dashboard creds
├── .gitignore                # Protect .env, acme.json, letsencrypt_data/
├── compose.yaml              # Traefik container definition
├── traefik/
│   ├── traefik.yaml          # Static config
│   └── dynamic.yaml          # Shared middlewares (dashboard auth, security headers, TLS options)
├── templates/
│   └── app-compose.yaml      # Copy-paste template for new apps
├── examples/
│   ├── canary-deployment/    # Weighted round-robin (90/10)
│   ├── ssl-setup/            # Rate limiting + IP allowlist
│   └── sablier-zero-scale/   # Auto-hibernate idle containers
└── README.md                 # Setup guide + how to add apps
```

## Static Config (`traefik/traefik.yaml`)

```yaml
global:
  checkNewVersion: false
  sendAnonymousUsage: false

log:
  level: INFO
  filePath: "/var/log/traefik/traefik.log"

accessLog:
  filePath: "/var/log/traefik/access.log"
  bufferingSize: 100

api:
  dashboard: true
  insecure: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
    http:
      tls:
        options: default
    http3: {}

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: "/etc/traefik/dynamic.yaml"
    watch: true

certificatesResolvers:
  letsencrypt:
    acme:
      email: "${ACME_EMAIL}"
      storage: "/letsencrypt/acme.json"
      httpChallenge:
        entryPoint: web
```

## Compose (`compose.yaml`)

```yaml
services:
  traefik:
    image: traefik:v3.7.1
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/traefik.yaml:/etc/traefik/traefik.yaml:ro
      - ./traefik/dynamic.yaml:/etc/traefik/dynamic.yaml:ro
      - letsencrypt_data:/letsencrypt
      - traefik_logs:/var/log/traefik
    env_file:
      - .env
    networks:
      - traefik-network

volumes:
  letsencrypt_data:
  traefik_logs:

networks:
  traefik-network:
    external: true
```

Key changes from current: no port 8080, docker socket read-only, `security_opt`, HTTP/3 UDP port, named volumes, env file for secrets.

## Dynamic Config (`traefik/dynamic.yaml`)

Contains only shared infrastructure:

1. **Dashboard router** — routes `traefik.${DOMAIN}` to `api@internal` with basic auth middleware
2. **TLS options** — minimum TLS 1.2, modern cipher suites, SNI strict
3. **Security headers middleware** — HSTS, content-type nosniff, server header removed

The dashboard basic auth credentials are hashed using `htpasswd` and stored directly in `dynamic.yaml`. The README will include the exact command to generate the hash:
```bash
htpasswd -nb admin changeme
```
The user runs this once with their chosen username/password and pastes the output into `dynamic.yaml`.

## `.env.example`

```
DOMAIN=mydomain.com
ACME_EMAIL=email@example.com
DASHBOARD_USERNAME=admin
DASHBOARD_PASSWORD=changeme
```

User copies to `.env` and fills in real values. `.gitignore` blocks `.env` from commits.

## App Template (`templates/app-compose.yaml`)

Ready-to-copy file with placeholder comments:

```yaml
services:
  APP_NAME:
    image: APP_IMAGE
    container_name: APP_NAME
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.APP_NAME.rule=Host(`APP_NAME.${DOMAIN}`)"
      - "traefik.http.routers.APP_NAME.entrypoints=websecure"
      - "traefik.http.routers.APP_NAME.tls.certresolver=letsencrypt"
      - "traefik.http.services.APP_NAME.loadbalancer.server.port=PORT"
    networks:
      - traefik-network

networks:
  traefik-network:
    external: true
```

Workflow: copy → replace `APP_NAME`, `APP_IMAGE`, `PORT` → `docker-compose up -d`.

## README Structure

1. **Quick Start** — prerequisites, one-time setup, verify it works
2. **Adding an App** — step-by-step with Cloudflare DNS instructions
3. **Dashboard** — how to access, how to change credentials
4. **Configuration** — what each file does, how to customize
5. **Troubleshooting** — common issues and fixes
6. **Advanced** — DNS-01 challenge for wildcard certs, IP allowlist, per-app middleware

## Security Posture

| Measure | Status |
|---------|--------|
| Dashboard not on public port | `api.insecure: false`, no 8080 mapping |
| Dashboard behind basic auth | Yes, via middleware |
| Docker socket read-only | Yes (`:ro`) |
| Container no-new-privileges | Yes (`security_opt`) |
| TLS 1.2 minimum | Yes (`tls.options.default`) |
| HSTS + security headers | Yes (middleware) |
| HTTP→HTTPS redirect | Yes (entrypoint) |
| No telemetry | Yes (`sendAnonymousUsage: false`) |
| Secrets not committed | Yes (`.env` + `.gitignore`) |
| Access logging | Yes (`accessLog`) |

## Examples Migration

Existing examples moved into `examples/` folder with updated paths and references. They remain as reference material, not part of the core setup.

## Not In Scope

- Docker socket proxy (adds complexity, overkill for personal VM)
- DNS-01 challenge (documented as advanced option, not default)
- Per-app network isolation (single shared network is fine for personal use)
- Automated setup scripts (compose + env file is simple enough)
