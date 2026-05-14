# How to Resume Work

## Resume Protocol

Follow these steps at the start of every new session:

### Step 1: Orient Yourself
Read `.claude/README.md` — understand the project, stack, and structure.

### Step 2: Know Current State
Read `.claude/state/CURRENT_STATUS.md` — know exactly what is done, in progress, and blocked.

### Step 3: Identify Next Task
Read `.claude/state/TASK_QUEUE.md` — identify the next task and confirm its dependencies are met.

### Step 4: Internalize Rules
Read `.claude/AGENT_RULES.md` — re-internalize all behavioral rules before touching anything.

### Step 5: Internalize Conventions
Read `.claude/CODING_STANDARDS.md` — re-internalize all YAML and Docker Compose conventions before editing.

### Step 6: Internalize Security Requirements
Read `.claude/SECURITY_STANDARDS.md` — re-internalize all security requirements before editing config.

### Step 7: Identify Active Environment
Determine if working in development or production. Consult `ENVIRONMENT_GUIDE.md` if unsure.

### Step 8: Read Task-Relevant Docs
Read the specific `compose.yaml`, `traefik/traefik.yaml`, or `traefik/dynamic.yaml` files relevant to the current task.

### Step 9: Verify Environment is Functional
```bash
# Check Traefik is running
docker ps | grep traefik

# Check Traefik logs
docker logs traefik --tail 20
```

If Traefik is not running and the task requires it:
```bash
docker network create traefik-network 2>/dev/null; docker compose up -d
```

### Step 10: Begin the Task
Implement → validate YAML syntax → verify Traefik accepted changes → update `.claude/` state files.

## Session End Checklist

Before closing any session:

- [ ] `state/CURRENT_STATUS.md` updated with session summary
- [ ] `state/TASK_QUEUE.md` updated — completed tasks marked DONE
- [ ] `state/DECISIONS_LOG.md` updated if any decision was made
- [ ] `CODING_STANDARDS.md` updated if new patterns were established
- [ ] `SECURITY_STANDARDS.md` updated if new findings were identified
- [ ] `CLAUDE.md` updated if project-level context changed
