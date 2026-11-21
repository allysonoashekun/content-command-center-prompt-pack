# Agent Rules

These rules govern every task in this build. They are not restated in task entries.

## Bootstrap

This directory is a prompt pack, not an application. Nothing here is source code yet.

Do not build the application directly from `SPEC.md`. `SPEC.md` states what the application must be; the pack states how it is built, in 45 verified steps. Read `RUN.md` and execute the run prompt it contains. Before `P1.1` that file is at the top level; after `P1.1` it is at `docs/prompt-pack/RUN.md`.

## Scope

Build only what `SPEC.md` specifies. Do not add features, routes, database columns, or dependencies that do not appear there. If a task appears to require something outside the specification, stop and report it rather than adding it.

## Dependencies

Do not install packages that are not listed in `SPEC.md`. If one appears necessary, stop and report it.

## Framework versions

This project uses package versions that may differ from your training data. Before writing code against any framework API, read the documentation shipped inside `node_modules` for the installed version. Do not rely on remembered API shapes. Observe deprecation notices.

## Existing patterns

Match the conventions already present in the codebase.

## Uncertainty

When required information is missing, stop and report what is missing. Do not guess a schema, an API response shape, or a file path.

## Operator input

One task in this build is designed to stop for the operator: `P1.2`.

Judgement remains yours for actions the pack does not clearly authorize and that are hard to undo: installing or upgrading system software, deleting or overwriting something the operator wrote, pushing to a repository that already exists. Confirming before one of those is correct even though it interrupts. `P1.0` is written that way deliberately. Keep such a confirmation to what is being changed and why, take the answer, and continue.

## Secrets

Never commit `.env.local`, `.env.credentials.local`, or any file containing credentials. Never print an API key or access token in full.

## Execution contract

**Order.** Tasks run in ascending order within a phase, and phases in ascending order. Each task's `Requires:` line names its predecessor. A task whose requirement has not completed and passed its gate must not run.

**Gates.** Every task ends with a `GATE:` line naming the condition under which the next task may run. An unmet gate stops the run. Do not repair forward past a failed gate by building the next task on top of it; fix the failing task, rerun its verification, then continue.

**Halts.** `P1.2` is the only task designed to stop for the operator. It stops because developer accounts cannot be created programmatically. Every credential the build needs is collected there, in one sitting. No later task asks the operator for a value. A confirmation before a hard-to-undo action, as described under Operator input, is not a halt.

**Sessions.** One phase per session. Run a phase from its first task to its Drift check, then stop; the next phase starts in a new session. Nothing needs to be carried across: this file is read at session start, and every prompt cites `SPEC.md` by section. A phase 8 session has the same grounding as a phase 1 session. `RUN.md` explains why the pack is built this way.

**Commits.** Each phase ends with a commit and a tag. Do not commit `.env.local` or `.env.credentials.local`.

## Verification

This policy applies to every `VERIFICATION` section in the pack.

**Run every check.** A check is not satisfied by inspecting code that looks correct. Run the command, make the request, execute the query.

**Report the output.** State the pass criterion and the actual result. Do not report a check as passed without the output that establishes it. Do not report a task complete until its checks have been run and their output examined. If a check fails, report the failure and stop rather than continuing to the next task.

**Browser-observable behavior.** Some requirements are drag-and-drop gestures, hover states, and values held in `localStorage`. Verify those in two parts:

1. **The persisted effect** — assert it through the API or a database query. A card dragged between lanes is verified by the row's `category` value, not by the animation.
2. **The code contract** — read the file you wrote and quote the lines that implement the requirement.

If browser automation is available in your environment, drive the interface directly and treat that as the primary check, with the two parts above as confirmation. If it is not available, the two parts above are sufficient and the run continues.

**The dev server.** Checks that make HTTP requests need `npm run dev` running and bound to `127.0.0.1:3000`. Start it in the background before the first such check and leave it running.

**Never call a live platform API from a test.** Tests use recorded responses. Live calls belong only in the credential validation in `P1.3` and in sync runs.
