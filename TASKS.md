# TASKS.md — Task Management

All agents manage tasks via the **Task API**, backed by SQLite.

**Base URL:** `http://localhost:18793` (same host) · `http://10.0.0.167:18793` (from other nodes)

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/tasks` | List tasks (supports filters) |
| GET | `/api/tasks/:id` | Get single task |
| POST | `/api/tasks` | Create task |
| PATCH | `/api/tasks/:id` | Update task (partial) |
| DELETE | `/api/tasks/:id` | Delete task |
| GET | `/api/health` | Health check |

**Query filters:** `?status=` · `?project=` · `?assignee=` · `?priority=` · `?instance_id=`

---

## Task Object

```json
{
  "title":    "string (required)",
  "description": "",
  "status":   "inbox | active | review | done",
  "priority": "A | B | C",
  "assignee": "",
  "project":  "personal-assistant | researcher | project-manager | homelab-engineer",
  "instance_id": "",
  "due_date": "YYYY-MM-DD",
  "tags":     "[\"label1\"]"
}
```

**Status flow:** `inbox → active → review → done`
**Priority:** `A` (high) · `B` (medium) · `C` (low)

---

## Workflow

- **Inbox** — new/unprocessed tasks
- **Active** — tasks being worked on
- **Review** — awaiting Mark's approval
- **Done** — completed tasks

---

## Examples

```bash
# Create a task
curl -X POST http://localhost:18793/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Review PR #42","priority":"A","project":"homelab-engineer","assignee":"darcy"}'

# List all inbox tasks
curl "http://localhost:18793/api/tasks?status=inbox"

# Move to active
curl -X PATCH http://localhost:18793/api/tasks/5 \
  -H "Content-Type: application/json" \
  -d '{"status":"active"}'

# Mark done
curl -X PATCH http://localhost:18793/api/tasks/5 \
  -H "Content-Type: application/json" \
  -d '{"status":"done"}'

# Delete
curl -X DELETE http://localhost:18793/api/tasks/5

# Get single
curl http://localhost:18793/api/tasks/5
```

---

## Projects

Valid project values:
- `personal-assistant`
- `researcher`
- `project-manager`
- `homelab-engineer`

---

## Instance Isolation

Set `instance_id` to isolate tasks to a specific agent instance (leave blank for shared pool).
