# Environment Guide

## Environment Definitions

| Environment | Purpose | Characteristics |
|---|---|---|
| `development` | Local testing and learning | Insecure dashboard on, all examples running, placeholder domains, no real certs |
| `production` | Live deployment | Dashboard secured/authenticated, real domains and certs, hardened config |

> **Note**: This project has no staging environment. It is a reference/example repository, not a deployed application. Production here means "deploying these configs to a live server."

## Agent Behavior by Environment

### In Development
- Traefik dashboard accessible at `http://localhost:8080` without auth
- Placeholder values in configs are expected
- All example services can be started/stopped freely
- Debug logging can be enabled by changing `log.level` to `DEBUG` in `traefik-config.yaml`

### In Production
- Never run destructive commands (`docker-compose down`, `docker rm`) without explicit written confirmation
- Never directly modify production config files without presenting a plan first
- Verify all placeholder values have been replaced before deploying
- Dashboard must be secured — `api.insecure: true` must be set to `false`

## Verified Commands

### Start Development Environment

```bash
# Create the external network (first time only)
docker network create traefik-networks

# Start Traefik core
cd traefik && docker-compose up -d

# Start individual example services
cd ../ssl-setup-test && docker-compose up -d        # SSL/TLS example
cd ../canary-deployment-test && docker-compose up -d # Canary deployment
cd ../sablier-test-zero-scale && docker-compose up -d # Zero-scale example
```

### Verify Environment Health

```bash
# Check Traefik is responding
curl http://localhost:8080/ping
# Expected: "OK"

# Check running containers
docker ps

# View Traefik configuration and routing state
curl http://localhost:8080/api/rawdata
```

### Validate Configuration Changes

```bash
# Validate a compose file
docker-compose -f <path-to-compose.yaml> config

# Check Traefik logs for config errors
docker logs traefik

# Monitor real-time Traefik logs
docker logs -f traefik
```

### Stop Environment

```bash
# Stop all services in a directory
cd <example-directory> && docker-compose down

# Stop Traefik core
cd traefik && docker-compose down
```

## Docker Compose Layout

| File | Purpose |
|---|---|
| `traefik/compose.yaml` | Core Traefik reverse proxy — always start first |
| `ssl-setup-test/compose.yaml` | SSL/TLS test service — depends on Traefik |
| `canary-deployment-test/compose.yaml` | Canary deployment test — depends on Traefik + dynamic config |
| `sablier-test-zero-scale/compose.yaml` | Zero-scale test — depends on Traefik + dynamic config |

Each compose file is independent and can be started/stopped individually. All depend on the `traefik-networks` external network and a running Traefik instance.

## Configuration Files

| File | Type | Reload Required? |
|---|---|---|
| `traefik/traefik-config.yaml` | Static | Yes — restart Traefik container |
| `traefik/dynamic-config.yaml` | Dynamic | No — auto-reloaded on file change (`watch: true`) |
| `*/compose.yaml` | Infrastructure | Yes — `docker-compose up -d` to apply |

## Known Gotchas

1. **External network must exist** before starting any service — `docker network create traefik-networks`
2. **Traefik must start first** — example services will fail to route without it
3. **Port conflicts** — if ports 80, 443, or 8080 are in use, services will fail to start
4. **Dynamic config changes are immediate** — changes to `dynamic-config.yaml` are auto-applied with no restart, so errors take effect immediately
5. **acme.json permissions** — if Let's Encrypt certificate generation fails, check that `letsencrypt_data/` is writable and `acme.json` has correct permissions (600)
