# Phase 2: Foundation

## What this phase delivers

A running Next.js application, a PostgreSQL database holding all seven tables, and the navigation shell every later screen sits inside.

The schema is defined in full here rather than a table at a time, so later phases add features rather than repeatedly altering the database structure.

---

### P2.1 - Bootstrap the application and install dependencies

Requires: P1.5
Halts: no

OBJECTIVE

A Next.js application in the repository with every specified package installed, a PostgreSQL database named `content_command_center`, and `.env.local` populated from the credentials the operator supplied in `P1.2`.

PROMPT

```text
--- PROMPT P2.1 ---
Read SPEC.md, including the pinned versions block and the environment
variables section. Install the versions recorded there rather than the
latest available.

1. Bootstrap a Next.js application. The repository root already holds
   AGENTS.md, README.md, SPEC.md, SETUP.md, and .env.credentials.local,
   and create-next-app refuses to write into a directory containing
   files it does not recognize. Scaffold into a temporary subdirectory
   and move the result up:

   npx create-next-app@latest next-init --typescript --tailwind --eslint
   --app --src-dir --import-alias "@/*" --no-turbopack --no-agents-md
   --skip-install --disable-git --yes

   The last four flags all matter. Without --no-agents-md the
   bootstrapper writes its own AGENTS.md and CLAUDE.md, and the move
   below would put its AGENTS.md over the pack's rules file. Without
   --disable-git it may initialize a second Git repository inside this
   one. --skip-install defers installation to step 2, so everything is
   installed once. The subdirectory name may not begin with a period.

   Separately, next dev appends a block delimited by
   <!-- BEGIN:nextjs-agent-rules --> to AGENTS.md when it detects a
   coding agent, and re-adds it if removed. It appends rather than
   replaces, so the pack's rules survive. Leave it in place and commit
   it; deleting it only produces the same change again on the next dev
   run.

   Then, in this order:

   - Move every file and directory from next-init/ to the repository
     root, including hidden ones, with the two exceptions below.
   - Leave next-init/README.md behind. The pack's README.md stays.
   - Do not move next-init/.gitignore over the existing one. Append only
     the rules it holds that the existing file lacks. Every rule P1.1
     wrote must survive. Its .env* rule is broader than P1.1's and would
     also ignore the .env.example written in step 6, so add !.env.example
     as the last line.
   - Delete next-init/ once it is empty.
   - Set the name field in package.json to content-command-center.
   - Confirm with git check-ignore that .env.local,
     .env.credentials.local, and SETUP.md are all still ignored. If any
     is not, restore the rule before continuing.

   If the bootstrapper refuses for any reason other than a non-empty
   directory, report the reason and stop.

2. Install the dependencies listed in SPEC.md together with the packages
   create-next-app recorded in package.json, in a single npm install.
   Do not install packages that are neither listed in SPEC.md nor
   already in package.json.

3. Initialize shadcn/ui and add the components listed in SPEC.md.

4. Set the dev script to bind to 127.0.0.1 explicitly.

5. Create a PostgreSQL database named content_command_center.

6. Create .env.local containing every variable named in the SPEC.md
   environment variables section. Populate it as follows:

   DATABASE_URL: postgresql://localhost:5432/content_command_center
   ANTHROPIC_API_KEY: copy from .env.credentials.local
   TIKTOK_CLIENT_KEY: copy from .env.credentials.local
   TIKTOK_CLIENT_SECRET: copy from .env.credentials.local

   Copy the values by reading the file, not by printing them. Do not add
   variables that SPEC.md does not name; the remaining platform
   credentials are loaded into the database in P5.5.

   Also create .env.example with the same variable names, empty values,
   and a short comment on each stating what it is and whether it is
   required.

7. Confirm .env.local and .env.credentials.local are both ignored by Git,
   then commit everything except them.

Do not create any application code, database table, or component.
--- END PROMPT P2.1 ---
```

EXPECTED RESULT

A Next.js project structure, `node_modules`, `src/`, a populated `.env.local`, an empty-valued `.env.example`, and a database with no tables. No `next-init/` directory remains, and `AGENTS.md`, `README.md`, `SPEC.md`, and `.gitignore` are the ones the pack and `P1.1` produced.

VERIFICATION

1. `psql -l` lists `content_command_center`.
2. `npm run dev` starts and `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3000` returns 200.
3. `git status --short` does not list `.env.local`, `.env.credentials.local`, or `SETUP.md`.
4. Every variable in `.env.local` has a non-empty value.
5. Read `package.json` and confirm no dependency is installed beyond those `SPEC.md` lists and those create-next-app installs with its own template.
6. `git diff phase-1-complete -- README.md SPEC.md` is empty, and `ls next-init` reports no such directory. For `AGENTS.md`, `git diff phase-1-complete -- AGENTS.md` may show one addition and nothing else: the block between `<!-- BEGIN:nextjs-agent-rules -->` and `<!-- END:nextjs-agent-rules -->` that `next dev` appends. Every rule the pack wrote must still be present above it. A diff that replaces the file rather than appending to it means the bootstrapper's `AGENTS.md` was moved over the pack's; restore it with `git checkout phase-1-complete -- AGENTS.md` and rerun.
7. `git check-ignore -v .env.local .env.credentials.local SETUP.md` prints a matching rule for all three, confirming the merged `.gitignore` kept them, and `git ls-files .env.example` lists the file, confirming the merge did not swallow it.

GATE: Checks 1 through 4 and checks 6 and 7 pass. Check 6 is what catches the bootstrapper overwriting the rules file the whole build reads.

---

### P2.2 - Define the database schema and connect to it

Requires: P2.1
Halts: no

OBJECTIVE

Seven tables created in PostgreSQL from a TypeScript schema, plus the database client the application uses.

PROMPT

```text
--- PROMPT P2.2 ---
Read SPEC.md section "Database Schema" in full before writing anything.

1. Create src/lib/db/schema.ts using Drizzle ORM, defining every enum and
   every table in that section exactly as specified: column names, types,
   nullability, defaults, indexes, unique constraints, and foreign keys
   with their delete behavior. Export an inferred select type and, where
   the specification records inserts, an inferred insert type for each
   table.

2. Create drizzle.config.ts pointing at that schema file, using the
   postgresql dialect, reading DATABASE_URL from .env.local.

3. Create src/lib/db/index.ts exporting a single db client built on the
   postgres.js driver with the schema attached.

4. Create src/types/index.ts holding the shared application types and
   label constants described in SPEC.md, including the ordered status and
   category lists and their display labels.

5. Push the schema to the database.

6. Report the tables that now exist, with their column counts.

Do not write migration files. Do not create API routes or components.
--- END PROMPT P2.2 ---
```

EXPECTED RESULT

Four new files and seven empty tables: `content_pieces`, `content_variants`, `platform_credentials`, `platform_posts`, `analytics_snapshots`, `scratchpad`, `ai_suggestions`.

VERIFICATION

1. `psql content_command_center -c "\dt"` lists exactly those seven tables.
2. `psql content_command_center -c "\d content_pieces"` shows both a `status` column and a `category` column. They are separate by design and phase 3 depends on both.
3. Compare every table and column in the database against the "Database Schema" section of `SPEC.md`. Report any column that is missing, extra, differently typed, or differently named.
4. `npm run build` exits 0.

GATE: Check 3 reports zero differences. A schema error is inexpensive now and expensive in phase 6, where analytics queries depend on exact column names.

---

### P2.3 - Build the application shell

Requires: P2.2
Halts: no

OBJECTIVE

Four routes that render, wrapped in a shared layout, with the root redirecting to `/analytics`.

PROMPT

```text
--- PROMPT P2.3 ---
Read SPEC.md sections "Shell & Navigation" and "File Structure". Read the
routing documentation shipped in node_modules for the installed Next.js
version before writing any route.

1. Create a route group named (dashboard) containing pages at /analytics,
   /drafts, /calendar, and /settings. Each page renders only its own name
   as a placeholder heading for now.

2. Create the (dashboard) layout as a full-height flex row: a main column
   that fills the remaining width, with overflow hidden on both the
   container and the main column. Mount the toast notification provider
   once here.

   Do not add a sidebar yet. P2.4 creates it and wires it into this
   layout.

3. Make the application root redirect to /analytics.

4. In the root layout, set the application metadata title and
   description. Use a font of your choosing from next/font/google; the
   visual design is deliberately not specified and is customized later.

5. In globals.css, define CSS custom properties for the three platform
   colors and their muted and foreground variants, plus success and
   warning colors, and expose them as Tailwind theme tokens. Later phases
   reference these tokens by name.

Do not create the sidebar, the top navigation component, or any feature
code.
--- END PROMPT P2.3 ---
```

EXPECTED RESULT

Five new page files, a layout, and a root redirect.

VERIFICATION

1. Request each of `/analytics`, `/drafts`, `/calendar`, `/settings` and confirm each returns 200. None may return 404 or 500.
2. Request `/` and confirm it returns a redirect to `/analytics`.
3. `npm run build` exits 0.

GATE: All five routes resolve.

---

### P2.4 - Build the sidebar and top navigation

Requires: P2.3
Halts: no

OBJECTIVE

A collapsible sidebar listing the four sections whose collapsed state survives a reload, and a reusable page header each page supplies its own title to.

PROMPT

```text
--- PROMPT P2.4 ---
Read SPEC.md section "Shell & Navigation".

1. Create the Sidebar as a client component with four navigation items:
   Analytics, Drafts, Calendar, Settings, each with an icon.

   - It toggles between an expanded and a collapsed width, with a short
     width transition.
   - When collapsed, labels are hidden and each item carries a title
     attribute so hovering reveals its name.
   - The collapsed state is written to localStorage under the key
     sidebar-collapsed and restored on mount.
   - The active item is determined by prefix, not exact match, so a
     nested route such as /analytics/posts/123 keeps Analytics
     highlighted.
   - The toggle button has an accessible label that reflects its current
     action.

2. Create TopNav as a server component taking a title and an optional
   actions slot rendered on the right. It is rendered by each page
   individually, not by the layout.

3. Wire the Sidebar into the (dashboard) layout, before the main column.

4. Render TopNav on each of the four placeholder pages with that page's
   name as its title.

Do not add navigation items that are not listed above.
--- END PROMPT P2.4 ---
```

EXPECTED RESULT

Two new components, an updated layout, and four updated pages.

VERIFICATION

1. Read `Sidebar` and confirm it writes to `localStorage` under the key `sidebar-collapsed` and reads it on mount. Quote the lines.
2. Read `Sidebar` and confirm the active item is matched by prefix rather than exact equality. Quote the line.
3. Confirm each collapsed navigation item carries a `title` attribute.
4. Request `/analytics` and confirm the response body contains all four navigation labels.
5. `npm run build` and `npm run lint` both exit 0.

GATE: Checks 1 and 2 both quote implementing lines. The persistence contract in check 1 is the one that fails most often.

---

## Checkpoint

```text
Commit all current changes with the message "Phase 2: foundation", tag
the commit phase-2-complete, and push both the commit and the tag.
```

## Drift check

```text
Read SPEC.md sections "Database Schema" and "Shell & Navigation" and
compare them against the current state of the application. List anything
specified there that is missing, and anything present that the
specification does not describe.
```

## What you have now

A running application with four routes, a working sidebar, and a database containing all seven tables, empty.

Phase 3 adds content creation, the board, and the script editor.
