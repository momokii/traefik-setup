# Bug Fix Checklist

## Reproduce First
- [ ] Bug is reproducible — document exact reproduction steps
- [ ] Expected behavior clearly stated
- [ ] Actual broken behavior clearly stated
- [ ] Reproduction performed in development environment only

## Root Cause Analysis
- [ ] Check Traefik logs for errors: `docker logs traefik`
- [ ] Check service logs: `docker logs <container-name>`
- [ ] Verify YAML syntax is valid: `docker-compose -f <path>/compose.yaml config`
- [ ] Verify Traefik dashboard shows expected routing state: `http://localhost:8080`
- [ ] Check if dynamic config was auto-reloaded correctly (file watcher)
- [ ] Verify network connectivity: `docker exec traefik ping <service-name>`
- [ ] Checked whether the same bug exists in other compose files or dynamic config sections

## Common Root Causes in This Project
- Missing closing backtick in `Host(\`...\`)` rule
- Mismatched service/router/middleware names between compose labels and dynamic config
- Service not on `traefik-networks` network
- `traefik.enable=true` label missing
- Incorrect provider suffix (`@docker` vs `@file`)
- Typo in certResolver name (must match `myresolver` in static config)
- Port mismatch between label and actual container port

## Fix
- [ ] Minimal, targeted fix applied — no opportunistic changes
- [ ] Fix resolves only the stated bug
- [ ] YAML syntax re-validated after fix

## Verification
- [ ] Bug is no longer reproducible with the fix applied
- [ ] Affected services restart cleanly: `docker-compose up -d`
- [ ] Traefik dashboard shows correct routing after fix
- [ ] No other routes or services broken by the change

## Completion
- [ ] DECISIONS_LOG.md updated if root cause revealed an important insight
- [ ] CURRENT_STATUS.md updated
- [ ] CODING_STANDARDS.md updated if the bug revealed a convention that should be documented
