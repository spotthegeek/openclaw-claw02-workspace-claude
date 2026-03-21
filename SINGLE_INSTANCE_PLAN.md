# Single-Instance Mission Control — Refactor Plan

> Converting from multi-instance (remote OpenClaw management) to single-instance (OpenClaw + MC on same host, MC as Docker containers).

## Why

**Current:** MC connects to multiple remote OpenClaw instances via WebSocket/HTTP, each with its own host, port, and auth token. The `mc_instances` registry is the core data model.

**New:** OpenClaw runs bare-metal on the host. MC runs as Docker containers on the same host. MC accesses OpenClaw's filesystem directly via a mounted volume — no networking between containers, no auth tokens, no WebSocket connections to remote hosts.

---

## Key Changes Summary

| Area | Before | After |
|---|---|---|
| **Architecture** | MC → remote OpenClaw instances via WS/HTTP | MC containers → shared filesystem volume |
| **Instance registry** | `mc_instances` table, instance switcher UI | Removed — single local instance only |
| **Auth** | Fernet-encrypted auth tokens per remote instance | No token vault needed |
| **Connection** | `ConnectionManager` with per-instance WS pools | Direct filesystem reads + local CLI |
| **Proxy** | `/instances/{id}/proxy/{path}` routed to remote | Removed — filesystem access instead |
| **Docker compose** | Isolated MC services, no OC files | OpenClaw workspace mounted into MC backend |
| **Frontend** | Instance switcher, per-instance status badges | Single-instance UI, simplified sidebar |

---

## Phase 1 — Architecture Decisions & Shared Filesystem Layout

### 1.1 Define the Shared Mount

OpenClaw's workspace is at `/opt/openclaw/.openclaw` (bare-metal install). This directory contains:

```
/opt/openclaw/.openclaw/
├── workspace-claude/     # agent workspaces
├── memory/               # shared memory files
├── config/               # openclaw config
├── tasks.db              # sqlite task store (if used)
└── ...other openclaw files
```

**Decision needed:** Does MC need read/write access to all of this, or a specific subset?

**Proposed mount:** `/opt/openclaw/.openclaw` → `/app/openclaw` inside the backend container.

### 1.2 OpenClaw Integration Strategy

Without the WebSocket connection, MC needs alternative access to:

| Data | Current Source | New Source |
|---|---|---|
| Agents list | `agents.list` WS call | `openclaw agents list` CLI command |
| Agent status | WS push events | `openclaw agents list` (polled or CLI events) |
| Approvals | `approvals.list` WS call | `openclaw approvals list` CLI command |
| Approve/Reject | `approvals.approve/reject` WS call | `openclaw approve/reject` CLI command |
| Tasks | `cron.list` + `tasks.*` WS calls | Direct filesystem (`tasks.db`) or CLI |
| Memory files | `memory.*` REST calls proxied | Direct filesystem read at `/app/openclaw/memory/` |
| Docs files | `docs.*` REST calls proxied | Direct filesystem read |

**Decision needed:** Should MC use CLI commands (via `exec`) or direct filesystem access for each data type?

### 1.3 Remove Instance Switcher (Frontend)

The sidebar's instance switcher dropdown becomes a static label: "Local OpenClaw". All per-instance routing (`/instances/{id}/...`) is removed.

---

## Phase 2 — Backend Simplification

### 2.1 Remove or Stub These Modules

| Module | Action | Reason |
|---|---|---|
| `app/routers/instances.py` | **Remove** | No more `mc_instances` table |
| `app/routers/proxy.py` | **Remove** | No proxying to remote instances |
| `app/services/token_vault.py` | **Remove** | No encrypted tokens needed |
| `app/services/openclaw_client.py` | **Rewrite** | Simplified to CLI invocations |
| `app/services/connection_manager.py` | **Remove** | No WS pool management |
| `app/models/instance.py` | **Remove** | `Instance` model gone |
| `app/models/agent_cache.py` | **Simplify** | Remove `instance_id` foreign key |
| `app/models/cron_cache.py` | **Simplify** | Remove `instance_id` foreign key |
| `app/routers/agents.py` | **Rewrite** | Use local CLI instead of WS |
| `app/routers/approvals.py` | **Rewrite** | Use local CLI instead of WS |
| `app/routers/tasks.py` | **Rewrite** | Use filesystem/CLI instead of WS |
| `app/routers/memory.py` | **Rewrite** | Direct filesystem instead of proxy |
| `app/routers/docs.py` | **Rewrite** | Direct filesystem instead of proxy |
| `app/routers/stream.py` | **Rewrite** | SSE from local source (CLI events or filesystem watch) |

### 2.2 New Local OpenClaw Client

Replace `connection_manager.py` with a simple `LocalOpenClawClient`:

```python
class LocalOpenClawClient:
    """Invokes openclaw CLI for commands, reads workspace files directly."""

    def __init__(self, workspace_path: str = "/app/openclaw"):
        self.workspace = workspace_path

    async def list_agents(self) -> list[dict]: ...
    async def get_approvals(self) -> list[dict]: ...
    async def approve(self, approval_id: str) -> None: ...
    async def reject(self, approval_id: str) -> None: ...
    async def spawn_agent(self, name: str, model: str, prompt: str) -> str: ...
    async def terminate_agent(self, agent_id: str) -> None: ...
    def read_memory_file(self, path: str) -> str: ...
    def write_memory_file(self, path: str, content: str) -> None: ...
```

**Implementation options (decide per method):**
- Use `asyncio.create_subprocess_exec("openclaw", ...)` for CLI commands
- Use `aiofiles` for filesystem reads/writes
- Poll filesystem or use `inotify` for change notifications

### 2.3 Remove `mc_instances` Table

Delete `Instance` model and all migrations. Remove instance FK from `AgentCache`, `CronCache`, `EventLog`.

### 2.4 Simplify Database Schema

- `mc_agent_cache.instance_id` → remove column, agents are local only
- `mc_cron_cache.instance_id` → remove column
- `mc_event_log.instance_id` → remove column (single event log for local OC)
- `mc_instances` → drop table

### 2.5 Simplify Auth

Since there's only one local OpenClaw (no remote tokens), the MC dashboard password (`MC_ADMIN_PASSWORD`) remains for MC's own login. The token vault is not needed.

If OpenClaw itself requires auth to CLI commands, MC may need the OpenClaw auth token — but this would be a single fixed token stored in `.env`, not in the database.

---

## Phase 3 — Docker Compose Changes

### 3.1 Mount OpenClaw Workspace

```yaml
services:
  backend:
    volumes:
      - /opt/openclaw/.openclaw:/app/openclaw:ro  # read-only or rw depending on needs
      - mc_data:/app/data
    # No BACKEND_URL change needed — still runs as two containers
```

**Decision:** Read-only or read-write? MC likely needs to write memory/docs files.

### 3.2 Remove Instance-Facing Networking

The current `docker-compose.yml` exposes ports for OpenClaw instances. Since OC runs on the host, no ports need to be exposed for that purpose.

### 3.3 Optional: Host Networking

Consider putting the backend on `network_mode: host` if it needs to invoke `openclaw` CLI commands directly on the host (though mounting the workspace should suffice).

### 3.4 Remove `nginx` Profile (if used)

Unless MC needs reverse proxy for TLS termination, nginx may be unnecessary for single-instance local use. Or keep it for a clean `:80` → `:3000` mapping.

---

## Phase 4 — Frontend Simplification

### 4.1 Remove Instance Switcher

- `src/components/layout/sidebar.tsx` — remove instance dropdown, static "Local OpenClaw" label
- `src/components/instances/instance-switcher.tsx` — remove component
- `src/components/instances/add-instance-dialog.tsx` — remove
- `src/context/instance-context.tsx` — simplify to null/disabled (no instance to track)
- `src/app/(dashboard)/settings/instances/page.tsx` — remove page entirely
- `src/app/api/proxy/[...path]/route.ts` — remove (no proxy needed)

### 4.2 Simplify Top Nav

- Remove instance status badge (single local = always connected or not)
- Keep: user info, refresh button, logout

### 4.3 Update All API Calls

Current pattern: `GET /api/proxy/instances/{id}/agents/list`
New pattern: `GET /api/agents` (backend calls local CLI/filesystem)

All `instanceId` parameters disappear from API requests.

### 4.4 Remove Instance-Related Types

```typescript
// Remove:
Instance, InstanceStatus, InstanceSwitcherProps, ...
// Keep:
Agent, Approval, Task, Project, CronJob, MemoryFile, DocFile, ...
```

---

## Phase 5 — Update Documentation

### 5.1 Files to Update

| File | Changes |
|---|---|
| `PRD.md` | Rewrite overview — single-instance focus |
| `PLAN.md` | Mark Phases 1-8 as arch-relevant only; new single-instance milestones |
| `CLAUDE.md` | Update architecture diagram, remove instance routing docs |
| `docker-compose.yml` | Add volume mount, update ports comment |
| `.env.example` | Remove per-instance token fields, add `OPENCLAW_WORKSPACE_PATH` |

### 5.2 Update `CLAUDE.md` Architecture Section

```markdown
## Architecture

```
browser → Next.js frontend (port 3000) → FastAPI backend (port 8000)
                                              ↓
                                      OpenClaw workspace (mounted volume)
                                              ↓
                                        openclaw CLI + filesystem
```

MC runs as Docker containers on the same host as OpenClaw. The backend accesses OpenClaw's workspace directly via a shared volume mount.
```

---

## Phase 6 — Migration & Testing

### 6.1 Data Migration

If upgrading an existing MC deployment with data:
- `mc_instances` table → DROP (no equivalent in single-instance)
- `mc_agent_cache` → keep but drop `instance_id` column
- `mc_cron_cache` → keep but drop `instance_id` column
- `mc_event_log` → keep but drop `instance_id` column

### 6.2 Test the Refactor

1. `docker compose up --build` — both services start
2. Login → dashboard loads (no instance switcher)
3. Agents page → lists agents from local `openclaw agents list`
4. Approvals → approve/reject calls local `openclaw approve/reject`
5. Memory browser → reads/writes `/app/openclaw/memory/`
6. All existing e2e tests — adapt selectors for removed UI elements

---

## Decision Checklist (Before Starting)

- [ ] What is the exact path to OpenClaw's workspace on the host?
- [ ] Will MC need read-only or read-write access to the workspace?
- [ ] Will OpenClaw CLI commands be invoked via `exec` inside the container, or via filesystem access?
- [ ] Should the existing `tasks.db` be used directly, or should tasks continue to go through OpenClaw's own API?
- [ ] Should MC keep any WebSocket connection to OpenClaw (e.g., for real-time agent events), or is polling sufficient?
- [ ] What to do with existing MC data? (Fresh install or migrate?)
- [ ] Keep or remove nginx from docker-compose?
- [ ] Should MC still require a login password, or is host-level security sufficient?

---

## Estimated Scope

| Phase | Effort |
|---|---|
| Phase 1 — Decisions + Plan | Low |
| Phase 2 — Backend simplification | **High** |
| Phase 3 — Docker compose | Low |
| Phase 4 — Frontend simplification | **Medium** |
| Phase 5 — Docs update | Low |
| Phase 6 — Migration + testing | **Medium** |
