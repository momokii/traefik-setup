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

- Ubuntu/Debian VM
- A domain name using Cloudflare DNS
- Ports 80 and 443 open on your VM firewall

### Installing Docker (if not already installed)

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin apache2-utils -y
sudo usermod -aG docker $USER
# Log out and back in for the docker group to take effect
```

> `apache2-utils` provides the `htpasswd` command needed for dashboard authentication.

### Opening firewall ports

If your VM uses `ufw`:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
```

> **Note:** Docker may bypass `ufw` rules by default. If ports appear blocked, check Docker's iptables settings or ensure `DEFAULT_FORWARD_POLICY="ACCEPT"` in `/etc/default/ufw`.

## One-Time Setup

### 1. Create the Docker network

```bash
docker network create traefik-network
```

This only needs to be done once on your VM. All apps + Traefik share this network. The network survives VM reboots.

### 2. Configure your environment

```bash
cp .env.example .env
```

Edit `.env` with your real email:

```
ACME_EMAIL=your@email.com
```

This email is used by Let's Encrypt for certificate expiry notifications.

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
rule: "Host(`traefik.yourdomain.com`)"    # replace YOUR-DOMAIN with your domain
```

### 4. Add DNS record in Cloudflare

Add an A record for the dashboard (proxy status = DNS only / grey cloud):

| Type  | Name    | Content      | Proxy     |
|-------|---------|--------------|-----------|
| A     | traefik | your VM IP   | DNS only  |

You'll add more records as you deploy apps (one per app subdomain).

### 5. Start Traefik

```bash
docker compose up -d
```

Verify it's running:

```bash
docker logs traefik
```

You should see Traefik start up with no errors. Access the dashboard at `https://traefik.yourdomain.com` with the credentials you set up.

> **Note:** The first time you access the dashboard, Let's Encrypt will provision an SSL certificate. This may take a few seconds. If it fails, check that your DNS is pointing to the correct VM IP and that port 80 is open.

## Adding a New App

### 1. Add DNS record

In Cloudflare, add an A record:

| Type  | Name      | Content    | Proxy     |
|-------|-----------|------------|-----------|
| A     | myapp     | your VM IP | DNS only  |

This creates `myapp.yourdomain.com` pointing to your VM. The proxy must be OFF (grey cloud) so Let's Encrypt can reach your VM directly for certificate provisioning.

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
    image: nginx:alpine                              # your app's Docker image
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
docker compose up -d
```

That's it. SSL is automatic. Your app is live at `https://myapp.yourdomain.com`.

> **First deploy:** The SSL certificate takes a few seconds to provision on first access. Subsequent requests will be fast.

### Removing an app

```bash
cd /path/to/myapp
docker compose down
```

Then remove the DNS record in Cloudflare. The SSL certificate will expire on its own (no action needed).

## Day-to-Day Operations

### Updating an app (new image version)

```bash
cd /path/to/myapp
# Edit compose.yaml with the new image version
docker compose pull
docker compose up -d
```

The old container is replaced. No downtime if the new image starts successfully.

### Changing the dashboard password

1. Generate a new hash: `htpasswd -nb admin "new-password" | sed -e 's/\$/\$\$/g'`
2. Replace the hash in `traefik/dynamic.yaml`
3. Done — dynamic.yaml auto-reloads, no restart needed

### Viewing logs

This setup logs to files inside the container. Use `docker exec` to view them:

```bash
# View Traefik logs in real-time
docker exec traefik tail -f /var/log/traefik/traefik.log

# View recent logs
docker exec traefik tail -50 /var/log/traefik/traefik.log

# View access logs
docker exec traefik tail -f /var/log/traefik/access.log
```

Logs are stored in the `traefik_logs` Docker volume at `/var/log/traefik/` and persist across container restarts.

### Enabling debug logging

Edit `traefik/traefik.yaml` and change the log level:

```yaml
log:
  level: DEBUG    # change from INFO to DEBUG
```

Then restart Traefik:

```bash
docker compose restart
```

Remember to change back to `INFO` when done — debug logs are verbose.

### After a VM reboot

All containers with `restart: unless-stopped` start automatically. The Docker network and named volumes persist across reboots. No manual intervention needed.

If Traefik doesn't come back after reboot:

```bash
docker ps                          # check if container is running
docker logs traefik --tail 20      # check for errors
docker compose up -d               # manually start if needed
```

### Backing up SSL certificates

Certificates are stored in the `letsencrypt_data` Docker volume. To back up:

```bash
docker run --rm -v traefik-setup_letsencrypt_data:/data -v $(pwd):/backup alpine tar czf /backup/letsencrypt-backup.tar.gz -C /data .
```

To restore:

```bash
docker run --rm -v traefik-setup_letsencrypt_data:/data -v $(pwd):/backup alpine tar xzf /backup/letsencrypt-backup.tar.gz -C /data
```

## Using Multiple Domains

This setup supports any number of domains. Just point the domain's DNS to your VM IP and use the full domain in the `Host()` rule:

```yaml
- "traefik.http.routers.myapp.rule=Host(`myapp.otherdomain.com`)"
```

No Traefik config changes needed. It just works.

## Adding Middleware to an App

You can add rate limiting, IP allowlist, or other middleware per-app via labels.

### Rate limiting

```yaml
labels:
  # ... existing labels ...
  - "traefik.http.middlewares.myapp-ratelimit.ratelimit.average=10"
  - "traefik.http.middlewares.myapp-ratelimit.ratelimit.burst=20"
  - "traefik.http.middlewares.myapp-ratelimit.ratelimit.period=1s"
  - "traefik.http.routers.myapp.middlewares=myapp-ratelimit"
```

### IP allowlist (restrict to specific IPs)

```yaml
labels:
  # ... existing labels ...
  - "traefik.http.middlewares.myapp-ipallowlist.ipallowlist.sourcerange=YOUR_IP/32"
  - "traefik.http.routers.myapp.middlewares=myapp-ipallowlist"
```

### Security headers (defined in dynamic.yaml)

```yaml
labels:
  # ... existing labels ...
  - "traefik.http.routers.myapp.middlewares=security-headers@file"
```

### Combining multiple middlewares

Separate middleware names with commas:

```yaml
  - "traefik.http.routers.myapp.middlewares=security-headers@file,myapp-ratelimit"
```

## Configuration Files

| File | What | Edit when |
|------|------|-----------|
| `.env` | ACME email for Let's Encrypt | Initial setup only |
| `compose.yaml` | Traefik container definition | Rarely (version upgrades) |
| `traefik/traefik.yaml` | Static config (entrypoints, providers, logging) | When changing log level or restart needed |
| `traefik/dynamic.yaml` | Dashboard auth, TLS options, security headers, IP allowlist | Initial setup only |
| `templates/app-compose.yaml` | Template for new apps | Never (copy it) |

**Important:** `traefik.yaml` changes require a container restart (`docker compose restart`). `dynamic.yaml` auto-reloads without restart — but be careful, errors apply immediately.

## Security

This setup includes:

- **Dashboard** behind basic auth, no insecure port exposed
- **Docker socket** mounted read-only
- **Container** runs with `no-new-privileges`
- **TLS 1.2 minimum** with modern cipher suites
- **HSTS** enabled with preload (2 years)
- **Security headers** (content-type nosniff, server header hidden, referrer same-origin)
- **HTTP to HTTPS redirect** on port 80
- **No telemetry** (`sendAnonymousUsage: false`)
- **Secrets** in `.env` (excluded from git via `.gitignore`)
- **Access logging** enabled for audit trail

## Troubleshooting

### App not reachable

```bash
# Is the container running?
docker ps | grep myapp

# Is it on the right network?
docker network inspect traefik-network

# Check Traefik logs
docker logs traefik --tail 50
```

Common causes:
- Container not on `traefik-network` — add `networks: [traefik-network]` to compose
- Missing `traefik.enable=true` label
- Wrong port in `loadbalancer.server.port` label
- DNS not pointing to your VM — check with `dig myapp.yourdomain.com`

### SSL certificate not issued

```bash
# Check Traefik logs for ACME errors
docker logs traefik 2>&1 | grep -i acme

# Verify DNS resolves to your VM
dig myapp.yourdomain.com

# Make sure port 80 is open (needed for HTTP-01 challenge)
curl -I http://myapp.yourdomain.com
```

Common causes:
- DNS not propagated yet — wait a few minutes after adding the record
- Port 80 blocked by firewall — Let's Encrypt needs to reach port 80 for the challenge
- Cloudflare proxy is ON (orange cloud) — switch to DNS only (grey cloud)
- Rate limit hit — see below

### Let's Encrypt rate limits

Let's Encrypt limits certificate issuance. If you're testing, you might hit these limits:
- **5 failed validations per hour** per domain
- **50 certificates per week** per registered domain
- **5 duplicate certificates per week**

If you hit a rate limit during testing, wait or use the [staging server](#using-staging-server-for-testing) first.

### SSL cert failed — how to retry

If a certificate request failed and you've fixed the issue (DNS, firewall, etc.):

```bash
# Traefik retries automatically on next request. Just access the domain again.
# If it still fails, check logs for the specific error:
docker logs traefik 2>&1 | grep -i acme
```

Traefik retries failed certificates automatically. You don't need to delete `acme.json` or restart.

### Dashboard not loading

```bash
# Check if the router is registered
docker logs traefik 2>&1 | grep dashboard

# Verify DNS for traefik subdomain
dig traefik.yourdomain.com

# Test without SSL (should redirect to HTTPS)
curl -I http://traefik.yourdomain.com
```

Common causes:
- Dashboard domain not updated in `dynamic.yaml` — still says `YOUR-DOMAIN`
- DNS record not added for `traefik` subdomain
- htpasswd hash incorrect — regenerate with the `htpasswd` command

### Encoded characters warning in logs

You may see this warning on startup:

```
WRN Traefik can reject some encoded characters in the request path.
```

This is **informational, not an error**. Traefik v3.6.7+ tightened URL encoding security per RFC 3986. Most apps work fine with the default settings.

If your backend app uses non-standard URL encoding and you see "404" or "split view" issues, you can relax this restriction by adding to `traefik/traefik.yaml`:

```yaml
experimental:
  encodedCharacters:
    percentMode: RejectUnvalid  # or AllowAll if needed (less secure)
```

For most setups, you can ignore this warning.

### Traefik won't start

```bash
# Check for config errors
docker logs traefik 2>&1 | head -20

# Validate compose file
docker compose config

# Common causes:
# - Port 80 or 443 already in use by another process
# - traefik-network doesn't exist — run: docker network create traefik-network
# - Invalid YAML in config files
```

### Container name conflict

If you see `container name "myapp" is already in use`:

```bash
# The old container is still there. Remove it:
docker rm -f myapp
docker compose up -d
```

This happens if you change the `container_name` in compose but the old container is still running.

## Advanced

### Using staging server for testing

To avoid Let's Encrypt rate limits while testing, temporarily use the staging server. Edit `traefik/traefik.yaml` and add the `caServer` line:

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      caServer: https://acme-staging-v02.api.letsencrypt.org/directory
      storage: "/letsencrypt/acme.json"
      httpChallenge:
        entryPoint: web
```

Staging certificates will show as untrusted in browsers (that's expected). Remove the `caServer` line when ready for production.

### Upgrading Traefik

1. Check the [Traefik release notes](https://github.com/traefik/traefik/releases) for breaking changes
2. Update the image version in `compose.yaml`:
   ```yaml
   image: traefik:v3.x.x
   ```
3. Restart:
   ```bash
   docker compose up -d
   ```
4. Verify:
   ```bash
   docker logs traefik --tail 20
   ```

### Using Cloudflare proxy (orange cloud)

If you want to use Cloudflare's proxy (orange cloud) instead of DNS-only:
1. Set Cloudflare SSL/TLS mode to **Full (Strict)**
2. Switch to DNS-01 challenge (HTTP-01 doesn't work behind proxy)
3. See Traefik's [DNS challenge documentation](https://doc.traefik.io/traefik/v3.0/user-guides/docker-compose/acme-dns/)

### Custom error pages

You can add a custom error page middleware in `dynamic.yaml`:

```yaml
middlewares:
  my-error-pages:
    errors:
      status:
        - "404"
        - "500-599"
      query: "/{status}.html"
      service: error-pages-service
```

## Cloudflare Settings

For this setup, use these Cloudflare settings:

- **SSL/TLS mode:** Full (Strict) — Traefik has valid Let's Encrypt certs
- **Always Use HTTPS:** Can be ON or OFF (Traefik already redirects)
- **Proxy status:** DNS only (grey cloud) for all records — Let's Encrypt handles SSL directly

## Examples

The `examples/` folder contains reference configurations. Each example has its own `README.md` with activation instructions.

### SSL Setup (`examples/ssl-setup/`)

whoami service with rate limiting (3 req/10sec) and IP allowlist. Ready to use — just replace the domain and IP in the labels.

### Canary Deployment (`examples/canary-deployment/`)

Weighted round-robin (90/10 traffic split) between two nginx services. Requires uncommenting the WRR service and router in `traefik/dynamic.yaml`. See `examples/canary-deployment/README.md`.

### Sablier Zero-Scale (`examples/sablier-zero-scale/`)

Auto-hibernate idle containers with a loading page. Requires uncommenting the plugin in `traefik/traefik.yaml` AND the router/service/middleware in `traefik/dynamic.yaml`. See `examples/sablier-zero-scale/README.md`.

These are for learning. Your production setup is the root config + the template.
