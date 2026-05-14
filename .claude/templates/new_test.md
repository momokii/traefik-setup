# New Test / Validation Checklist

> **Note:** This project has no automated test framework. Testing is done by deploying configurations and verifying behavior manually.

## Before Starting
- [ ] Test objective clearly defined — what specific routing or middleware behavior is being verified
- [ ] Test type identified: configuration validation / routing verification / TLS check / middleware behavior

## Manual Test Procedures

### Configuration Validation
- [ ] YAML syntax validated: `docker compose -f <path>/compose.yaml config`
- [ ] Dynamic config YAML validated (use `docker logs traefik` to check for parse errors)
- [ ] No duplicate service/router/middleware names across all compose files and dynamic config

### Routing Verification
- [ ] Service starts without errors: `docker compose up -d`
- [ ] Service visible in Traefik dashboard
- [ ] Route accessible: `curl -H "Host: <domain>" http://localhost`
- [ ] HTTP-to-HTTPS redirect works: `curl -I http://<domain>` returns 301/302

### TLS Verification
- [ ] Certificate generated successfully: check `docker logs traefik` for ACME activity
- [ ] HTTPS endpoint returns valid certificate: `curl -v https://<domain>`

### Middleware Verification
- [ ] Rate limiting works: send rapid requests and verify 429 response after limit
- [ ] IP allowlist works: request from non-allowed IP receives 403
- [ ] Security headers present in response: `curl -I https://<domain>`

## Completion
- [ ] All validation steps passed
- [ ] CURRENT_STATUS.md updated
- [ ] Any issues found added to TASK_QUEUE.md
