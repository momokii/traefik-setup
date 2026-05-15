# Traefik Production Revamp Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure the Traefik setup repo into a production-ready reverse proxy with proper security, dashboard auth, and an easy app template.

**Architecture:** Single Traefik container with static + dynamic config. Apps connect via external Docker network and self-register via labels. Dashboard secured with basic auth behind Traefik itself. All secrets in `.env`.

**Tech Stack:** Traefik v3.1.5, Docker Compose v3.9, Let's Encrypt HTTP-01

---

### Task 1: Create `.gitignore`

**Files:**
- Create: `.gitignore`

- [ ] **Step 1: Write `.gitignore`**

```
.env
acme.json
letsencrypt_data/
*.log
```

- [ ] **Step 2: Validate**

Run: `git status` — should not track `.env`, `acme.json`, or `letsencrypt_data/`

---

### Task 2: Create `.env.example`**

**Files:**
- Create: `.env.example`

- [ ] **Step 1: Write `.env.example`**

```
DOMAIN=mydomain.com
ACME_EMAIL=email@example.com
DASHBOARD_USERNAME=admin
DASHBOARD_PASSWORD=changeme
```

---

### Task 3: Create new `compose.yaml` at root

**Files:**
- Create: `compose.yaml` (root level, replaces `traefik/compose.yaml`)

- [ ] **Step 1: Write `compose.yaml`**

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
      - ./traefik/dynamic.yaml:/etc/traefik/dynamic.yaml
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

Note: `dynamic.yaml` is NOT mounted `:ro` so Traefik can auto-reload it when edited. `traefik.yaml` IS `:ro` since it requires restart anyway.

- [ ] **Step 2: Validate syntax**

Run: `docker-compose -f compose.yaml config`
Expected: Valid config output with no errors

---

### Task 4: Create new static config `traefik/traefik.yaml`

**Files:**
- Create: `traefik/traefik.yaml` (replaces `traefik/traefik-config.yaml`)

- [ ] **Step 1: Write `traefik/traefik.yaml`**

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

---

### Task 5: Create new dynamic config `traefik/dynamic.yaml`

**Files:**
- Create: `traefik/dynamic.yaml` (replaces `traefik/dynamic-config.yaml`)

- [ ] **Step 1: Write `traefik/dynamic.yaml`**

Contains: dashboard router, basic auth middleware, TLS options, security headers middleware.

```yaml
http:
  routers:
    dashboard:
      rule: "Host(`traefik.${DOMAIN}`)"
      entryPoints:
        - websecure
      service: api@internal
      middlewares:
        - dashboard-auth
      tls:
        certResolver: letsencrypt

  middlewares:
    dashboard-auth:
      basicAuth:
        users:
          - "admin:$apr1$placeholder$replace_with_htpasswd_hash"

    security-headers:
      headers:
        stsSeconds: 63072000
        stsIncludeSubdomains: true
        stsPreload: true
        forceSTSHeader: true
        contentTypeNosniff: true
        browserXssFilter: true
        referrerPolicy: "same-origin"
        customResponseHeaders:
          server: ""

tls:
  options:
    default:
      minVersion: VersionTLS12
      sniStrict: true
      cipherSuites:
        - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
        - TLS_AES_128_GCM_SHA256
        - TLS_AES_256_GCM_SHA384
        - TLS_CHACHA20_POLY1305_SHA256
      curvePreferences:
        - CurveP521
        - CurveP384
```

---

### Task 6: Create app template `templates/app-compose.yaml`

**Files:**
- Create: `templates/app-compose.yaml`

- [ ] **Step 1: Write template**

```yaml
# Traefik Reverse Proxy — App Template
#
# HOW TO USE:
# 1. Copy this file to your app's directory
# 2. Replace all PLACEHOLDER values below
# 3. Run: docker-compose up -d
# 4. Add DNS A record in Cloudflare: PLACEHOLDER_SUBDOMAIN.yourdomain.com → your VM IP (proxy off)
#
# PLACEHOLDERS TO REPLACE:
#   APP_NAME        → your app's container name (e.g. "myapp")
#   APP_IMAGE       → docker image (e.g. "nginx:alpine")
#   APP_PORT        → port your app listens on (e.g. 80, 3000, 8080)
#   APP_SUBDOMAIN   → subdomain for this app (e.g. "myapp" → myapp.yourdomain.com)

services:
  APP_NAME:
    image: APP_IMAGE
    container_name: APP_NAME
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.APP_NAME.rule=Host(`APP_SUBDOMAIN.yourdomain.com`)"
      - "traefik.http.routers.APP_NAME.entrypoints=websecure"
      - "traefik.http.routers.APP_NAME.tls.certresolver=letsencrypt"
      - "traefik.http.services.APP_NAME.loadbalancer.server.port=APP_PORT"
    networks:
      - traefik-network

networks:
  traefik-network:
    external: true
```

---

### Task 7: Move examples into `examples/` folder

**Files:**
- Move: `ssl-setup-test/` → `examples/ssl-setup/`
- Move: `canary-deployment-test/` → `examples/canary-deployment/`
- Move: `sablier-test-zero-scale/` → `examples/sablier-zero-scale/`

- [ ] **Step 1: Create examples directory and move folders**

```bash
mkdir -p examples
git mv ssl-setup-test examples/ssl-setup
git mv canary-deployment-test examples/canary-deployment
git mv sablier-test-zero-scale examples/sablier-zero-scale
```

- [ ] **Step 2: Fix the unclosed backtick bug in canary deployment**

In `examples/canary-deployment/html1/index.html` and `html2/index.html` — no changes needed (HTML files).

The unclosed backtick was in `traefik/dynamic-config.yaml` which is being replaced entirely. No fix needed in examples since they use Docker labels, not dynamic config, for their rules.

---

### Task 8: Remove old files

**Files:**
- Delete: `traefik/compose.yaml`
- Delete: `traefik/traefik-config.yaml`
- Delete: `traefik/dynamic-config.yaml`

- [ ] **Step 1: Remove old config files**

```bash
git rm traefik/compose.yaml traefik/traefik-config.yaml traefik/dynamic-config.yaml
```

---

### Task 9: Write README.md

**Files:**
- Modify: `README.md` (complete rewrite)

- [ ] **Step 1: Write new README**

Covering: Quick Start, Adding an App, Dashboard, Configuration, Troubleshooting, Advanced.

Full content provided in implementation (too long to include inline in plan — the implementer will write it based on the spec sections).

---

### Task 10: Update `.claude/` state files

**Files:**
- Modify: `.claude/state/CURRENT_STATUS.md` — update to reflect new structure
- Modify: `.claude/state/TASK_QUEUE.md` — mark TASK-001 and TASK-002 as completed
- Modify: `CLAUDE.md` — update structure, commands, gotchas to match new repo layout
- Modify: `.claude/CODING_STANDARDS.md` — update file naming references
- Modify: `.claude/ENVIRONMENT_GUIDE.md` — update commands for new file paths
- Modify: `.claude/SECURITY_STANDARDS.md` — update to reflect resolved issues

- [ ] **Step 1: Update all state files to reflect new structure**

Update file path references, mark completed tasks, update security posture.

---

### Task 11: Validate and commit

- [ ] **Step 1: Validate compose syntax**

Run: `docker-compose -f compose.yaml config`
Expected: Valid output, no errors

- [ ] **Step 2: Validate Traefik static config**

Run: `docker run --rm -v $(pwd)/traefik/traefik.yaml:/etc/traefik/traefik.yaml traefik:v3.7.1 validate /etc/traefik/traefik.yaml`
Expected: "Configuration loaded successfully" or similar

- [ ] **Step 3: Review all changes**

Run: `git diff --cached` and `git status`
Verify: no `.env` staged, no secrets, all old files removed, all new files present

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: production-ready Traefik reverse proxy setup

- Restructured repo: core setup at root, examples in examples/
- Security hardened: no insecure dashboard, docker socket read-only,
  no-new-privileges, TLS 1.2 minimum, HSTS, security headers
- Dashboard behind basic auth on traefik subdomain
- App template for easy drop-in deployment
- .env for secrets, .gitignore for protection
- Access logging enabled
- HTTP/3 (QUIC) support added"
```
