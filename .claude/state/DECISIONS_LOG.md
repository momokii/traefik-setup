# Decisions Log

---

**Decision:** Use external Docker network `traefik-network` for all service communication
**Date:** Pre-takeover (initial commit), updated 2026-05-14
**Context:** Traefik and backend services need shared network for routing
**Rationale:** External network allows independent compose files to connect to the same Traefik instance without being in the same compose project
**Alternatives Rejected:** Internal networks (requires all services in one compose file); host networking (no network isolation)
**Security Implications:** Any container on this network can reach Traefik and other services — do not add untrusted containers
**Impact:** All services must include `traefik-network` as an external network

---

**Decision:** Separate static and dynamic Traefik configuration into two files
**Date:** Pre-takeover (initial commit), updated 2026-05-14
**Context:** Traefik supports both static (entrypoints, providers) and dynamic (routers, services, middlewares) configuration
**Rationale:** Dynamic config can be reloaded without restarting Traefik (`watch: true`); static config changes are infrequent and require restart
**Alternatives Rejected:** Single config file (requires restart for all changes); CLI-only configuration (harder to version control)
**Security Implications:** Dynamic config changes take effect immediately — erroneous changes apply with no manual gate
**Impact:** Routine changes go in `dynamic.yaml`; infrastructure changes go in `traefik.yaml`

---

**Decision:** Use Docker labels for per-service Traefik configuration (drop-in compose pattern)
**Date:** Pre-takeover (initial commit), updated 2026-05-14
**Context:** Traefik can discover services via Docker labels or via file-based configuration
**Rationale:** Labels keep service-specific routing co-located with the service definition, making each app self-contained. Adding a new app = copy template + edit labels + deploy.
**Alternatives Rejected:** All routing in `dynamic.yaml` (centralized but harder to understand per-service); static backends (no auto-discovery)
**Security Implications:** `exposedByDefault: false` ensures only explicitly labeled services are routed
**Impact:** New apps define routing via labels in their own compose file; complex multi-service patterns go in `dynamic.yaml`

---

**Decision:** Use Let's Encrypt with HTTP-01 challenge for TLS certificates
**Date:** Pre-takeover (initial commit)
**Context:** Automated TLS certificate provisioning needed for HTTPS
**Rationale:** HTTP-01 is the simplest challenge type, works for most single-domain setups, requires only port 80 access
**Alternatives Rejected:** DNS-01 challenge (requires DNS provider API access, more complex but supports wildcards); TLS-ALPN-01 (less commonly supported); manual certificates (no automation)
**Security Implications:** Certificates stored in `acme.json` — must be protected from exposure; HTTP-01 requires port 80 to be publicly accessible
**Impact:** Does not support wildcard certificates; for wildcards, DNS-01 with Cloudflare API token would need to be added

---

**Decision:** Secrets in `.env` file, not hardcoded in YAML
**Date:** 2026-05-14
**Context:** Production setup needs to separate secrets from config
**Rationale:** `.env` is gitignored, keeps real values out of the repo. Traefik reads ACME email via env var naming convention (`TRAEFIK_CERTIFICATESRESOLVERS_LETSENCRYPT_ACME_EMAIL`)
**Alternatives Rejected:** Hardcoded in YAML (can't commit safely); Docker secrets (overkill for single-node setup)
**Impact:** `.env.example` documents required vars; compose reads and passes to Traefik

---

**Decision:** Dashboard secured with basic auth on a subdomain
**Date:** 2026-05-14
**Context:** Dashboard needs to be accessible from browser but protected
**Rationale:** Basic auth via `htpasswd` hash in `dynamic.yaml` is simple, standard, and sufficient for personal use. No insecure port exposed.
**Alternatives Rejected:** Insecure mode on port 8080 (no auth); forwardAuth via Authelia (overkill for personal VM); SSH tunnel only (inconvenient)
**Impact:** User generates hash during setup and puts it in `dynamic.yaml`; dashboard at `traefik.yourdomain.com`
