# Runbook — SupportEngineerChallenge

## Service overview
- **Service:** SupportEngineerChallenge.Api
- **Purpose:** Minimal task tracker (create + list tasks)
- **Data store:** SQLite (`app.db` in the API working directory)

## Common commands

### Run locally
```bash
cd src/SupportEngineerChallenge.Api
dotnet run
```

### Run tests
```bash
dotnet test
```

## Key endpoints
- `GET /api/tasks?userId={id}&limit={n}`
- `POST /api/tasks`

## Using log artifacts

### Create-task 500
Inspect `artifacts/sample_api_log.txt` or production logs.

Look for:
- `CreateTask request`
- `X-Client-Timestamp present=False`
- `length=0`
- `System.FormatException`
- `DateTime.Parse`
- `TaskEndpoints.cs`

### Slow task list
Inspect `ListTasks completed` logs and database command logs.

Look for:
- high `elapsedMs`
- requested `userId`
- requested `limit`
- SQL that scans all tasks instead of applying `WHERE`, `ORDER BY`, and `LIMIT`

Expected fixed query shape:

```sql
WHERE "t"."UserId" = @__userId_0
ORDER BY "t"."CreatedAt" DESC, "t"."Id" DESC
LIMIT @__p_1
```

## Troubleshooting checklist

### “Create task fails with 500”
- Check API logs for `CreateTask request`.
- Confirm request body includes `userId` and `title`.
- Check for unhandled `FormatException`.
- Verify the endpoint uses server-side `DateTime.UtcNow` for task creation time.
- Confirm invalid payloads return `400`, not `500`.

### “Tasks list is slow”
- Check `ListTasks completed` logs for high `elapsedMs`.
- Review generated SQL.
- Confirm filtering, ordering, and limiting are applied in SQL.
- Confirm the endpoint does not load the full task table into memory.
- Validate large seeded datasets still return quickly.

### “Duplicates / wrong order after refresh”
- Refresh the UI repeatedly for the same user.
- Confirm task IDs are not repeated.
- Confirm newest tasks appear first.
- Confirm switching users clears/replaces the current task list.
- Compare browser output with `GET /api/tasks?userId={id}&limit={n}` response.

## Verification steps

### Automated
Run:

```bash
dotnet test
```

Tests should cover:
- valid task creation
- validation failure for missing required fields
- list endpoint only returns requested user
- list endpoint respects limit
- list endpoint returns newest tasks first

### Manual
- Start the API locally.
- Open the UI.
- Create 10+ tasks.
- Confirm no intermittent 500s.
- Confirm newly created tasks appear at the top.
- Refresh 10+ times.
- Confirm no duplicate rows appear.
- Switch between users.
- Confirm only the selected user's tasks are shown.
- Confirm logs show SQL-level filtering, ordering, and limiting.

## Rollback / mitigation

If the fix regresses:
- Roll back to the previous known-good deployment.
- Temporarily disable the UI path that sends malformed task creation requests.
- Monitor `POST /api/tasks` 500 rates.
- Monitor `GET /api/tasks` latency.
- If list latency spikes, reduce default/max `limit` temporarily.