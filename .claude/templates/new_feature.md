# New Feature (Example Setup) Checklist

## Before Starting
- [ ] Task exists in TASK_QUEUE.md with clear acceptance criteria
- [ ] All task dependencies are complete
- [ ] Relevant sections of `traefik/traefik-config.yaml` and `traefik/dynamic-config.yaml` have been read and understood
- [ ] Active environment identified and confirmed as development

## Design
- [ ] New directory created with kebab-case name (e.g., `new-feature-name/`)
- [ ] Scope defined — list every file to be created or modified:
  - `compose.yaml` in the new directory
  - Optional: static assets (HTML, config files)
  - Optional: updates to `traefik/dynamic-config.yaml` if shared routing/middleware needed
  - Optional: updates to `README.md` with new example documentation
- [ ] Service image selected and pinned to a specific version (not `latest`)
- [ ] Security implications assessed — no real credentials, no unnecessary port exposure

## Implementation
- [ ] `compose.yaml` follows the project pattern:
  - `version: "3.9"`
  - Service with `restart: unless-stopped`
  - `traefik.enable=true` label if service needs routing
  - Correct `traefik-networks` external network configuration
  - Inline comments explaining each section (matching existing comment style)
- [ ] If Traefik labels are used, service/router/middleware names follow naming conventions in CODING_STANDARDS.md
- [ ] If dynamic config changes needed, `dynamic-config.yaml` updated following existing section structure
- [ ] All sensitive values use placeholder pattern (`YOUR-DOMAIN`, `YOUR_EMAIL_HERE`, etc.)

## Security Review
- [ ] No real domain names, emails, IPs, or secrets in committed files
- [ ] Docker socket mounted only if the service requires it
- [ ] Service does not expose unnecessary ports to the host
- [ ] `.gitignore` covers any new generated files (e.g., cert data)

## Validation
- [ ] YAML syntax valid: `docker-compose -f <path>/compose.yaml config`
- [ ] Service starts successfully: `docker-compose up -d`
- [ ] Traefik routes to the service correctly (check dashboard at `http://localhost:8080`)
- [ ] Service accessible via configured route (if domains are set up)

## Completion
- [ ] TASK_QUEUE.md updated — task marked DONE
- [ ] CURRENT_STATUS.md updated with session summary
- [ ] README.md updated with new example documentation (in Configuration Examples section)
- [ ] DECISIONS_LOG.md updated if any significant decision was made
