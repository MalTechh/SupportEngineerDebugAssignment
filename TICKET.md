# Follow-up Ticket

**Title:** Add observability and regression coverage for task creation and task list workflows  
**Priority:** P2  
**Owner:** SupportEngineerChallenge maintainers  

## Description
After resolving the immediate task creation failures, slow task list responses, and duplicate UI rendering, add stronger observability and regression coverage around the task workflows.

The incident showed that a malformed client request could produce a 500, list latency could degrade due to inefficient query shape, and UI refresh behavior could render duplicate rows. These should be easier to detect before customers report them.

## Acceptance criteria
- [ ] Add structured logging for `POST /api/tasks` validation failures and successful creates.
- [ ] Add structured logging for `GET /api/tasks` including `userId`, requested `limit`, returned count, and elapsed time.
- [ ] Add alert definition or documented threshold for elevated `POST /api/tasks` 5xx rate.
- [ ] Add alert definition or documented threshold for high `GET /api/tasks` p95 latency.
- [ ] Add regression coverage for task list ordering and limit behavior.
- [ ] Add UI-level test or documented manual test for repeated refreshes not duplicating rows.

## Notes / context
Relevant areas:
- `src/SupportEngineerChallenge.Api/Endpoints/TaskEndpoints.cs`
- `src/SupportEngineerChallenge.Api/wwwroot/main.js`
- `tests/SupportEngineerChallenge.Tests/TaskApiTests.cs`
- `artifacts/sample_api_log.txt`
- `artifacts/sample_slow_list_log.txt`

Suggested future improvement:
If the task table grows beyond the seeded/local dataset size, add an index supporting the list query