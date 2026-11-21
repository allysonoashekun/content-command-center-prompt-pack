# Content Command Center - Prompt Pack

Build a complete content creator dashboard with analytics, AI insights, kanban-style idea tracking, and a calendar for planning your posts.

45 prompts across 8 phases. At the end you have an application that pulls live metrics from YouTube, TikTok, and Instagram into your own database, runs Claude over your actual performance data, holds your ideas on a kanban board, and schedules them on a calendar.


## What you need before starting

**A coding agent.** This project is intended to be built by a coding agent. You will need an agent that writes files to your computer and runs commands, such as Claude Code or Cursor. A chat-only assistant CANNOT execute this pack.

**API Accounts.** Free to create, all four:

- Anthropic, for the AI features. Requires a payment method; the API is paid per request.
- YouTube Account.
- Instagram Developer Account.
- TikTok Developer Account.

**Software.** Node.js 20+, PostgreSQL 14+, Git, and the GitHub CLI. Task `P1.0` checks for these and installs what is missing, so you do not need to install anything in advance.

## How to run it

1. At the top of this page, click **Use this template → Create a new repository**. Give it a name such as `content-command-center` and choose **Private**. This is your own copy; the application is built in it.
2. Clone your new repository and open your coding agent in that folder:

   ```bash
   git clone https://github.com/<your-username>/<your-repo-name>.git
   cd <your-repo-name>
   ```

3. Give the agent the run prompt from `RUN.md`.
4. When the phase reports its checkpoint, start a new session and give the same prompt naming phase 2. Repeat through phase 8.

One phase per session, eight sessions. The agent works through all 45 tasks in order, verifying each one before starting the next.

## The one thing it needs from you

Developer accounts cannot be created programmatically. At task `P1.2` the agent stops, writes a file called `SETUP.md` with exact instructions, and waits.

In one sitting you:

- Sign in to the GitHub CLI.
- Create the four developer accounts.
- Obtain seven credential values.
- Paste them into `.env.credentials.local`, a file the agent has already created and confirmed Git will never upload.

Three further keys sit in that file for TikTok's access token, refresh token, and expiry. Leave them empty; the agent fills them at `P1.3` by exchanging the authorization code, which is the seventh value you supply.

Then you tell the agent to continue. It validates every credential against the live service and builds the rest of the application without stopping again.

Expect this to take one to two hours, most of it spent in developer consoles. Nothing else in the build requires you.

## Customizing it

Every task cites `SPEC.md` rather than restating it. To change what gets built, edit `SPEC.md` before starting and the tasks follow the change. This is how you would swap TikTok for LinkedIn, or add a field to the schema.

## A note on security

The application is built to run locally on your own computer and has no login screen, by design. Your platform tokens live in its database, so anyone who can reach the settings page can read or remove them. Do not put it on a public address without adding authentication first. `phase-8-ship.md` covers what that would take.

Your credentials live in `.env.credentials.local` and `.env.local`. Both are ignored by Git, along with `SETUP.md`, and `P1.1` verifies that before any of the three exists.

## If something fails

The agent stops at the failed task rather than building on top of it. `troubleshooting.md` maps symptoms to causes for every phase. Give the agent the exact error text and the task ID that failed.
