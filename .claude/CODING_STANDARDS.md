# Coding Standards

Derived from the actual codebase conventions observed in this repository.

## Project Type

This is a **configuration-only** repository. There is no application code, no build step, no test framework, and no CI pipeline. All work involves YAML configuration files (Docker Compose and Traefik configuration).

## File Organization

### Where New Files Go

| File Type | Location | Naming |
|---|---|---|
| New example setup | `new-example-name/compose.yaml` | kebab-case directory name |
| Traefik static config | `traefik/traefik-config.yaml` | Fixed name |
| Traefik dynamic config | `traefik/dynamic-config.yaml` | Fixed name |
| Test HTML content | `example-name/htmlN/index.html` | Sequential numbering |
| Documentation | `README.md` (root) | Fixed name |

### Directory Structure Pattern

Each example follows this structure:
```
example-name/
├── compose.yaml        # Required — Docker Compose service definition
└── (optional assets)   # HTML files, config mounts, etc.
```

## Naming Conventions

### Directories
- kebab-case: `ssl-setup-test`, `canary-deployment-test`, `sablier-test-zero-scale`
- Descriptive of the feature being demonstrated

### Docker Compose
- `version` field: `"3.9"` or `"3.8"` (both used — prefer `"3.9"` for new files)
- Service names: kebab-case (`nginx-first`, `nginx-second`, `whoami`)
- Container names: snake_case (`whoami_sablier`, `whoami-ssl`)
- Network names: kebab-case (`traefik-networks`)

### Traefik Resources (in Docker labels and dynamic config)
- Services: kebab-case with `-service` suffix (`app1-service`, `app-v1-service`)
- Routers: kebab-case with `-router` suffix (`app1-router`, `app-ab-test-router`)
- Middlewares: kebab-case with descriptive prefix (`test-ratelimit`, `test-ipallowlist`)
- Provider reference suffix: `@docker` for label-defined, `@file` for dynamic-config-defined

### Placeholder Values
- Domain: `YOUR-DOMAIN`, `YOUR-AB-TEST-DOMAIN`
- Email: `YOUR_EMAIL_HERE`
- IP: `YOUR_IP_ADDRESS` or `YOUR IP ADDRESS THAT WILL BE ALLOWED`
- Never use real values in committed files

## Docker Compose Patterns

### Network Configuration

Every `compose.yaml` must include:
```yaml
networks:
  default:
  traefik-networks:
    external: true
```

Every service that needs Traefik routing must include:
```yaml
networks:
  - traefik-networks
```

### Traefik Label Pattern

Services routed through Traefik must include:
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.services.<service-name>.loadbalancer.server.port=<port>"
  - "traefik.http.routers.<router-name>.rule=Host(`YOUR-DOMAIN`)"
  - "traefik.http.routers.<router-name>.entrypoints=websecure"
  - "traefik.http.routers.<router-name>.tls.certresolver=myresolver"
```

Services NOT using Traefik must NOT include any traefik labels.

### Volume Mount Pattern
- Config files: `./filename:/etc/traefik/filename:ro` (read-only)
- Data directories: `./dirname:/mountpoint` (read-write for mutable data like certs)
- Docker socket: `/var/run/docker.sock:/var/run/docker.sock` (required for Traefik and Sablier)

## Traefik Configuration Patterns

### Static Config (`traefik-config.yaml`)
- EntryPoints: `web` (port 80) and `websecure` (port 443)
- HTTP-to-HTTPS redirect configured on the `web` entrypoint
- Docker provider with `exposedByDefault: false`
- File provider with `watch: true` for dynamic config auto-reload
- Certificate resolver uses Let's Encrypt with HTTP-01 challenge

### Dynamic Config (`dynamic-config.yaml`)
- Three top-level sections: `services`, `middlewares`, `routers`
- Weighted services use the `weighted` key with explicit weights
- Load-balanced services use the `loadBalancer` key with direct server URLs
- Middleware references use `@file` suffix (e.g., `my-sablier@file`)
- All routers targeting HTTPS use `entryPoints: websecure`

## Comment Style

- Inline comments explaining **why** something is configured a certain way
- Section headers in ALL CAPS (e.g., `# SETUP THE PLUGIN`, `# === SETUP THE ROUTERS CONFIGURATIONS`)
- Comments on Docker labels explain what each label controls
- References to related configuration (e.g., "make sure the name here match the certResolver name")
- New configurations must follow this comment style

## Error Handling

This project has no application-level error handling. Traefik's own error responses are used. If custom error pages are needed, they should be implemented via Traefik middleware.

## Dependencies

- No package manager — only Docker images
- Image versions should be pinned (e.g., `traefik:v3.1.5`, `sablierapp/sablier:1.10.1`)
- Avoid using `latest` tag for production-relevant images (note: `traefik/whoami:latest` is currently used for test services — this is acceptable for disposable test containers)
- New images must be discussed with the user before introduction

## Patterns to Follow

1. All services set `restart: unless-stopped`
2. All services connect to the external `traefik-networks` network
3. Configuration changes in `dynamic-config.yaml` take effect immediately (no restart needed)
4. Static config changes (`traefik-config.yaml`) require a Traefik container restart
5. Each example directory is self-contained with its own `compose.yaml`

## Patterns Explicitly Forbidden

1. Never use `exposedByDefault: true` in the Docker provider — always use explicit `traefik.enable=true` labels
2. Never hardcode real domain names, emails, or IPs in committed files
3. Never mount the Docker socket in containers that don't need it
4. Never disable HTTP-to-HTTPS redirection in production configurations
5. Never use the insecure API dashboard (`api.insecure: true`) in production
