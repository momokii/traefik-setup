# Agent Behavioral Rules

## Session Start — Mandatory Before Any Action

1. Read `.claude/HOW_TO_RESUME.md` completely
2. Read `.claude/state/CURRENT_STATUS.md` — understand exact current state
3. Read `.claude/state/TASK_QUEUE.md` — identify the next task
4. Read `.claude/CODING_STANDARDS.md` — internalize conventions before editing YAML
5. Read `.claude/SECURITY_STANDARDS.md` — internalize security requirements
6. Identify the active environment — consult `ENVIRONMENT_GUIDE.md`
7. Verify Traefik is healthy before modifying configuration:
   ```bash
   docker ps | grep traefik
   docker logs traefik --tail 10
   ```

## During Implementation

- Never make changes outside the stated scope of the current task
- Never delete, rename, or overwrite existing YAML files without explicit user instruction
- Never introduce a new Docker image or plugin without surfacing the proposal to the user and receiving explicit confirmation
- Always follow the patterns and conventions in `CODING_STANDARDS.md` — do not introduce new patterns without logging them in `DECISIONS_LOG.md`
- When modifying `dynamic.yaml`, be aware that Traefik auto-reloads this file on change — test changes carefully since they take effect immediately
- Never commit files with real domain names, email addresses, or IP addresses — always use the placeholder pattern

## Configuration Validation Rules

- After modifying any Traefik configuration, validate the YAML syntax
- After modifying `dynamic.yaml`, verify Traefik picked up the change via logs
- After modifying `compose.yaml`, validate with `docker compose config`
- Never leave a configuration in a broken state — if a change fails, revert immediately

## Security Rules — Non-Negotiable

- Never write configuration that stores or exposes real secrets, tokens, or credentials
- Never commit `.env` files, `acme.json`, or any file containing real certificates
- If a security vulnerability is discovered in existing configuration during any session, flag it to the user immediately before proceeding
- All placeholder values must remain as placeholders in committed files
- Consult `SECURITY_STANDARDS.md` before implementing any feature involving TLS, auth, or external access

## Environment Awareness Rules

- Always identify the active environment before running any command
- In production: present a written plan and receive explicit confirmation before executing any change
- Never expose the Traefik dashboard without authentication
- Verify `.gitignore` exists and covers sensitive files before the first commit of any session

## Session End — Mandatory Before Closing

- Update `state/CURRENT_STATUS.md` with accurate current state and a session summary
- Update `state/TASK_QUEUE.md` — mark completed tasks, add newly discovered tasks
- Log any significant decision in `state/DECISIONS_LOG.md`
- Update `CODING_STANDARDS.md` if new patterns were established
- Update `SECURITY_STANDARDS.md` if new security findings were identified
- Update `CLAUDE.md` if project-level context changed materially

## Self-Maintenance Directive

- The `.claude/` files must stay accurate at all times
- If a convention in `CODING_STANDARDS.md` is wrong or outdated, correct it immediately and log the change in `DECISIONS_LOG.md`
- If the project state in `CURRENT_STATUS.md` is stale, update it before proceeding

## Escalation Rule

When blocked, uncertain about scope, or facing a decision with significant architectural or security impact: document the blocker in `CURRENT_STATUS.md` and ask the user — do not assume and proceed.
