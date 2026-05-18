# Incident Summary

**Title:** Task creation 500s, slow task list responses, and duplicate task rendering  
**Date:** 2026-05-18  
**Severity:** P2  

## Impact
Users intermittently saw task creation fail with a 500. Some users also experienced slower task list refreshes, and the UI could show duplicate or incorrectly ordered tasks after repeated refreshes.

## Detection
The issues were identified from customer reports and confirmed through local reproduction, application logs, SQL logs, and provided artifact logs.

The production-style artifact `artifacts/sample_api_log.txt` showed `X-Client-Timestamp present=False length=0`, followed by an unhandled `System.FormatException` from `DateTime.Parse`.

## Timeline (UTC)
- 18:00 — Reviewed challenge scenario and started local setup.
- 18:15 — Ran API locally and reproduced task list behavior.
- 18:25 — Observed slow list logs showing full-table task query.
- 18:35 — Reproduced duplicate task rows after repeated refreshes.
- 18:45 — Reviewed production log artifact and identified missing timestamp header failure.
- 18:55 — Reproduced create-task 500 locally after repeated task creation.
- 19:10 — Implemented backend query fix and UI refresh fix.
- 19:25 — Implemented create-task timestamp handling fix.
- 19:40 — Added regression tests.
- 19:50 — Verified automated and manual tests passed.

## Root cause
There were three separate issues:

1. `POST /api/tasks` parsed `X-Client-Timestamp` before safely validating it. Missing or empty values caused `DateTime.Parse` to throw `FormatException`, returning a 500.

2. `GET /api/tasks` loaded all tasks into memory before filtering, sorting, and limiting. This caused list latency to scale with total table size instead of the requested user's result set.

3. The UI appended refreshed task results to existing state instead of replacing state. This caused duplicate rendered rows after repeated refreshes.

## Mitigation / resolution
- Updated task creation to use server-side `DateTime.UtcNow` instead of trusting/parsing a client timestamp.
- Kept request validation so missing `userId` or `title` returns `400`.
- Updated list endpoint to apply `WHERE`, `ORDER BY`, and `LIMIT` in the database query.
- Added deterministic secondary ordering by task ID.
- Updated UI refresh logic to replace task state instead of appending.
- Removed intentionally malformed client timestamp behavior from the UI.
- Added regression tests for validation, user filtering and, limits.

## Verification
Verified through:
- `dotnet test`
- manual task creation
- repeated refreshes
- switching users
- checking no duplicate rows appeared
- checking new tasks appeared at the top
- checking SQL logs showed `WHERE`, `ORDER BY`, and `LIMIT`
- confirming no 500s occurred after repeated task creation

## Follow-ups / action items
- [ ] Add alerting for elevated `POST /api/tasks` 5xx rate.
- [ ] Add alerting for high `GET /api/tasks` p95 latency.
- [ ] Add browser/UI regression test coverage for refresh behavior.
- [ ] Consider adding database indexes for `UserId`, `CreatedAt`, and `Id` if dataset size grows.