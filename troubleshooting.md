# Troubleshooting Index

Symptom, cause, and fix, by phase. This covers the failures most likely to occur during a build. It is not exhaustive.

Two rules apply everywhere. When a gate fails, fix the failing task and re-run its verification rather than building the next task on top of it. When a phase has gone badly wrong, returning to its tag is usually faster than repairing forward:

```text
Report what would be lost by resetting to the tag phase-N-complete, then
wait for confirmation before doing anything.
```

## Phase 1: Prerequisites and credentials

| Symptom | Cause and fix |
|---------|---------------|
| `gh: command not found` | The GitHub CLI is not installed. Re-run `P1.0`. |
| `gh auth status` reports not logged in | Installing `gh` does not sign in. This is an operator action listed in `SETUP.md`; it cannot be performed by the agent. |
| `gh auth status` names the wrong account | Signed in to a different GitHub account. `gh auth logout`, then `gh auth login`. |
| `P1.4` reports `origin` belongs to another account | The original pack was cloned instead of your own copy. On the pack's GitHub page use **Use this template**, clone the new repository, and start again from `P1.0`. |
| Push to `origin` is rejected with a permission error | Same cause: `origin` is the original pack, which you cannot write to. Create your own copy from the template. |
| `gh repo create` reports the name exists | Only when the pack was downloaded rather than cloned: a repository named `content-command-center` is already on the account. Delete it or use a different name in `P1.4`. |
| `pg_isready` reports no response | PostgreSQL is installed but not running. Re-run `P1.0`, which distinguishes the two states. |
| `git check-ignore` prints nothing for `.env.local` | The `.gitignore` is in the wrong folder. It must sit beside `.git`, not inside it and not in a parent. Stop; no credential may be written until this passes. |
| Anthropic key rejected as invalid | Usually a partial copy. Keys are long. Reissue and copy in full. |
| Anthropic response mentions credit balance | Valid key, no payment method on the account. |
| `fetch is not defined` in a verification script | Node is older than 20. Re-run `P1.0`. |
| YouTube returns an empty result with no error | The key works but the channel ID is wrong. It begins with `UC`. |
| YouTube returns 403 saying the API is not enabled | YouTube Data API v3 was not enabled, or was enabled on a different Google Cloud project than the one that issued the key. |
| Instagram account type is `PERSONAL` | Convert to Business or Creator, then generate a new token. An existing token does not gain permissions retroactively. |
| Instagram token expires within the hour | A short-lived token was generated instead of a long-lived one. |
| TikTok code rejected as invalid or expired | Authorization codes are single-use and expire in minutes. Obtain a fresh code and exchange it immediately. Do not report this as a bad client secret. |
| TikTok redirect mismatch | The redirect address in the authorization URL must match the registered one exactly, including any trailing slash. |
| TikTok exchange succeeds but user info returns nothing | The operator's account was not added as a sandbox target user. |

## Phase 2: Foundation

| Symptom | Cause and fix |
|---------|---------------|
| The bootstrapper refuses, saying the directory is not empty | A file from phase 1 is blocking it. Report which one. Leftover verification scripts are the usual cause. |
| `drizzle-kit push` cannot connect | `DATABASE_URL` is empty or wrong in `.env.local`, or PostgreSQL is not running. |
| `.env.local` has empty values | `P2.1` step 6 did not read `.env.credentials.local`. Re-run that step. |
| Tables created but columns do not match the spec | Re-run `P2.2` naming the specific table. This is inexpensive now and expensive in phase 6. |
| A route returns 404 | The route group folder name must be parenthesized, and page files named as the installed framework version expects. |

## Phase 3: Content

| Symptom | Cause and fix |
|---------|---------------|
| A `PATCH` to change category returns 500 | The update route is rejecting the value. Check it against the category enum in `SPEC.md`. |
| Clicking a card starts a drag | The 8px activation distance is missing from the pointer sensor. |
| Clicking the drag handle navigates to the editor | The handle is not excluded from the click handler. |
| Script does not survive a reload | Autosave is not firing, or is writing the wrong field. The script belongs in `script_notes`. |
| The stored date is the day before the one set | The date is being parsed from the `YYYY-MM-DD` string rather than built from local year, month, and day. |
| A scheduled piece shows `12:00 AM` | Expected. The date input has no time component, so the date lands at local midnight. |
| Variant fields reset after a save | The form is sending only changed fields. `platform_data` is replaced wholesale; every save must include all fields. |
| More than one row in `scratchpad` | The single-row behavior is wrong. It must find the existing row before inserting. |

## Phase 4: Calendar

| Symptom | Cause and fix |
|---------|---------------|
| Dates align under the wrong weekday | Inconsistent week start. Every week boundary calculation must use a Monday start. |
| A piece renders one day early | Days are being compared in UTC rather than local time. |
| A drop moves the date but not the status | The drop must write both `scheduledFor` and `status` in one request. |
| A failed drop leaves the interface and database disagreeing | The rollback restored only one of the two fields. Both must be restored. |

## Phase 5: Platform integrations

| Symptom | Cause and fix |
|---------|---------------|
| Only the first 50 YouTube videos appear | Paging is not following the page token, or stops after one page. |
| Far fewer YouTube videos than the channel has | `search.list` is being used instead of the uploads playlist. It caps at roughly 500 results. |
| Instagram reach equals view count on every post | The insights requests are failing while the media requests succeed. Inspect the insights response. |
| Instagram returns a permissions error naming a scope | The Meta app is missing a required permission for the Instagram product. |
| TikTok works for a day, then stops | The rotated refresh token was not persisted. TikTok returns a new one on every refresh and invalidates the old. |
| A credential is stored even though the token is bad | Validation is running after the save, or not at all. It must run first and store nothing on rejection. |
| A token appears in an API response | The credentials route is returning the full row. It must return metadata and a `hasRefreshToken` boolean only. |
| `P5.5` connects no platforms | `.env.credentials.local` is missing, empty, or was deleted. The values are not recoverable from the repository; the operator must re-obtain them. |
| Sync doubles the snapshot count on a second run | The daily snapshot key is wrong. One snapshot per post per day; a second run overwrites. |
| A post's link to a draft disappears after a sync | The upsert is overwriting `content_variant_id`. It must be excluded from the update set. |
| One platform failing stops the whole sync | The platforms are not running under `Promise.allSettled`. |

## Phase 6: Analytics dashboard

| Symptom | Cause and fix |
|---------|---------------|
| Most rows in the post table have no title | The caption fallback is missing. Instagram and TikTok posts have captions, not titles. |
| Every growth figure is a dash | The previous window is not being loaded, or there is genuinely no data in it. A dash is correct when the earlier period is empty. |
| Growth shows `0%` where it should show a dash | Null and zero are being conflated. They mean different things and must render through distinct branches. |
| One post flattens every bar to the baseline | The axis cap is not applied, or there are fewer than six data points, which correctly disables capping. |
| Bars are cut off with no indication | The cap is applied but not communicated. Clipped bars need a torn edge, a true-value label, and a note in the header. |
| An invalid query parameter returns 500 | The parameter check is inside the general error handler. A rejected parameter is a 400. |
| Freshly synced data does not appear | The date range was not re-resolved after the sync, so a snapshot stamped moments ago falls outside a range fixed at page load. |
| Tests cannot import the aggregation helpers | The helpers are importing database code. They must import none. |
| The dashboard is empty and sync returns nothing | No platform is connected, or the accounts have no published posts. Run `seed-data.md`. |

## Phase 7: AI integration

| Symptom | Cause and fix |
|---------|---------------|
| Every route fails to load when the API key is missing | The Anthropic client is constructed at import time instead of on demand. |
| The stream throws on every second render, in development only | React StrictMode mounts twice and the stream is read twice. Guard against a locked stream and release the reader on cleanup. |
| Approve and Dismiss do nothing | The suggestion id is not reaching the client. It comes from the `X-Suggestion-Id` header, sent before the text finishes. |
| Approving a suggestion changes a draft | Approval must record a decision only. It copies nothing. |
| Text takes a long time to start appearing | Extended thinking is enabled. It must be off for these responses. |
| Repurposed ideas save but the editor will not open them | The generated document contains an empty text node, which the editor rejects. Emit a bare paragraph instead. |
| Repurposing creates suggestion rows | It must not. The saved drafts are the artifact; the ideas are not persisted. |
| Caption generation is slow with three variants | The requests are running in sequence. They must run in parallel. |
| A 500 mentioning the API key | `ANTHROPIC_API_KEY` is empty in `.env.local`. `P2.1` copies it from `.env.credentials.local`. |

## Phase 8: Test, audit, operate

| Symptom | Cause and fix |
|---------|---------------|
| Tests are slow or fail intermittently | A test is calling a real API. Replace the boundary with recorded responses. |
| The suite passes after a deliberate break | The tests are not asserting on the thing that changed. Add one that does. |
| `git log --all -- .env.local` returns output | The file entered history. Revoke and reissue every credential it contained. |
| A restore produces fewer rows | The dump was taken during a write, or the restore failed partway. Compare row counts rather than trusting the exit code. |

## When the index does not cover it

State three things before changing anything: what was expected, what happened, and the exact error text. Then state the diagnosis. Editing before stating a diagnosis is guessing, and a guess that removes the symptom often leaves the cause.
