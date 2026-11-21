# Run

This file is the entry point. Give the prompt below to a coding agent opened at this folder, once per phase.

## One phase per session

Eight phases, eight sessions. Open a session, give it the run prompt with that phase's file named, and let it work to the phase's checkpoint. Then start the next phase in a fresh session. Do not run the whole pack in one conversation because the context window will get polluted and important information will get lost.

## The run prompt

```text
--- PROMPT RUN ---
You are building the application described in SPEC.md by executing this
prompt pack. AGENTS.md holds the execution contract and the verification
policy that govern every task. Read it before starting, and follow it
for the whole session.

This session executes one phase: <PHASE FILE>.

Run every task in that file, in order, from its first to its last. For
each task, in this order:
  a. Execute the text between its --- PROMPT Pn.n --- and
     --- END PROMPT Pn.n --- delimiters.
  b. Run every numbered check in its VERIFICATION section.
  c. Evaluate its GATE line. If the gate is not met, stop and report
     which task failed, which check failed, and the exact output.

Do not skip a task. Do not run tasks out of the order given by their
Requires lines. Do not continue past an unmet gate. Do not run a task
from any other phase file, and do not read ahead to one.

When every task in the phase has passed its gate, run that phase's
Checkpoint prompt, then its Drift check prompt, report the result of
both, and stop. The next phase runs in a new session.

P1.2 halts and waits for the operator. Present the report it asks for
and wait there.
--- END PROMPT RUN ---
```

Replace `<PHASE FILE>` with the phase for that session: `phase-1-prerequisites.md` first, through `phase-8-ship.md` last. `P1.1` moves the phase files into `docs/prompt-pack/`, so from the second session on, name that path.

If a phase is long enough that a single session still runs out of room, stop at the last task that passed its gate, commit, and resume in a new session by naming the phase file and the task to start from. The requirement is only that a task never begins before its predecessor's gate has passed.

## Files

| File | Role |
|------|------|
| `AGENTS.md` | Standing rules, execution contract, verification policy. Read at the start of every session. |
| `SPEC.md` | The specification. Every task cites it. Editing it changes what gets built. |
| `RUN.md` | This file. The run prompt and the contents of the pack. |
| `phase-1-prerequisites.md` | Tooling, local repository, the operator halt, credential validation, GitHub remote |
| `phase-2-foundation.md` | Application, database schema, navigation shell |
| `phase-3-content.md` | Board, editor, platform variants, scratchpad |
| `phase-4-calendar.md` | Month calendar and drag-to-schedule |
| `phase-5-platforms.md` | Platform modules, credentials, sync |
| `phase-6-analytics.md` | Calculations, API, dashboard, charts, post detail |
| `phase-7-ai.md` | AI layer, from client through repurposing |
| `phase-8-ship.md` | Tests, audit, operations, extension |
| `seed-data.md` | Optional. Fills the database with a realistic fictional catalog. |
| `troubleshooting.md` | Symptom to diagnosis to fix, per phase. |

After `P1.1` these files live in `docs/prompt-pack/`, except `AGENTS.md`, `README.md`, and `SPEC.md`, which stay at the repository root.

## Task count

45 tasks: 6, 4, 7, 3, 7, 7, 7, 4 across phases 1 through 8.

Each phase runs in its own session, so this table is also the reading order.

## Why files rather than conversation

The prompts in phases 2 through 8 are short because they do not restate requirements; they cite `SPEC.md`. A file does not degrade the way a conversation does, so the schema can be re-read in phase 8 with the accuracy it had in phase 2. That is what makes one phase per session workable rather than merely tidy.

It is also what makes the pack customizable. Editing `SPEC.md` changes what all 45 tasks build.
