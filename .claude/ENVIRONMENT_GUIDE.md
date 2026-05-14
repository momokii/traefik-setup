# Environment Guide

## Setup (One-Time)

```bash
# 1. Create the external Docker network
docker network create traefik-network

# 2. Configure environment
cp .env.example .env
# Edit .env with real domain, email, credentials

# 3. Set up dashboard auth in traefik/dynamic.yaml
htpasswd -nb admin "your-password" | sed -e 's/\$/\$\$/g'
# Paste output into dynamic.yaml under dashboard-auth.users
# Also update the domain in the dashboard router rule

# 4. Start Traefik
docker-compose up -d
```

## Verified Commands

### Health Check
```bash
docker logs traefik
docker ps | grep traefik
```

### Validate Configuration
```bash
docker-compose config                    # Validate compose syntax
docker logs traefik 2>&1 | grep -i error # Check for config errors
```

### Stop Traefik
```bash
docker-compose down
```

## Configuration Files

| File | Type | Reload Required? |
|---|---|---|
| `traefik/traefik.yaml` | Static | Yes — `docker-compose restart` |
| `traefik/dynamic.yaml` | Dynamic | No — auto-reloaded on file change |
| `compose.yaml` | Infrastructure | Yes — `docker-compose up -d` |
| `.env` | Secrets | Yes — `docker-compose up -d` |

## Adding a New App

```bash
# On your VM, in the app's directory:
cp /path/to/traefik-setup/templates/app-compose.yaml compose.yaml
# Edit the 4 CHANGE ME values
docker-compose up -d
```

## Gotchas

1. **External network must exist** before starting — `docker network create traefik-network`
2. **Traefik must start first** — apps can't route without it
3. **Port conflicts** — if 80 or 443 are in use, Traefik won't start
4. **Dynamic config changes are immediate** — errors take effect right away
5. **DNS propagation** — new subdomains need time to propagate before SSL certs are issued
