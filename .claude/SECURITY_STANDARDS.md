# Security Standards

## Audit Findings Summary (2026-05-13)

### Current Security Posture: YELLOW (minor issues, no critical vulnerabilities)

The repository is a configuration-only project with no application code, no secrets, and no runtime data processing. Security concerns are limited to infrastructure configuration.

### Issues Found

#### Issue SEC-001: Missing `.gitignore` (Medium)
- **What**: No `.gitignore` file exists in the repository
- **Risk**: Sensitive files like `acme.json` (TLS certificates), `.env` files, or `letsencrypt_data/` could be accidentally committed
- **Status**: TODO — see TASK_QUEUE.md TASK-001

#### Issue SEC-002: Insecure API Dashboard Enabled (Low, testing-only)
- **What**: `traefik-config.yaml` has `api.insecure: true` which exposes the dashboard without authentication on port 8080
- **Risk**: Anyone with network access can view full routing configuration, service details, and TLS certificate info
- **Status**: Acceptable for development/testing — must be disabled or secured with auth for production
- **File**: `traefik/traefik-config.yaml:9`

#### Issue SEC-003: Docker Socket Mounted (Low, required)
- **What**: Both Traefik (`traefik/compose.yaml`) and Sablier (`sablier-test-zero-scale/compose.yaml`) mount `/var/run/docker.sock`
- **Risk**: Container escape vector — if Traefik or Sablier is compromised, attacker gains full Docker control
- **Status**: Required for Traefik and Sablier functionality — standard practice. Mitigate by limiting what containers run on the same host and keeping images updated

#### Issue SEC-004: Unclosed Backtick in Host Rule (Low)
- **What**: `dynamic-config.yaml:55` has `Host(\`YOUR-DOMAIN-HERE)` — missing closing backtick
- **Risk**: Malformed routing rule could cause unexpected behavior or prevent the route from matching
- **Status**: TODO — see TASK_QUEUE.md TASK-002

#### Issue SEC-005: No Container User Restrictions (Low)
- **What**: No containers specify `user:` or `userns:` — all run as root (default Docker behavior)
- **Risk**: If a container is compromised, the attacker runs as root inside the container
- **Status**: Low priority for test/example containers — should be addressed for production deployments

### Positive Security Practices Observed

- `docker.exposedByDefault: false` — services must explicitly opt-in to Traefik routing
- `global.sendAnonymousUsage: false` — no data sent to Traefik
- `global.checkNewVersion: false` — no automatic outbound calls
- Placeholder values used consistently for sensitive configuration (domains, email, IPs)
- TLS certificates stored in a dedicated volume (`letsencrypt_data`)
- HTTP-to-HTTPS redirect enforced at the entrypoint level

## Standing Security Requirements

### Secrets & Sensitive Data

- Never commit real domain names, email addresses, IP addresses, or API keys
- All sensitive values in configuration files must use the placeholder pattern: `YOUR-DOMAIN`, `YOUR_EMAIL_HERE`, `YOUR_IP_ADDRESS`
- The `.gitignore` file must exclude: `acme.json`, `letsencrypt_data/`, `.env`, `*.env`

### TLS Configuration

- Always use HTTPS in production (enforced by `web` → `websecure` redirect)
- Never disable the HTTP-to-HTTPS redirect in production configurations
- Monitor certificate renewal — `acme.json` must be backed up regularly in production
- Use HTTP-01 challenge for most scenarios; DNS-01 for wildcard certificates

### Docker Security

- Never use `exposedByDefault: true` — always require explicit `traefik.enable=true` labels
- Limit Docker socket mounts to containers that require them (Traefik, Sablier)
- Pin image versions — avoid `latest` tag for production services
- Consider running containers as non-root users in production deployments
- Never expose the Traefik dashboard (port 8080) to the public internet without authentication

### Network Security

- The `traefik-networks` external network must not be shared with untrusted containers
- Use IP allowlists for sensitive internal services
- Implement rate limiting on all public-facing endpoints

### Configuration Validation

- After modifying any configuration, validate YAML syntax before deploying
- After modifying `dynamic-config.yaml`, verify Traefik accepted the change via `docker logs traefik`
- After modifying `compose.yaml`, validate with `docker-compose config`

### Production Hardening Checklist

When deploying to production:
1. Set `api.insecure: false` and configure authentication for the dashboard
2. Remove port 8080 from public exposure
3. Replace all `YOUR-*` placeholders with real values
4. Ensure `.gitignore` covers all sensitive files
5. Pin all image versions (replace `latest` tags)
6. Consider adding `securityOpt` and `user` directives to containers
7. Enable access logs for audit trail
