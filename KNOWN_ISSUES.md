# Known Issues

This file records issues identified during the code review of the changes merged from PRs 393-397. The upstream report was rechecked against the local branch on 2026-10-02. Local verification and fixes are recorded below; these statuses do not imply that upstream has merged the fixes.

## 1. Task and log API routes are not authenticated

- **Severity:** High
- **Location:** `render-backend/server.js:294`
- **Status:** Fixed locally (not pushed)
- **Impact:** `/api/tasks`, `/api/logs`, `/api/logs/db`, and `/api/task-definitions` are reachable without the `requireApiKey` middleware. In particular, an unauthenticated caller can create, modify, delete, or manually run scheduled tasks. Manual execution can use the game tokens stored in Supabase.
- **Planned direction:** Protect all `/api/*` routes with the existing API-key middleware, leaving only `/health` intentionally public.

- **Local verification (2026-10-02):** Existing API-key middleware is mounted before every `/api/*` route. Authentication tests cover missing, invalid and valid keys plus route ordering. `/health` remains public. This was already fixed during the preceding public-code review.

## 2. Batch Apex guessing processes only the first open round

- **Severity:** High
- **Location:** `src/utils/batch/tasksApex.js:57`
- **Status:** Conditional defect fixed locally (not pushed)
- **Impact:** `resolveOpenGuesses()` returns as soon as it finds one round with open stages. When multiple rounds overlap, an earlier or later round with open guesses can be skipped silently.
- **Planned direction:** Return every round with open stages and process each round in the batch task.

- **Local verification (2026-10-02):** The implementation did return the first open round. A mocked overlapping schedule reproduced a skipped second round. Scanning 1,106 boundary instants derived from the current local schedule snapshot did not find two simultaneously guessable rounds; therefore a live missed guess is not established for this snapshot. The resolver now collects all open rounds, and the task processes each with its own stage limits.

## 3. Apex pagination can mark a rate-limited stage as exhausted

- **Severity:** High
- **Location:** `src/components/Apex/ApexChallenge.vue:1269`
- **Status:** Fixed locally (not pushed)
- **Impact:** A 200400 rate-limit response produces no new rows. `ensureBetRows()` treats that as a definitive end-of-data condition and sets `grp.exhausted = true`, which can prevent later retries for that stage.
- **Planned direction:** Distinguish a rate-limit interruption from a confirmed empty page and only mark a stage exhausted for the latter.

- **Local verification (2026-10-02):** A mocked 200400 response reproduced `exhausted = true` without a successful empty-page response. An already in-flight request triggered the same false exhaustion. Pagination now returns explicit completeness/interruption flags; no-growth stops only the current loop, while a confirmed empty page controls exhaustion. Tests verify recovery and continuation from the loaded row count.

## 4. Apex vote board can be replaced by a partial rate-limited result

- **Severity:** Medium
- **Location:** `src/components/Apex/ApexChallenge.vue:1534`
- **Status:** Fixed locally (not pushed)
- **Impact:** `fetchPagedList()` can return partial rows after a rate-limit interruption, but `fetchVoteBoard()` assigns those rows to `currentVoteBoard` without checking whether the result is complete. The visible board can therefore shrink to an incomplete list during polling.
- **Planned direction:** Return an explicit interruption/completeness flag and replace the visible board only after a complete fetch; otherwise retain the last complete result.

- **Local verification (2026-10-02):** A successful first page followed by a mocked 200400 response replaced the prior complete vote board with partial rows. The board now changes only for complete results, including a successful empty result. Tests cover rate limits, page-budget interruption and network failure.

## Verification context

- `pnpm build` completed successfully on 2026-09-21.
- The repository currently has no GitHub Actions workflow configured, so no workflow run can be started until one is added under `.github/workflows/`.

## Local validation (2026-10-02)

- `pnpm run lint`: zero errors and warnings.
- `pnpm run format:check`: passed.
- `pnpm test`: 71 tests passed, including 9 new Apex regression tests. The first six targeted tests were run before production edits: five failed, reproducing the defects and associated boundary cases; all now pass.
- `pnpm run build`: passed; existing optional-plugin and chunk-size notices remain.
- No live backend job, game login, guess or vote was performed. Tests execute the actual source functions with isolated request dependencies.
- Local private content and Git isolation are preserved; no GitHub push was performed.
