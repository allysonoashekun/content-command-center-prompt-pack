# Phase 8: Test, Audit, and Operate

## What this phase delivers

A test suite over the parts that cannot be checked by looking at the screen, a full audit against `SPEC.md`, an operations document with a verified backup, and one worked extension carried through every layer.

The application is complete at the end of phase 7. This phase proves it.

---

### P8.1 - Complete the test suite

Requires: P7.7
Halts: no

OBJECTIVE

Tests covering the three platform modules, the shared metric helpers, and the prompt builders, running in seconds without touching a network or a database.

PROMPT

```text
--- PROMPT P8.1 ---
Read SPEC.md sections "Platform Sync" and "AI Insights".

1. Create a test helper that installs a mock global fetch, records every
   call it receives with its URL, method, headers, and body, and returns
   a queue of prepared responses in order. Tests must never call a real
   platform API: real calls are slow, consume quota, and fail for
   reasons unrelated to the code.

2. Write tests for each platform module covering:
   - validateCredentials on success and on a rejected credential.
   - Pagination across more than one page, asserting the cursor or page
     token was actually sent on the second request.
   - The guard against a cursor that does not advance, confirming it
     stops rather than looping.
   - Metric normalization, including the reach proxy on YouTube and
     TikTok, saves left at zero on TikTok, and impressions left at zero
     on Instagram.
   - Token refresh, and specifically that TikTok's rotated refresh token
     is returned to the caller.

3. Write tests for the shared platform helpers, covering the engagement
   rate formula and its behavior when reach is zero.

4. Write tests for the prompt builders, covering: each type produces a
   system block marked for caching, an invalid context is rejected by
   its schema, an unknown type throws, and extractTopInsights pulls the
   trailing insights section out of a content gap response.

5. Run the full suite and report the file count and test count.

Do not write tests that require a database connection or a network
connection.
--- END PROMPT P8.1 ---
```

EXPECTED RESULT

One test helper and several test files, joining the aggregation tests from `P6.1`. The suite runs in seconds.

VERIFICATION

1. `npm test` exits 0. Report the file and test counts and the wall-clock duration.
2. Grep the test files for any import of the database client or any un-mocked `fetch`. There must be none.
3. Mutate one thing deliberately: change `computeEngagementRate` to divide by views instead of reach. Run the suite and confirm tests fail. Restore the formula and confirm they pass again. A suite that passes both before and after a real change is not testing what it claims to.
4. Confirm a pagination test asserts the cursor or page token was sent on the second request, not merely that two requests happened.
5. Confirm a TikTok refresh test asserts the rotated refresh token is returned to the caller.

GATE: Check 3 shows failures on the mutation and passes after restoring.

---

### P8.2 - Audit the build against the specification

Requires: P8.1
Halts: no

OBJECTIVE

A written comparison of the finished application against every section of `SPEC.md`, with no code changed.

PROMPT

```text
--- PROMPT P8.2 ---
Audit the entire application against SPEC.md. Do not change any code
during this audit. Produce a report only.

Go through every section of SPEC.md in order. For each, report:

- Specified and present.
- Specified and missing.
- Specified but behaving differently, describing the difference.
- Present in the application but not described in the specification.

Then verify separately:
- Every mutating API route calls checkSameOrigin.
- Every database table and column in SPEC.md exists in the database with
  the specified type, and nothing extra is present.
- The full test suite passes.
- npm run build and npm run lint both complete without errors.
- .env.local and .env.credentials.local are still ignored by Git and
  have never been committed.

Present the result as a table, and list anything not fully present in
priority order, with a one-line description of the work each would take.
--- END PROMPT P8.2 ---
```

EXPECTED RESULT

A written report. No code changes.

VERIFICATION

1. `npm test && npm run build && npm run lint` all exit 0.
2. `git log --all --oneline -- .env.local .env.credentials.local` returns no output. Any output means a credential file entered the history and every credential it contained must be revoked and reissued.
3. Independently re-verify three items the report marks as present, by a different means than the report used. An audit is only as reliable as its checks.
4. Re-run the schema comparison from `P2.2` and confirm it still reports zero differences.
5. Confirm every mutating route calls `checkSameOrigin`, by grepping the route files rather than trusting the report.

GATE: Check 2 returns no output, and checks 1, 4, and 5 pass.

Differences the report finds are not automatically defects. Where the built behavior is preferred, update `SPEC.md` rather than the code, then re-run this task.

---

### P8.3 - Establish the operating routine

Requires: P8.2
Halts: no

OBJECTIVE

An operations document, a database backup script proven restorable, and a report of AI token spend to date.

PROMPT

```text
--- PROMPT P8.3 ---
1. Write OPERATIONS.md in the repository root covering:

   - How to start the application and what to do first each session.
   - Why syncing regularly matters: a snapshot is recorded per post per
     day, so a day without a sync is a permanent gap in the history. The
     charts can only show days that were captured.
   - Which credentials expire and when, and which refresh themselves.
   - How to back up and restore the database, with the exact commands
     for this machine.
   - How to check AI spending.
   - That .env.credentials.local is no longer needed once every platform
     is connected in Settings, and may be deleted. Note that deleting it
     does not revoke anything.

2. Create a backup script that dumps the database to a timestamped file
   in a directory ignored by Git, and confirm the .gitignore covers it.
   A database dump contains platform tokens and must never be committed.

3. Run the backup once and report the file size. Then verify the dump is
   restorable by restoring it into a scratch database and comparing row
   counts per table against the original, and drop the scratch database
   afterward.

4. Report total input and output tokens recorded in ai_suggestions,
   grouped by suggestion type.

Do not set up scheduled jobs, cron, or any background process.
--- END PROMPT P8.3 ---
```

EXPECTED RESULT

An operations document, a backup script, one backup file, and a token usage report.

VERIFICATION

1. `git check-ignore -v backups/` prints the matching rule.
2. The backup file exists and is non-zero in size.
3. The restore test compared row counts per table between the original and the scratch database, not merely the exit code of the restore command. A backup that has never been restored is an assumption.
4. The scratch database no longer exists: `psql -l` does not list it.
5. `OPERATIONS.md` covers all six required topics.
6. `psql content_command_center -c "select type, sum(input_tokens), sum(output_tokens) from ai_suggestions group by type;"` returns the reported figures.

GATE: Check 3 shows a row-count comparison.

---

### P8.4 - Carry one change through every layer

Requires: P8.3
Halts: no

OBJECTIVE

Total watch time displayed on the platform summary cards, carried from the database through the calculations, the query layer, the API, and the component.

The value is already collected and stored by the platform modules that report it. Nothing displays it. This task is the worked example of a change that touches every layer.

PROMPT

```text
--- PROMPT P8.4 ---
Read SPEC.md sections "Analytics Dashboard" and "Database Schema:
analytics_snapshots".

Add total watch time to the platform summary cards. The value is already
stored in analytics_snapshots.watch_time_seconds and is already collected
by the platform modules that report it.

1. In aggregate.ts, extend summarizePlatforms to total watch time across
   the latest snapshot per post, alongside the existing totals. Treat a
   missing value as absent rather than as zero.

2. Add tests covering a platform that reports watch time and a platform
   that does not.

3. Confirm the query layer and the API pass the new field through
   without changes. If they do not, make the smallest change that lets
   them.

4. In PlatformMetricsCard, display total watch time formatted as hours
   and minutes. Omit it entirely for a platform that reports none rather
   than showing zero.

Do not change the database schema. Do not change the platform modules.
--- END PROMPT P8.4 ---
```

EXPECTED RESULT

An updated calculation with tests, an unchanged schema, and watch time on the cards for platforms that report it.

VERIFICATION

1. `npm test` exits 0, including the two new tests.
2. Sum `watch_time_seconds` across the latest snapshot per post per platform over the last 30 days with a direct SQL query. Confirm `GET /api/analytics?view=overview` returns the same figures.
3. Confirm a platform reporting no watch time returns the field as absent, not zero, at every layer: the aggregate, the API response, and the rendered card.
4. Confirm `git diff` touches no file under `src/lib/platforms/` and no schema file.
5. `npm run build` exits 0.

GATE: Checks 2, 3, and 4 pass.

---

## Changing it further

**Changing what the application does starts in `SPEC.md`.** Edit the specification, then instruct the agent to bring the application in line with it. Every task in this pack reads the specification rather than containing it, which is what makes this work.

**Swapping a platform** is the largest version of that. To replace TikTok with a different service, edit the platform list and the "Platform Sync" section of `SPEC.md`, then rerun the tasks from `P5.3` onward. A new platform module must produce the same `FetchedPost` and `NormalizedMetrics` shapes as the existing three; everything above that layer is written against those shapes and does not depend on their source.

## Running it beyond one machine

The application has no login screen. That is recorded in `SPEC.md` as a decision: it is built for one person running locally.

The consequence is that it must not be placed on a public address as it stands. Platform tokens are in the database, and anyone reaching the settings page could remove them or read the account identities behind them.

Two paths exist. Run it at home and reach it over a VPN or private mesh network, so it is never publicly exposed. Or add authentication before any hosting, and move credentials into managed secret storage rather than a file on disk. Both are outside this pack.

## Checkpoint

```text
Commit all current changes with the message "Phase 8: tests, operations,
and first extension", tag the commit v1.0, and push both the commit and
the tag.
```

## Drift check

```text
Read SPEC.md in full and compare it against the finished application.

1. P8.4 added total watch time to the platform metric cards. SPEC.md's
   "Analytics Dashboard" section does not describe that field. Add it to
   the platform metric cards bullet, so the specification still describes
   what the application does. This is the only place where a task in this
   pack changes behavior ahead of the specification, and closing it is
   what keeps SPEC.md the source of truth for anything built after this
   pack ends.

2. List every other difference between SPEC.md and the application: a
   behavior the specification describes that is not built, and a behavior
   that is built and not described. P8.2 audited the same ground before
   P8.3 and P8.4 ran; this pass covers what those two changed.

3. For each difference, state which side is wrong. Fix the specification
   where the application is right. Report, without fixing, anywhere the
   application is wrong, so the operator decides whether to change it.

Commit any SPEC.md edit with the message "Record watch time in the
specification" and push it.
```

## What you have now

A complete, tested application. It plans content on a board and a calendar, holds scripts and per-platform variants, pulls live metrics from YouTube, TikTok, and Instagram into a local database, compares each period against the one before it, finds posts that outperformed their platform average, and runs an AI layer across all of it that suggests what to do next without changing anything on its own.
