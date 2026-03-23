# HEARTBEAT.md

## Task Polling

Every heartbeat, check the Task API for pending tasks assigned to claude (status: inbox or active).

```bash
curl -s "http://localhost:18793/api/tasks?assignee=claude&status=inbox"
curl -s "http://localhost:18793/api/tasks?assignee=claude&status=active"
```

If tasks found, include in heartbeat response:
`🔔 claude has N pending task(s).`

## Approvals

When Mark approves/rejects a plan, log it to `memory/approvals/YYYY-MM-DD.md`.

## Notes

- Keep polling lightweight — just GET requests
- Darcy reads memory/heartbeat-state.json to track last check times
