# Traefik Reverse Proxy

Production-ready Traefik reverse proxy setup for Docker. Automatic SSL via Let's Encrypt, secure dashboard, and a dead-simple workflow for adding new apps.

## How It Works

```
Internet → Cloudflare DNS (proxy off)
  → Your VM
    → Traefik (ports 80/443)
      → auto-routes to your app containers
      → auto-provisions SSL certificates
```

Each app is a separate folder with its own `compose.yaml`. Apps self-register with Traefik via Docker labels. No shared config files to edit. Adding an app = copy template, change domain, deploy.

## Prerequisites

- Ubuntu/Debian VM with Docker + Docker Compose installed
- A domain name using Cloudflare DNS
- Ports 80 and 443 open on your VM firewall

## One-Time Setup

### 1. Create the Docker network

```bash
docker network create traefik-network
```

This only needs to be done once on your VM. All apps + Traefik share this network.

### 2. Configure your environment

```bash
cp .env.example .env
```

Edit `.env` with your real values:

```
DOMAIN=yourdomain.com
ACME_EMAIL=your@email.com
```

### 3. Set up dashboard authentication

Generate a password hash for the Traefik dashboard:

```bash
htpasswd -nb admin "your-password" | sed -e 's/\$/\$\$/g'
```

Copy the output (looks like `admin:$$apr1$$...$$...`) and paste it into `traefik/dynamic.yaml`, replacing the placeholder line under `dashboard-auth`:

```yaml
users:
  - "admin:$$apr1$$...$$..."   # your htpasswd output here
```

Also update the dashboard domain in `dynamic.yaml`:

```yaml
rule: "Host(`traefik.yourdomain.com`)"    # change YOUR-DOMAIN
```

### 4. Add DNS records in Cloudflare

Add two A records (proxy status = DNS only / grey cloud):

| Type  | Name    | Content      | Proxy     |
|-------|---------|--------------|-----------|
| A     | traefik | your VM IP   | DNS only  |

You'll add more records as you deploy apps (one per app).

### 5. Start Traefik

```bash
docker-compose up -d
```

Verify it's running:

```bash
docker logs traefik
```

Access the dashboard at `https://traefik.yourdomain.com` with the credentials you set up.

## Adding a New App

### 1. Add DNS record

In Cloudflare, add an A record:

| Type  | Name      | Content    | Proxy     |
|-------|-----------|------------|-----------|
| A     | myapp     | your VM IP | DNS only  |

This creates `myapp.yourdomain.com` pointing to your VM.

### 2. Create the app

On your VM, create a folder for your app and use the template:

```bash
mkdir -p /path/to/myapp && cd /path/to/myapp
cp /path/to/traefik-setup/templates/app-compose.yaml compose.yaml
```

Edit `compose.yaml` — change the 4 values marked `CHANGE ME`:

```yaml
services:
  myapp:
    image: nginx:alpine                              # your app's image
    container_name: myapp
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.yourdomain.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
      - "traefik.http.services.myapp.loadbalancer.server.port=80"
    networks:
      - traefik-network

networks:
  traefik-network:
    external: true
```

### 3. Deploy

```bash
docker-compose up -d
```

That's it. SSL is automatic. Your app is live at `https://myapp.yourdomain.com`.

### Removing an app

```bash
cd /path/to/myapp
docker-compose down
```

Then remove the DNS record in Cloudflare.

## Using Multiple Domains

This setup supports any number of domains. Just point the domain's DNS to your VM IP and use the full domain in the `Host()` rule:

```yaml
- "traefik.http.routers.myapp.rule=Host(`myapp.otherdomain.com`)"
```

No Traefik config changes needed. It just works.

## Configuration Files

| File | What | Edit when |
|------|------|-----------|
| `.env` | Domain, email, credentials | Initial setup only |
| `compose.yaml` | Traefik container definition | Rarely (version upgrades) |
| `traefik/traefik.yaml` | Static config (entrypoints, providers) | Rarely (needs restart) |
| `traefik/dynamic.yaml` | Dashboard auth, TLS options, security headers | Initial setup only |
| `templates/app-compose.yaml` | Template for new apps | Never (copy it) |

**Important:** `traefik.yaml` changes require a container restart (`docker-compose restart`). `dynamic.yaml` auto-reloads without restart.

## Security

This setup includes:

- **Dashboard** behind basic auth, no insecure port exposed
- **Docker socket** mounted read-only
- **Container** runs with `no-new-privileges`
- **TLS 1.2 minimum** with modern cipher suites
- **HSTS** enabled with preload
- **Security headers** (content-type nosniff, server header hidden)
- **HTTP to HTTPS redirect** on port 80
- **No telemetry** (`sendAnonymousUsage: false`)
- **Secrets** in `.env` (excluded from git)

## Adding Middleware to an App

You can add rate limiting, IP allowlist, or other middleware per-app via labels:

```yaml
labels:
  # ... existing labels ...
  # Rate limiting: 10 requests per second average
  - "traefik.http.middlewares.myapp-ratelimit.ratelimit.average=10"
  - "traefik.http.routers.myapp.middlewares=myapp-ratelimit"
```

To use the global security headers middleware on your app:

```yaml
  - "traefik.http.routers.myapp.middlewares=security-headers@file"
```

## Troubleshooting

**App not reachable:**
```bash
# Is the container running?
docker ps | grep myapp

# Is it on the right network?
docker network inspect traefik-network

# Check Traefik logs
docker logs traefik --tail 50
```

**SSL certificate not issued:**
```bash
# Check Traefik logs for ACME errors
docker logs traefik 2>&1 | grep acme

# Verify DNS resolves to your VM
dig myapp.yourdomain.com

# Make sure port 80 is open (needed for HTTP-01 challenge)
curl -I http://myapp.yourdomain.com
```

**Dashboard not loading:**
```bash
# Check if the router is registered
docker logs traefik 2>&1 | grep dashboard

# Verify DNS for traefik subdomain
dig traefik.yourdomain.com
```

## Examples

The `examples/` folder contains reference configurations:

- **canary-deployment** — weighted round-robin (90/10 traffic split)
- **ssl-setup** — rate limiting + IP allowlist
- **sablier-zero-scale** — auto-hibernate idle containers

These are for learning. Your production setup is the root config + the template.

## Cloudflare Settings

For this setup, use these Cloudflare settings:

- **SSL/TLS mode:** Full (Strict) — Traefik has valid Let's Encrypt certs
- **Always Use HTTPS:** Can be ON or OFF (Traefik already redirects)
- **Proxy status:** DNS only (grey cloud) for all records — Let's Encrypt handles SSL directly

If you want to use Cloudflare's orange cloud (proxied) later, switch to DNS-01 challenge with a Cloudflare API token. See Traefik's [DNS challenge documentation](https://doc.traefik.io/traefik/v3.0/user-guides/docker-compose/acme-dns/).
