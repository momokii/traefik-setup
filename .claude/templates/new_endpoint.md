# New Traefik Route / Service Endpoint Checklist

## Before Starting
- [ ] Route requirements are clear: domain, entrypoint (web/websecure), TLS requirements
- [ ] Middleware requirements identified: rate limiting, IP allowlist, auth, etc.
- [ ] Active environment confirmed as development

## Implementation — Docker Labels (Preferred for Single Services)

- [ ] `traefik.enable=true` label present on the service
- [ ] Service name follows convention: `descriptive-name-service` (e.g., `app1-service`)
- [ ] Router name follows convention: `descriptive-name-router` (e.g., `app1-router`)
- [ ] Port specified: `traefik.http.services.<name>-service.loadbalancer.server.port=<port>`
- [ ] Host rule uses placeholder: `traefik.http.routers.<name>-router.rule=Host(\`YOUR-DOMAIN\`)`
- [ ] Entrypoint set: `traefik.http.routers.<name>-router.entrypoints=websecure`
- [ ] TLS cert resolver set: `traefik.http.routers.<name>-router.tls.certresolver=myresolver`
- [ ] Middleware chain applied if needed: `traefik.http.routers.<name>-router.middlewares=<mw>@docker,<mw>@file`

## Implementation — Dynamic Config (For Multi-Service Routing)

- [ ] Service defined in `http.services` section of `dynamic-config.yaml`
- [ ] Router defined in `http.routers` section referencing the service
- [ ] Middleware defined in `http.middlewares` section if new middleware needed
- [ ] Provider references correct: `@file` for dynamic config resources, `@docker` for label-defined resources

## Security Review
- [ ] No real domain names in committed config — use `YOUR-DOMAIN` placeholder
- [ ] TLS enabled on all public-facing routes (use `websecure` entrypoint)
- [ ] Middleware applied for sensitive endpoints (rate limiting, IP allowlist)
- [ ] Service port not exposed directly to the host — only via Traefik

## Validation
- [ ] YAML syntax valid
- [ ] Service appears in Traefik dashboard (`http://localhost:8080`)
- [ ] Route is listed under HTTP Routers in the dashboard
- [ ] TLS certificate status correct (if real domain configured)

## Completion
- [ ] CURRENT_STATUS.md and TASK_QUEUE.md updated
- [ ] README.md updated if the new route adds a new example pattern
