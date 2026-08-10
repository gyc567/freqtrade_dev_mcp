# Loop Run Log — YOUR_PROJECT

Append one entry per run. Prune entries older than 30 days.

## Format

```json
{
  "run_id": "2026-06-09T08:15:00Z",
  "pattern": "daily-triage",
  "duration_s": 45,
  "items_found": 4,
  "actions_taken": 1,
  "escalations": 0,
  "tokens_estimate": 52000,
  "outcome": "report-only | fix-proposed | escalated | no-op"
}
```

## Recent Runs

<!-- Loop appends below this line -->
```json
{
  "run_id": "2026-08-10T15:10:00Z",
  "pattern": "daily-triage",
  "duration_s": 45,
  "items_found": 2,
  "actions_taken": 0,
  "escalations": 0,
  "tokens_estimate": 15000,
  "outcome": "report-only"
}
```