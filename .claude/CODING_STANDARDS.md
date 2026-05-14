# Coding Standards

## Project Type

Configuration-only repository for a production Traefik reverse proxy. No application code, no build step. All work involves YAML configuration files (Docker Compose and Traefik configuration).

## File Organization

| File Type | Location | Naming |
|---|---|---|
| Core compose | `compose.yaml` (root) | Fixed name |
| Traefik static config | `traefik/traefik.yaml` | Fixed name |
| Traefik dynamic config | `traefik/dynamic.yaml` | Fixed name |
| Environment config | `.env` (gitignored) | Fixed name |
| Environment template | `.env.example` | Fixed name |
| App template | `templates/app-compose.yaml` | Fixed name |
| Example setups | `examples/<name>/` | kebab-case directory |
| Documentation | `README.md` (root) | Fixed name |

## Naming Conventions

### Docker Compose
- Service names: kebab-case or simple lowercase (`myapp`, `traefik`)
- Container names: match service name
- Network name: `traefik-network` (external)

### Traefik Resources (in Docker labels and dynamic config)
- Routers: match service name (`myapp` → `traefik.http.routers.myapp`)
- Services: match service name + `-svc` if needed
- Middlewares: descriptive kebab-case (`dashboard-auth`, `security-headers`)
- Provider reference suffix: `@file` for dynamic-config-defined resources

### Placeholder Values
- Domain: `YOUR-DOMAIN`
- Email: `email@example.com`
- Password hash: `replace_with_your_htpasswd_hash`
- Never use real values in committed files

## Docker Compose Patterns

### Network Configuration

Every app's `compose.yaml` must include:
```yaml
networks:
  traefik-network:
    external: true
```

Every service that needs Traefik routing must include:
```yaml
networks:
  - traefik-network
```

### Traefik Label Pattern

Services routed through Traefik must include:
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.<name>.rule=Host(`<subdomain>.<domain>`)"
  - "traefik.http.routers.<name>.entrypoints=websecure"
  - "traefik.http.routers.<name>.tls.certresolver=letsencrypt"
  - "traefik.http.services.<name>.loadbalancer.server.port=<port>"
```

### Volume Mount Pattern
- Static config: `./traefik/traefik.yaml:/etc/traefik/traefik.yaml:ro` (read-only)
- Dynamic config: `./traefik/dynamic.yaml:/etc/traefik/dynamic.yaml` (writable for auto-reload)
- Docker socket: `/var/run/docker.sock:/var/run/docker.sock:ro` (read-only!)
- Data volumes: named volumes (`letsencrypt_data`, `traefik_logs`)

## Traefik Configuration Patterns

### Static Config (`traefik/traefik.yaml`)
- `api.dashboard: true`, `api.insecure: false`
- EntryPoints: `web` (port 80, redirects to HTTPS), `websecure` (port 443)
- Docker provider with `exposedByDefault: false`
- File provider with `watch: true` for dynamic config auto-reload
- Cert resolver: `letsencrypt` with HTTP-01 challenge
- Email passed via `TRAEFIK_CERTIFICATESRESOLVERS_LETSENCRYPT_ACME_EMAIL` env var

### Dynamic Config (`traefik/dynamic.yaml`)
- Contains: dashboard router, auth middleware, TLS options, security headers
- TLS options enforce TLS 1.2 minimum with modern cipher suites
- Changes auto-reload without restart

## Security Requirements

1. Never use `exposedByDefault: true`
2. Never use `api.insecure: true`
3. Always mount Docker socket as `:ro`
4. Always add `security_opt: no-new-privileges:true` to Traefik
5. Never commit real secrets (use `.env` + `.gitignore`)
6. Never disable HTTP-to-HTTPS redirect
