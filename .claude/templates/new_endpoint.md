# New Traefik Route / Service Endpoint Checklist

## Before Starting
- [ ] Route requirements are clear: domain, entrypoint (web/websecure), TLS requirements
- [ ] Middleware requirements identified: rate limiting, IP allowlist, auth, etc.

## Implementation — Docker Labels (Preferred for Single Services)

- [ ] `traefik.enable=true` label present on the service
- [ ] Port specified: `traefik.http.services.<name>.loadbalancer.server.port=<port>`
- [ ] Host rule uses placeholder: `traefik.http.routers.<name>.rule=Host(\`YOUR-DOMAIN\`)`
- [ ] Entrypoint set: `traefik.http.routers.<name>.entrypoints=websecure`
- [ ] TLS cert resolver set: `traefik.http.routers.<name>.tls.certresolver=letsencrypt`
- [ ] Middleware chain applied if needed: `traefik.http.routers.<name>.middlewares=<mw>@docker,<mw>@file`

## Implementation — Dynamic Config (For Multi-Service Routing)

- [ ] Service defined in `http.services` section of `dynamic.yaml`
- [ ] Router defined in `http.routers` section referencing the service
- [ ] Middleware defined in `http.middlewares` section if new middleware needed
- [ ] Provider references correct: `@file` for dynamic config resources, `@docker` for label-defined resources

## Security Review
- [ ] No real domain names in committed config — use placeholder
- [ ] TLS enabled on all public-facing routes (use `websecure` entrypoint)
- [ ] Middleware applied for sensitive endpoints (rate limiting, IP allowlist)
- [ ] Service port not exposed directly to the host — only via Traefik

## Validation
- [ ] YAML syntax valid
- [ ] Service appears in Traefik dashboard
- [ ] Route is listed under HTTP Routers in the dashboard
- [ ] TLS certificate status correct (if real domain configured)

## Completion
- [ ] CURRENT_STATUS.md and TASK_QUEUE.md updated
- [ ] README.md updated if the new route adds a new pattern
