# Task Queue

## Security Remediation Tasks

| Field               | Value                                               |
|---------------------|-----------------------------------------------------|
| Task ID             | TASK-001                                            |
| Name                | Create `.gitignore` for sensitive files             |
| Priority            | High                                                |
| Status              | TODO                                                |
| Complexity          | S                                                   |
| Depends On          | None                                                |
| Scope               | Create `.gitignore` excluding `acme.json`, `letsencrypt_data/`, `.env`, `*.env` |
| Acceptance Criteria | `.gitignore` exists and covers all sensitive file patterns |
| Security Concerns   | Without this, TLS certificates or secrets could be committed |
| Source              | Phase 1 security audit (SEC-001)                    |

| Field               | Value                                               |
|---------------------|-----------------------------------------------------|
| Task ID             | TASK-002                                            |
| Name                | Fix unclosed backtick in dynamic-config.yaml        |
| Priority            | Medium                                              |
| Status              | TODO                                                |
| Complexity          | S                                                   |
| Depends On          | None                                                |
| Scope               | Fix `Host(\`YOUR-DOMAIN-HERE)` → `Host(\`YOUR-DOMAIN-HERE\`)` on line 55 of `traefik/dynamic-config.yaml` |
| Acceptance Criteria | Valid YAML and correct Traefik Host rule syntax     |
| Security Concerns   | None — placeholder value                            |
| Source              | Phase 1 audit (SEC-004)                             |

## Feature Tasks

| Field               | Value                                               |
|---------------------|-----------------------------------------------------|
| Task ID             | TASK-003                                            |
| Name                | Add authentication middleware example               |
| Priority            | Low                                                 |
| Status              | TODO                                                |
| Complexity          | M                                                   |
| Depends On          | None                                                |
| Scope               | Create example showing Traefik BasicAuth or ForwardAuth middleware setup |
| Acceptance Criteria | New directory with compose.yaml, documented in README.md |
| Security Concerns   | Auth examples must use placeholder credentials only |
| Source              | Open question from audit                            |

| Field               | Value                                               |
|---------------------|-----------------------------------------------------|
| Task ID             | TASK-004                                            |
| Name                | Standardize compose.yaml version fields             |
| Priority            | Low                                                 |
| Status              | TODO                                                |
| Complexity          | S                                                   |
| Depends On          | None                                                |
| Scope               | Change all `version: "3.8"` to `version: "3.9"` for consistency (only `canary-deployment-test/compose.yaml` uses 3.8) |
| Acceptance Criteria | All compose.yaml files use `version: "3.9"`        |
| Security Concerns   | None                                                |
| Source              | Convention inconsistency observed in audit          |

| Field               | Value                                               |
|---------------------|-----------------------------------------------------|
| Task ID             | TASK-005                                            |
| Name                | Pin `latest` tags to specific versions              |
| Priority            | Low                                                 |
| Status              | TODO                                                |
| Complexity          | S                                                   |
| Depends On          | None                                                |
| Scope               | Replace `traefik/whoami:latest` with a pinned version in `ssl-setup-test/compose.yaml` and `sablier-test-zero-scale/compose.yaml` |
| Acceptance Criteria | No `latest` tags in any compose.yaml                |
| Security Concerns   | Reproducible builds — avoid unexpected image updates |
| Source              | Best practice gap observed in audit                 |
