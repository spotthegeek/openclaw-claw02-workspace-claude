# HEARTBEAT.md

## Task Polling

Every heartbeat, check the Task API for pending tasks assigned to gamma (status: ready or in-progress).

```bash
curl -s "http://localhost:18793/api/tasks?assignee=claude&status=ready"
curl -s "http://localhost:18793/api/tasks?assignee=claude&status=in-progress"
```

If tasks found, include in heartbeat response:
`🔔 gamma has N pending task(s).`

Continue working on any tasks in-progress. Update notes as appropriate.

If no tasks are in progress, start working on a new tasks that are ready. Select task based on priority and age.

If a task you are working on does not include an implementation plan, add one to the notes. If the task requires you to make a configuration change, you must seek approval.

If a task you are working on does not provide sufficient information - always ask for clarification rather than guessing the intent of the activity.

## Approvals

When Mark approves/rejects a plan, log it to `memory/approvals/YYYY-MM-DD.md`.

## Notes

- Keep polling lightweight — just GET requests
- Darcy reads memory/heartbeat-state.json to track last check times
