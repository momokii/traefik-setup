# Decisions Log

---

**Decision:** Use external Docker network `traefik-networks` for all service communication
**Date:** Pre-takeover (initial commit)
**Context:** Traefik and backend services need shared network for routing
**Rationale:** External network allows independent compose files to connect to the same Traefik instance without being in the same compose project
**Alternatives Rejected:** Internal networks (requires all services in one compose file); host networking (no network isolation)
**Security Implications:** Any container on this network can reach Traefik and other services — do not add untrusted containers
**Impact:** All future example directories must include `traefik-networks` as an external network

---

**Decision:** Separate static and dynamic Traefik configuration into two files
**Date:** Pre-takeover (initial commit)
**Context:** Traefik supports both static (entrypoints, providers) and dynamic (routers, services, middlewares) configuration
**Rationale:** Dynamic config can be reloaded without restarting Traefik (`watch: true`); static config changes are infrequent and require restart
**Alternatives Rejected:** Single config file (requires restart for all changes); CLI-only configuration (harder to version control)
**Security Implications:** Dynamic config changes take effect immediately — erroneous changes apply with no manual gate
**Impact:** Routine routing/service changes go in `dynamic-config.yaml`; infrastructure changes go in `traefik-config.yaml`

---

**Decision:** Use Docker labels for per-service Traefik configuration
**Date:** Pre-takeover (initial commit)
**Context:** Traefik can discover services via Docker labels or via file-based configuration
**Rationale:** Labels keep service-specific routing config co-located with the service definition in `compose.yaml`, making each example self-contained
**Alternatives Rejected:** All routing in `dynamic-config.yaml` (centralized but harder to understand per-service); static backends (no auto-discovery)
**Security Implications:** `exposedByDefault: false` ensures only explicitly labeled services are routed
**Impact:** New examples should define routing via labels; complex multi-service patterns (like weighted routing) go in `dynamic-config.yaml`

---

**Decision:** Use Let's Encrypt with HTTP-01 challenge for TLS certificates
**Date:** Pre-takeover (initial commit)
**Context:** Automated TLS certificate provisioning needed for HTTPS
**Rationale:** HTTP-01 is the simplest challenge type, works for most single-domain setups, requires only port 80 access
**Alternatives Rejected:** DNS-01 challenge (requires DNS provider API access, more complex but supports wildcards); TLS-ALPN-01 (less commonly supported); manual certificates (no automation)
**Security Implications:** Certificates stored in `acme.json` — must be protected from exposure; HTTP-01 requires port 80 to be publicly accessible
**Impact:** This setup does not support wildcard certificates; for wildcards, DNS-01 challenge would need to be added

---

**Decision:** Use placeholder values for all sensitive configuration
**Date:** Pre-takeover (initial commit)
**Context:** Repository is public and serves as a reference — must not contain real credentials
**Rationale:** Placeholders (`YOUR-DOMAIN`, `YOUR_EMAIL_HERE`, `YOUR_IP_ADDRESS`) allow users to understand what needs configuring without exposing real values
**Alternatives Rejected:** Environment variables (adds complexity for a config-only reference repo); real values (security risk)
**Security Implications:** Placeholders must never be replaced with real values in committed files
**Impact:** All new configuration must follow the same placeholder pattern

---

**Decision:** Enable Sablier plugin for zero-scale resource optimization
**Date:** Pre-takeover (initial commit)
**Context:** Some services are used infrequently and should not consume resources when idle
**Rationale:** Sablier automatically stops containers after inactivity and provides a loading page while restarting — saves resources in homelab/dev environments
**Alternatives Rejected:** Manual start/stop (user overhead); always-on containers (wastes resources)
**Security Implications:** Sablier requires Docker socket access; stopped containers are unavailable until woken up
**Impact:** Services using Sablier must be referenced by container name in the dynamic config, and must connect to `traefik-networks`
