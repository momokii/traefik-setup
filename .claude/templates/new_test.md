# New Test / Validation Checklist

> **Note:** This project has no automated test framework. Testing is done by deploying configurations and verifying behavior manually. This checklist covers validation procedures for configuration changes.

## Before Starting
- [ ] Test objective clearly defined — what specific routing or middleware behavior is being verified
- [ ] Test type identified: configuration validation / routing verification / TLS check / middleware behavior
- [ ] Active environment confirmed as development

## Manual Test Procedures

### Configuration Validation
- [ ] YAML syntax validated: `docker-compose -f <path>/compose.yaml config`
- [ ] Dynamic config YAML validated (use `docker logs traefik` to check for parse errors)
- [ ] No duplicate service/router/middleware names across all compose files and dynamic config

### Routing Verification
- [ ] Service starts without errors: `docker-compose up -d`
- [ ] Service visible in Traefik dashboard: `http://localhost:8080`
- [ ] Route accessible via configured domain (if DNS configured) or `curl -H "Host: <domain>" http://localhost`
- [ ] HTTP-to-HTTPS redirect works: `curl -I http://localhost -H "Host: <domain>"` returns 301/302

### TLS Verification
- [ ] Certificate generated successfully: check `docker logs traefik` for ACME activity
- [ ] HTTPS endpoint returns valid certificate: `curl -v https://<domain>`
- [ ] Certificate stored in `letsencrypt_data/acme.json`

### Middleware Verification
- [ ] Rate limiting works: send rapid requests and verify 429 response after limit
- [ ] IP allowlist works: request from non-allowed IP receives 403
- [ ] Sablier plugin works: service stops after inactivity, shows loading page on next request, then restarts

### Canary / Load Balancing Verification
- [ ] Traffic distribution matches weights (e.g., 90/10 split): send multiple requests and count responses
- [ ] Both service versions are reachable independently

## Completion
- [ ] All validation steps passed
- [ ] CURRENT_STATUS.md updated
- [ ] Any issues found added to TASK_QUEUE.md
