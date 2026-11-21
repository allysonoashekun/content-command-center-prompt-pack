# Phase 1: Prerequisites and Credentials

## What this phase delivers

A version-controlled project folder backed up to GitHub, and validated working access to Anthropic, YouTube, Instagram, and TikTok.

Credentials are obtained and proven here rather than in phase 5, where a failure could plausibly be caused by the credential, the code, or the specification. Verifying each one in isolation against the live service removes that ambiguity.

This phase contains the pack's only designed halt. `P1.2` stops and waits for the operator to create four developer accounts and supply seven credential values. No other task in this phase or in the rest of the pack is written to stop, with one qualification: `P1.0` may install or upgrade system software, and it confirms before doing so. That is a safety confirmation rather than a handoff, and the run continues from the answer.

Git work is split across that halt. The operator starts from their own copy of the pack, created from the GitHub template and cloned to this machine. `P1.1` files the guides, writes the `.gitignore`, and commits locally, which is entirely offline; `P1.4` pushes to the operator's repository, which needs the `gh` authentication the operator performs in `P1.2`. The local half comes first because the `git check-ignore` gate in `P1.1` is what makes it safe for `P1.2` to write a credential file to disk.

---

### P1.0 - Install and verify the required tooling

Requires: None
Halts: no, other than to confirm a system-modifying install before running it

OBJECTIVE

Node.js 20 or later with npm, PostgreSQL 14 or later with its server running, Git, and the GitHub CLI, all present on this machine.

PROMPT

```text
--- PROMPT P1.0 ---
Survey and install the tooling this build requires. Ask before running
any command that modifies the system.

1. Report the installed version of each of node, npm, git, psql, and gh,
   or state that it is not installed. Flag node below version 20 and
   psql below version 14 as problems.

2. Report whether the PostgreSQL server is running and accepts a
   connection. Installed and running are different states; check both.

3. Identify the operating system and the package manager standard for
   it.

4. Install every missing or outdated tool using that package manager,
   asking before each command. Start the PostgreSQL server if it is
   installed but not running.

5. Do not run gh auth login. It is interactive and requires a browser.
   The operator authenticates gh in P1.2. Nothing before P1.4 contacts
   GitHub, so an unauthenticated gh does not block P1.1.

6. Present a table with one row per tool, a status of OK or MISSING, and
   the installed version where there is one.

Do not create any file. Do not initialize a Git repository. Do not
install Node packages or anything that is not one of the five tools
above.
--- END PROMPT P1.0 ---
```

EXPECTED RESULT

A five-row table. Nothing written to the project folder.

VERIFICATION

1. `node --version` reports 20 or later.
2. `npm --version`, `git --version`, `gh --version` each print a version.
3. `psql --version` reports 14 or later.
4. `pg_isready` reports `accepting connections`.

GATE: All four checks pass. `gh` is installed but not yet authenticated, which is expected at this point.

---

### P1.1 - Initialize the local repository

Requires: P1.0
Halts: no

OBJECTIVE

The operator's clone of the pack prepared as the project repository, with a `.gitignore` proven to exclude credential files, the pack's guide files filed under `docs/prompt-pack/`, and a local commit made.

Local only. `git add` and `git commit` run entirely on this machine and need no network and no authenticated `gh`. Pushing to GitHub is `P1.4`, after the operator has authenticated in `P1.2`.

The `.gitignore` is written here because `P1.2` writes the first credential file, and the `git check-ignore` gate below is what makes that safe.

PROMPT

```text
--- PROMPT P1.1 ---
Work in the current directory. This directory is the operator's own copy
of the prompt pack, created from the GitHub template and cloned here, and
it is where the application will be built. Do not create a new project
folder and do not clone anything.

1. Confirm the current directory contains AGENTS.md, README.md, SPEC.md,
   RUN.md, and phase-1-prerequisites.md. If it does not, stop and report
   what the directory contains.

2. Create a directory named docs/prompt-pack and move these files into
   it, preserving their names:

   RUN.md
   phase-1-prerequisites.md
   phase-2-foundation.md
   phase-3-content.md
   phase-4-calendar.md
   phase-5-platforms.md
   phase-6-analytics.md
   phase-7-ai.md
   phase-8-ship.md
   seed-data.md
   troubleshooting.md

   Move any other markdown file that shipped with the pack into the same
   directory. Leave AGENTS.md, README.md, and SPEC.md at the top level.
   If a listed file is not present, report it and continue.

3. Create or extend the .gitignore at the top level so it contains, at
   minimum:

   node_modules/
   .next/
   .env
   .env.local
   .env.credentials.local
   .env*.local
   SETUP.md
   backups/
   *.log
   .DS_Store
   tsconfig.tsbuildinfo

   SETUP.md is listed because P1.2 writes it at the root as the
   operator's own working sheet. It is not part of the application and
   must not reach the remote.

4. This directory should already be a Git repository, cloned from the
   operator's copy of the template, with origin pointing at it. Do not
   run git init and do not replace or re-point origin. If the directory
   is not a Git repository at all (the pack was downloaded rather than
   cloned), initialize one with the default branch named main.

   Make sure the current branch is main. Stage everything and commit
   with the message "Initial commit: prompt pack and gitignore". This is
   local work only: do not push.

5. Confirm that .env.local, .env.credentials.local, and SETUP.md are all
   matched by the ignore rules, using git check-ignore, and report the
   evidence. None of the three exists yet; check-ignore matches on the
   path, so it answers before P1.2 creates them. Do not continue if any
   is unmatched.

6. List every file and directory at the top level, including hidden ones.
   The tracked top level must contain only .git, .gitignore, docs,
   AGENTS.md, README.md, and SPEC.md. A configuration directory belonging
   to the coding agent or editor in use - .claude, .cursor, .vscode, or
   similar - may also be present; note it and continue. Report anything
   else.

Every step above is local. Do not run gh, do not create a GitHub
repository, do not change any remote, and do not push. Pushing happens
in P1.4, after the operator authenticates gh in P1.2. An unauthenticated
gh is expected here and is not a reason to stop.

Do not create any other file. Do not install anything. Do not modify the
contents of any pack file.
--- END PROMPT P1.1 ---
```

EXPECTED RESULT

A Git repository whose newest commit is the one made here. The top level holds `.git`, `.gitignore`, `docs/`, `AGENTS.md`, `README.md`, and `SPEC.md`. The pack's guide files are at `docs/prompt-pack/`. Nothing has been pushed; `P1.4` does that.

The guide files are read from their new paths from this point forward.

VERIFICATION

1. `git check-ignore -v .env.local` prints the matching rule. No output means the file is unprotected.
2. `git check-ignore -v .env.credentials.local` prints the matching rule.
3. `git log --oneline -1` shows "Initial commit: prompt pack and gitignore". A repository created from the template also carries GitHub's own first commit beneath it; that is expected.
4. `ls -a` lists `.`, `..`, `.git`, `.gitignore`, `docs`, `AGENTS.md`, `README.md`, `SPEC.md`, and nothing else beyond an agent or editor configuration directory.
5. `ls docs/prompt-pack` lists the eleven moved files.
6. `git status -sb` shows `main` ahead of `origin/main` by the commit made here, or shows no upstream if the pack was downloaded rather than cloned. Nothing has been pushed.
7. `git check-ignore -v SETUP.md` prints the matching rule, so the file `P1.2` is about to write is already covered.

GATE: Checks 1, 2, and 7 all print a matching rule. No credential may be written to disk until they do.

---

### P1.2 - Operator setup

Requires: P1.1
Halts: yes - the operator authenticates the GitHub CLI, creates four developer accounts, and fills in `.env.credentials.local`

OBJECTIVE

Every credential the build needs, present in `.env.credentials.local`, and the GitHub CLI authenticated.

This is the only task in the pack that stops for a person. Developer accounts cannot be created programmatically and `gh auth login` requires a browser. Everything a person must do is collected here so it happens in one sitting.

PROMPT

```text
--- PROMPT P1.2 ---
Prepare the operator handoff, then stop and wait.

1. Create .env.credentials.local at the repository root containing
   exactly these keys, each with an empty value, in this order:

   ANTHROPIC_API_KEY=
   TIKTOK_CLIENT_KEY=
   TIKTOK_CLIENT_SECRET=
   YOUTUBE_API_KEY=
   YOUTUBE_CHANNEL_ID=
   INSTAGRAM_ACCESS_TOKEN=
   TIKTOK_ACCESS_TOKEN=
   TIKTOK_REFRESH_TOKEN=
   TIKTOK_EXPIRES_IN=

   Confirm with git check-ignore that this file is ignored. If it is not,
   delete it and stop.

2. Write SETUP.md at the repository root. It is the operator's
   instruction sheet and the only document in this build written for a
   person. It must contain, in this order:

   - A statement that the build is paused and will resume when the
     operator says to continue.
   - GitHub CLI: run gh auth login, choose HTTPS, and answer yes when
     asked whether to authenticate Git operations.
   - Anthropic: create an account at console.anthropic.com, add a payment
     method under billing, create an API key, and copy it immediately
     because it is shown once. Fills ANTHROPIC_API_KEY.
   - YouTube: at console.cloud.google.com create or select a project,
     enable YouTube Data API v3 under APIs and Services then Library,
     create an API key under Credentials, and copy the channel ID from
     the YouTube channel's About page via Share channel. The channel ID
     begins with UC. Fills YOUTUBE_API_KEY and YOUTUBE_CHANNEL_ID.
   - Instagram: the account must be Business or Creator; personal
     accounts cannot read these metrics at all. Create an app at
     developers.facebook.com, add the Instagram product with Instagram
     Login, link the Instagram account, and generate a long-lived access
     token. Fills INSTAGRAM_ACCESS_TOKEN.
   - TikTok: at developers.tiktok.com create an app, add Login Kit and
     Display API, request the user.info.basic and video.list scopes, add
     the operator's own TikTok account as a sandbox target user, and
     register a redirect address. Any address the operator controls is
     acceptable, including one that does not load, because only the code
     appended to it is needed. Fills TIKTOK_CLIENT_KEY and
     TIKTOK_CLIENT_SECRET.
   - TikTok tokens: state that the access token, refresh token, and
     expiry are obtained by an authorization exchange, that this build
     performs that exchange automatically in P1.3, and that the operator
     only needs to complete the browser authorization step described
     there. Then describe that step: open the authorization URL this
     prompt prints below, approve access, and copy the code parameter out
     of the resulting address bar into TIKTOK_AUTH_CODE in
     .env.credentials.local. Note that the code expires within minutes,
     so it should be obtained last, immediately before resuming.
   - A table of every key in .env.credentials.local, which service it
     comes from, and whether the operator fills it or P1.3 does. Seven
     are the operator's; TIKTOK_ACCESS_TOKEN, TIKTOK_REFRESH_TOKEN, and
     TIKTOK_EXPIRES_IN are written by P1.3 and are to be left empty.
   - A note that each console description was verified as of August 2026
     and that the operator should consult the provider's current
     documentation if an interface differs.
   - A warning not to paste any credential into the agent conversation.
     The file is read directly.

3. Add TIKTOK_AUTH_CODE= to .env.credentials.local, after
   TIKTOK_CLIENT_SECRET.

4. Print the TikTok authorization URL template, with the client key and
   redirect address left as clearly marked substitutions the operator
   fills from their own app, requesting the user.info.basic and
   video.list scopes.

5. Report to the operator: the path to SETUP.md, the path to
   .env.credentials.local, and the seven keys they fill themselves -
   ANTHROPIC_API_KEY, YOUTUBE_API_KEY, YOUTUBE_CHANNEL_ID,
   INSTAGRAM_ACCESS_TOKEN, TIKTOK_CLIENT_KEY, TIKTOK_CLIENT_SECRET, and
   TIKTOK_AUTH_CODE. State that the remaining three keys,
   TIKTOK_ACCESS_TOKEN, TIKTOK_REFRESH_TOKEN, and TIKTOK_EXPIRES_IN, are
   written by P1.3 from the authorization exchange and are to be left
   empty. Close with a statement that the build resumes on their
   instruction.

Then stop. Do not proceed to P1.3. Do not create a GitHub repository. Do
not write any application code. Do not ask the operator to paste any
value into this conversation.
--- END PROMPT P1.2 ---
```

EXPECTED RESULT

Two new files, `SETUP.md` and `.env.credentials.local`, and a halted run. Both are ignored by the rules `P1.1` wrote.

The operator completes `SETUP.md`, fills in the seven values that come from the developer consoles, and instructs the agent to continue. The three TikTok token keys stay empty; `P1.3` writes them.

An operator who does not use all three platforms may leave that platform's keys empty. `P1.3` reports them as absent rather than failing, and phases 5 through 7 build against whichever platforms are connected.

VERIFICATION

1. `git check-ignore -v .env.credentials.local` and `git check-ignore -v SETUP.md` each print the matching rule.
2. `SETUP.md` exists at the repository root and covers all five items: GitHub CLI, Anthropic, YouTube, Instagram, TikTok.
3. `.env.credentials.local` contains ten keys, seven of them for the operator to fill.
4. `git status --short` lists neither `SETUP.md` nor `.env.credentials.local`.
5. No credential value appears anywhere in the conversation.

GATE: The operator has stated that `SETUP.md` is complete. Do not run `P1.3` before then.

---

### P1.3 - Validate every credential

Requires: P1.2
Halts: no

OBJECTIVE

Every supplied credential confirmed working against its live service, the TikTok authorization code exchanged for tokens, and the GitHub CLI confirmed authenticated.

PROMPT

```text
--- PROMPT P1.3 ---
Validate the credentials in .env.credentials.local. Read values from that
file directly. Never print a credential in full and never ask the
operator to paste one.

1. Confirm gh is authenticated by running gh auth status, and report
   which GitHub account it names. If it is not authenticated, stop and
   tell the operator to run gh auth login.

2. Write a temporary script at scripts/verify-credentials.mjs that loads
   .env.credentials.local and checks each service. Treat an empty value
   as "not supplied" and report it as such rather than as a failure.

   Anthropic: send one Messages API request with max_tokens of 1.
   Distinguish three outcomes: valid with billing active, rejected as
   invalid, and valid but unbillable.

   YouTube: call the Data API v3 channels endpoint with
   part=snippet,statistics for YOUTUBE_CHANNEL_ID. Report the channel
   title, subscriber count, and video count.

   Instagram: call graph.instagram.com/me requesting id, username, and
   account_type. Report the username and account type. If the account
   type is not BUSINESS or MEDIA_CREATOR, report that metrics will not be
   readable.

   TikTok: if TIKTOK_ACCESS_TOKEN is empty and TIKTOK_AUTH_CODE is set,
   exchange the code at /v2/oauth/token/ using TIKTOK_CLIENT_KEY and
   TIKTOK_CLIENT_SECRET, then write the returned access token, refresh
   token, and expires_in back into .env.credentials.local under
   TIKTOK_ACCESS_TOKEN, TIKTOK_REFRESH_TOKEN, and TIKTOK_EXPIRES_IN.
   Then call /v2/user/info/ and report the display name, whether a
   refresh token was returned, and the seconds until expiry.

   Authorization codes are single use and expire within minutes. If the
   exchange fails as expired, report exactly that and tell the operator
   to fetch a fresh code, rather than reporting a bad client secret.

3. Run the script and present a table with one row per service, a status
   of OK, NOT SUPPLIED, or FAILED, and the identity the service returned.
   For a failure, report the service's own error message.

4. Delete scripts/verify-credentials.mjs and confirm no verify-* script
   remains anywhere in the repository.

Do not write any application code. Do not create a database. Do not
commit .env.credentials.local.
--- END PROMPT P1.3 ---
```

EXPECTED RESULT

A status table with one row per service. `.env.credentials.local` now holds TikTok's access token, refresh token, and expiry, written by the exchange. The verification script is deleted.

The identity each service returns is the substantive check. The YouTube channel title, Instagram username, and TikTok display name must belong to the operator. A different name means the wrong identifier was supplied.

VERIFICATION

1. `gh auth status` reports a logged-in account.
2. Anthropic reports valid with billing active, or the operator has accepted that the AI features in phase 7 will not run.
3. Each supplied platform reports OK with an identity returned by the platform.
4. If TikTok is in use, `.env.credentials.local` now contains non-empty `TIKTOK_ACCESS_TOKEN` and `TIKTOK_REFRESH_TOKEN`. A missing refresh token means the application cannot renew access and TikTok data stops updating after 24 hours.
5. `ls scripts/` does not exist or contains no `verify-*` file.
6. `git status --short` does not list `.env.credentials.local`.

GATE: No service reports FAILED. `NOT SUPPLIED` is acceptable for a platform the operator is not connecting.

---

### P1.4 - Push to the GitHub repository

Requires: P1.3
Halts: no

OBJECTIVE

The commit made in `P1.1` pushed to the operator's own GitHub repository, the remote named `origin`.

PROMPT

```text
--- PROMPT P1.4 ---
1. If a remote named origin exists, confirm with gh that it belongs to
   the authenticated account. If it belongs to a different account, such
   as the original pack the template came from, stop and report it: the
   operator cloned the original instead of creating their own copy.
   Otherwise push main to origin.

   If there is no origin (the pack was downloaded rather than cloned),
   create a private GitHub repository named content-command-center with
   gh, set it as origin, and push main. If a repository with that name
   already exists on the account, stop and report it rather than pushing
   to it.

2. Report the repository's owner and visibility. If it is public, say so
   and recommend making it private in its GitHub settings. Do not change
   the visibility yourself.

3. Confirm the push succeeded and that the local main branch matches
   origin/main.

4. Confirm that neither .env.local nor .env.credentials.local was
   pushed, by listing the files present on the remote.

Do not create any file. Do not install anything.
--- END PROMPT P1.4 ---
```

EXPECTED RESULT

The operator's GitHub repository containing `README.md`, `SPEC.md`, and `docs/`, level with the local `main`.

VERIFICATION

1. `git remote -v` names `origin`, owned by the account `gh auth status` reports.
2. `git status -sb` reports the local branch level with `origin/main`.
3. `gh repo view --json owner,isPrivate` reports the operator's account. `isPrivate` should be `true`; if it is `false`, the report says so. Public is not a failure, since no credential is ever committed.
4. `git ls-tree -r origin/main --name-only` does not list `.env.local`, `.env.credentials.local`, or `SETUP.md`.

GATE: Check 4 lists no credential file. If one is present, revoke every credential it contained and reissue before continuing.

---

### P1.5 - Preflight

Requires: P1.4
Halts: no

OBJECTIVE

A single report confirming the machine, the repository, and the credentials are in the state phase 2 requires.

PROMPT

```text
--- PROMPT P1.5 ---
Produce a readiness report for phase 2. Do not fix anything. Report only.

1. Tooling. Report the installed versions of node, npm, git, psql, and
   gh. Flag node below 20 or psql below 14. Confirm gh is authenticated.

2. PostgreSQL. Confirm the server is running and accepts a connection.

3. Repository state. List every file and directory at the top level,
   including hidden ones. Expected: .git, .gitignore, docs, AGENTS.md,
   README.md, SETUP.md, SPEC.md, and .env.credentials.local. An agent or
   editor configuration directory may also be present; note it without
   flagging it. Flag anything else, especially leftover verification
   scripts.

4. Ignore rules. Confirm .env.local, .env.credentials.local, and
   SETUP.md are all ignored by Git, and that none of them appears in the
   repository history.

5. Credentials. Report which of the four services validated in P1.3
   returned OK, and which were not supplied. Do not print any value.

6. Git state. Confirm the working tree is clean and that the local branch
   matches the remote. SETUP.md and .env.credentials.local are ignored
   and so are absent from git status; that is the expected state, not a
   pending change.

Present the results as a table with a status of READY or ACTION NEEDED
for each item, followed by the specific action required for anything not
ready.
--- END PROMPT P1.5 ---
```

EXPECTED RESULT

A six-row table.

VERIFICATION

1. Every row reports READY.
2. `git log --all --oneline -- .env.local .env.credentials.local SETUP.md` returns no output.
3. `git status --porcelain` prints nothing, apart from an untracked agent or editor configuration directory.

GATE: Every row reads READY. Phase 2 does not start otherwise.

---

## Checkpoint

```text
Commit any uncommitted changes with the message "Phase 1: prerequisites
and credentials". This phase may leave nothing to commit, because
SETUP.md and .env.credentials.local are both ignored; in that case say so
and tag the commit P1.1 made rather than creating an empty one. Tag it
phase-1-complete and push both the commit and the tag. Confirm
.env.credentials.local, .env.local, and SETUP.md are in neither the
commit nor the remote.
```

## Drift check

```text
Read SPEC.md and confirm the credentials validated in this phase cover
every external service the specification requires. List any service named
in the specification for which no credential validated.
```

## What you have now

A version-controlled project folder backed up to GitHub, holding the pack's guide files and nothing else, and validated access to every platform the operator supplied. No application code exists.

Phase 2 creates the application, the database, and the navigation shell.
