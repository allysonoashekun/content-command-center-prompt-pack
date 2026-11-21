# Content Command Center - Specification

This document describes the application the prompt pack builds. It is the source of truth. Every prompt in the pack cites sections of this file rather than restating requirements, so editing this file changes what the prompts build.

This file belongs at the root of your repository. `P1.1` leaves it there, and every later task cites it at that path. Do not rename it.

**Scope:** A single-user application, run locally. There is no authentication layer, by design. See [Key Decisions](#key-decisions).

## Contents

The sections run in build order: first what the application is configured with, then each screen and mechanism in the order the pack builds it, then the reference material the rest of the document rests on.

**Configuration** - [Pinned Versions](#pinned-versions), [Dependencies](#dependencies), [Environment Variables](#environment-variables)

**Application** - [Shell & Navigation](#shell--navigation), [Drafts & Kanban](#drafts--kanban), [Scratchpad](#scratchpad), [Content Editor](#content-editor), [Platform Variants](#platform-variants), [Calendar](#calendar), [Platform Sync](#platform-sync), [Settings](#settings), [Analytics Dashboard](#analytics-dashboard), [Post Detail](#post-detail), [Full Analysis](#full-analysis), [AI Insights](#ai-insights), [Cross-cutting](#cross-cutting)

**Reference** - [Database Schema](#database-schema), [File Structure](#file-structure), [Key Decisions](#key-decisions)

---

## Pinned Versions

Every version the build depends on is recorded here. To move the project to newer versions, change them here and rerun the affected prompts.

| Component | Version |
|-----------|---------|
| Node.js | 20 or later |
| PostgreSQL | 14 or later |
| Next.js | 16.2.9 |
| React | 19.2.4 |
| TypeScript | 5.x |
| Tailwind CSS | 4.x |
| Drizzle ORM | 0.45.x |
| AI model | `claude-sonnet-5` |

Package versions may have moved on since this pack was published. If your agent installs something newer and the API differs from what a prompt describes, the rule in `AGENTS.md` applies: read the documentation shipped inside `node_modules` for the version actually installed, and follow that rather than the remembered API.

## Dependencies

Runtime:

```
drizzle-orm  postgres  @neondatabase/serverless
@dnd-kit/core  @dnd-kit/sortable  @dnd-kit/utilities
@tiptap/react  @tiptap/pm  @tiptap/starter-kit
@tiptap/extension-placeholder  @tiptap/extension-image
@tiptap/extension-link  @tiptap/extension-character-count
@tiptap/extension-underline
recharts  date-fns  lucide-react  next-themes
react-hook-form  @hookform/resolvers  zod
react-markdown  remark-gfm  sonner
@anthropic-ai/sdk
class-variance-authority  clsx  tailwind-merge
```

Development:

```
drizzle-kit  dotenv  vitest  @vitejs/plugin-react
vite-tsconfig-paths  @tailwindcss/postcss  eslint  eslint-config-next
```

shadcn/ui components to add:

```
badge  button  card  dialog  dropdown-menu  input  label
select  separator  sheet  skeleton  sonner  table  tabs  textarea
```

## Environment Variables

```
DATABASE_URL=
ANTHROPIC_API_KEY=

TIKTOK_CLIENT_KEY=
TIKTOK_CLIENT_SECRET=
```

The TikTok client key and secret are used only to refresh TikTok's 24-hour access tokens. Every other platform credential - the YouTube API key and channel ID, the Instagram access token, the TikTok access token - is entered through the Settings page and stored in the database, not here.

---

## Shell & Navigation

- **Collapsible sidebar** - toggles between an expanded and a collapsed width with a short transition. Collapsed mode hides labels and shows a title attribute on each item. The state persists in `localStorage` under the key `sidebar-collapsed`.
- **Four navigation items** - Analytics, Drafts, Calendar, Settings. The active item is matched by prefix, so `/analytics/posts/123` keeps Analytics highlighted.
- **Per-page TopNav** - a header with a title and an optional right-side actions slot. Each page renders its own; the layout does not.
- **Toasts** - mounted once in the dashboard layout. All success and failure feedback flows through it.
- **Root redirect** - `/` redirects to `/analytics`.

## Drafts & Kanban

Route: `/drafts`

### The two status axes

A content piece carries two independent state fields.

| Field | Values | Purpose | How it changes |
|-------|--------|---------|----------------|
| `category` | `backlog`, `idea`, `in_progress`, `complete` | Which Kanban lane the piece sits in | Dragging a card between lanes; the repurpose flow files new ideas into `backlog` |
| `status` | `idea`, `draft`, `draft_complete`, `filming`, `filming_complete`, `editing`, `editing_complete`, `scheduled`, `posted` | Production stage, shown as a badge | The status select in the editor header; a calendar drag sets `scheduled`; linking a synced post sets `posted` |

`category` decides placement. `status` decides what the badge says. A card never displays its own category, because the lane conveys it.

### Board

- **Four horizontal swimlanes** in category order. Each lane header shows a live count, each lane body scrolls independently and highlights while a card hovers over it.
- **Compact content rows** - hover-revealed drag handle, title, series tag, scheduled date rendered only when `status` is `scheduled`, per-platform chips for existing variants, status badge, hover-revealed delete.
- **Drag to recategorize** - 8px activation distance so clicks are not mistaken for drags. Dropping on a lane updates `category` optimistically and rolls back with a toast on failure. A drag overlay renders a floating copy.
- **Click-through** - clicking a row opens `/drafts/[id]`. The drag handle and delete control are excluded.
- **Delete with confirmation** - a dialog naming the piece and warning that its platform variants go too. Optimistic removal, restored on failure.
- **New Content** - creates a piece titled "Untitled" and opens its editor. No intermediate dialog.
- **Loading state** - per-category skeletons. Empty lanes read "Drop here".

## Scratchpad

- A persistent freeform notepad in the right pane of `/drafts`, beside the board.
- A full editor sharing the same toolbar as the script editor.
- **A single global row**, created automatically on first read. One scratchpad for the whole application.
- **Autosave** 800ms after the last change, stored as stringified editor JSON.
- **Save-state indicator** reading Saved, Saving, or Unsaved.
- **Legacy migration** - content that is not valid editor JSON is treated as older plain text and converted on load.

## Content Editor

Route: `/drafts/[id]`

### Header controls

Each saves independently. Back link, editable title, series tag, scheduled date, status select.

- **Editable title** - a borderless inline input that saves on blur, skips unchanged values, trims, and falls back to "Untitled" when emptied.
- **Series tag** - free text. Clearing it writes `null`.
- **Scheduled date** - a native date input, built from local year, month, and day to avoid the UTC off-by-one.
- **Status select** - all nine production statuses, optimistic with rollback.

### Script editor

- Tiptap with StarterKit, Placeholder, Image, Link, Underline, and CharacterCount. Content persists as editor JSON in `content_pieces.script_notes`.
- **Autosave** 800ms after the last change.
- **Save indicator** - color-coded Saving, Saved, or Unsaved in the footer.
- **Live counts** - words and characters in the footer.
- **Shared toolbar** - bold, italic, underline; heading 1 and 2; bullet list, ordered list, blockquote; link and image collected with `window.prompt`; undo and redo, disabled when unavailable.
- **ScriptViewer** - a read-only renderer used to show a linked script on the post detail page.

## Platform Variants

- **Add-then-tab** - an Add button for each platform without a variant. A tab appears once the variant exists and the Add button disappears.
- **Per-tab remove** - a small control on each tab deletes that variant and its saved data.
- **Autosave** - each form watches the whole form and upserts 800ms after the last change.
- **YouTube** - Title with a 110-character limit and live counter, Description, and Tags entered comma-separated and stored as an array.
- **TikTok** - Caption with a 2200-character limit and counter, Hashtags separated by spaces or commas with any leading `#` stripped, and an optional Sound name.
- **Instagram** - Caption with a 2200-character limit and counter, Media URLs one per line, and optional Alt text.

`platform_data` is replaced wholesale on save, not merged, so each form sends all of its fields every time.

## Calendar

Route: `/calendar`

- **Month grid** - Monday-first, spanning whole weeks, up to six rows of seven columns, vertically scrollable.
- **Navigation** - a Today button and chevrons stepping one month at a time. The label reads month and full year.
- **Drag to reschedule** - each day is a drop target keyed `yyyy-MM-dd`. Dropping updates `scheduledFor` and sets `status` to `scheduled`, optimistically, rolling back both fields with a toast on failure.
- **Click-through** - clicking a card opens `/drafts/[id]`.
- **Visual cues** - out-of-month cells are muted, the week row containing today is tinted, today's date number is in a filled circle, and the hovered drop target highlights.
- **Overflow** - up to three cards per day, then a `+N more` count. Cards within a day sort by scheduled time.

Time of day is not part of a drop; the date lands at local midnight.

## Platform Sync

`POST /api/analytics/sync`, same-origin guarded, with an extended maximum duration.

Sync pulls the entire published catalog from each connected platform into `platform_posts` and records a daily metrics snapshot for every post. Linking a post back to a drafted piece is optional and done afterward. Posts never drafted in the application are still fully tracked and can still be analyzed.

Only JSON metadata and numeric counts are stored, never media. `permalink` and `thumbnail_url` are kept as URLs, so thumbnails load from the platform when rendered. A post costs roughly 1 to 4 KB.

### Per platform

**YouTube** - authenticates with a Data API v3 API key. API keys read public data only and cannot identify the caller, so the channel ID is supplied explicitly in Settings.
- `validateCredentials(apiKey, channelId)` reads the channel with `part=snippet,statistics`.
- The catalog is read by resolving the channel's uploads playlist and paging `playlistItems` at 50 per page, then fetching `videos?part=statistics` in chunks of 50 ids. `search.list` is not used: it caps at roughly 500 results and costs 100 quota units per request against 1 for `playlistItems`.
- Publish time comes from `contentDetails.videoPublishedAt`, not `snippet.publishedAt`, which is when the video was added to the playlist.
- Metrics: views, likes, comments. Reach is proxied by view count, recorded as `reachProxy` in `platform_metrics`.

**Instagram** - host `https://graph.instagram.com`. Requires a Business or Creator account.
- `validateCredentials(accessToken)` reads `/me?fields=id,username,account_type`.
- Media paged via `paging.cursors.after` at 100 per page. Per-media insights fetched through a concurrency pool of 4.
- `GET /{mediaId}/insights?metric=reach,likes,comments,shares,saved,views`. `impressions` is deprecated by Meta in favor of `views`, so that column stays 0.
- `refreshLongLivedToken(token)` calls `/refresh_access_token?grant_type=ig_refresh_token`. No app secret is needed at runtime.

**TikTok** - host `https://open.tiktokapis.com`.
- `validateCredentials(accessToken)` reads `/v2/user/info/`.
- `POST /v2/video/list/` with `max_count` and a cursor, 20 per page, returning metadata and metrics together.
- `refreshAccessToken(refreshToken)` posts to `/v2/oauth/token/` and returns a rotated refresh token that must be persisted.
- Reach is not available in the Display API; view count is used as a proxy. Saves are not reported.

All three guard against a cursor or page token that does not advance, and enforce a hard page limit.

### Orchestration

For each stored credential:

1. Refresh the token if it expires within 7 days, persisting the new access token, expiry, and any rotated refresh token.
2. Fetch the full paginated catalog.
3. Upsert each post on `(platform, external_id)`, preserving any existing `content_variant_id`, counting new against updated.
4. Upsert today's snapshot keyed on `(platform_post_id, day)`, so syncing twice in a day overwrites rather than duplicates.
5. Stamp `last_synced_at`.

Platforms run under `Promise.allSettled`, so one failing never blocks the others. The response is `{ platforms: [{ platform, new, updated, total }], totals, errors: [{ platform, message }], newPosts: [...] }`.

**Sync modal** - a spinner while running, then a summary with totals, per-platform new and updated counts, and inline errors for any platform that failed. Run in background dismisses the modal while the sync continues, toasting on completion. Newly discovered posts are listed with inline link-to-draft controls. The request is held by the page, not the modal, so dismissing it does not cancel the sync. After a sync the date range is re-resolved before refetching.

### Association

`PATCH /api/platform-posts/[id]` with `{ contentPieceId }` links a post: it finds or creates the variant for that post's platform on the chosen piece, releases any other post claiming that variant, attaches the post, and sets the piece's status to `posted`. `{ contentPieceId: null }` unlinks and leaves the piece untouched.

## Settings

Route: `/settings`

Platform credentials are the only thing configurable.

- **YouTube** - API key, which reads public statistics and never expires, plus Channel ID.
- **Instagram** - long-lived access token and an optional refresh token. Requires a Business or Creator account.
- **TikTok** - access token, optional refresh token, and an optional expires-in value in seconds, converted to an absolute timestamp before saving. TikTok tokens last 24 hours.
- **Live validation** - saving calls the real platform API first. A rejected token surfaces the platform's own error message and nothing is stored.
- **Status display** - a connected badge showing the account identity returned by the platform, last-synced as a relative time or "never", and token expiry with a note that it auto-refreshes on sync when a refresh token is on file.
- **Remove** - disconnects the platform.

Tokens are never returned by the API. Reads expose metadata and a `hasRefreshToken` boolean only.

Instagram is assigned a 60-day expiry when none is supplied, because its card has no expires-in field and auto-refresh would otherwise never fire.

## Analytics Dashboard

Route: `/analytics`

- **Platform tabs** - All, YouTube, TikTok, Instagram. The active tab drives every query and decides which charts render.
- **Date range** - last 7, 30 (default), or 90 days, or a custom from-and-to pair.
- **Sync button** - opens the sync modal and pulls fresh data.
- **Platform metric cards** - one per platform with data in range, leading with views growth and engagement growth against the preceding equal-length window, shown with direction and color and rendered as a dash when there is no prior data, plus average engagement and post count.
- **Posting cadence card** - posts in range and average posts per week. Shown only when a single platform is selected.
- **Engagement chart** - a line per platform, engagement rate over time.
- **Views chart** - on the All tab, daily summed views per platform.
- **Views-per-post chart** - on a single platform tab, one bar per post ordered by publish date, each split into an engagement segment and remaining views. Clicking a bar runs an AI analysis of that post. Outliers are handled by capping the axis at the 95th percentile; clipped bars get a torn top edge and a true-value label, and the header explains the cap.
- **Breakout posts** - up to five recent posts at 1.5 times or more of their platform's average views or engagement, badged with the multiple, linking to the post and to the platform permalink, each with an Analyze action.
- **Top insights** - a carousel of the three headline action items from the most recent approved content gap analysis, with dot pagination and a link to the full analysis.
- **Posts table** - the synced catalog. Sortable on every column except platform, a platform multi-select filter in the header, a sticky header over a scrolling list with a maximum height, thumbnails, links to post detail and to the platform, a linked badge jumping to the associated draft, and per-row Analyze and Link actions.
- **Loading and empty states** - a skeleton layout while fetching, and "No data yet - connect a platform in Settings and run a sync" when there is nothing.

### Aggregation

Pure helpers in `src/lib/analytics/aggregate.ts`, which imports nothing from the database layer, wrapped by `src/lib/db/queries/analytics.ts` around a shared `loadSnapshots` join.

- `loadSnapshots` joins snapshots to posts to variants to pieces, filtered by platform and snapshot date. Titles fall back from the post title, to an 80-character caption snippet, to "Untitled post", because Instagram and TikTok posts have no title.
- `selectLatestPerPost` - current metrics are always the newest snapshot per post. Everything except the time series builds on this.
- `aggregateEngagementOverTime` and `aggregateViewsOverTime` - bucket by UTC day, averaging engagement and summing views per platform.
- `summarizePlatforms(current, previous)` - totals, averages, and percent-change deltas against the preceding equal-length window. Deltas are null when there is no prior data, which is distinct from zero. Platforms with no rows are omitted.
- `computeBreakoutPosts` - flags posts at 1.5 times or more of their platform's average views or engagement, ranked by the larger multiple, limit five. Eligibility filters on `published_at`, not snapshot date, because an old post re-synced today still has a snapshot dated today.
- `computePostingCadence` - posts published in range, plus posts per week.
- `sortPostMetrics` - non-mutating, type-aware sort shared by the API and the table.

### Chart scaling

`src/lib/analytics/chart-scale.ts` caps a chart axis at the 95th percentile with 5 percent headroom, but only when there are at least six points and the peak exceeds the percentile by more than three times. The percentile is taken at the nearest rank rather than interpolated. `niceCeil` rounds to a readable bound. If rounding would leave nothing clipped, it falls back to a full-range axis. Returns `{ max, clamped, clippedCount }`, which drives the chart subtitle, the torn bar tops, and the tooltip note.

### Analytics API

`GET /api/analytics` with Zod-validated `platform`, `dateFrom`, `dateTo`, `view`, `sortBy`, and `sortDir`.

- `view=overview` returns `{ summaries, engagement, views, breakouts, cadence }`, queried in parallel.
- `view=posts` returns `{ posts }`, sorted server-side.

An invalid parameter returns 400.

## Post Detail

Route: `/analytics/posts/[id]`

- **Header** - thumbnail, title, permalink, platform badge, publish date, and the linked draft when there is one.
- **Metrics grid** - views, likes, comments, shares, saves, reach, engagement rate, and watch time when present, labeled with the snapshot date. Saves are hidden for TikTok, which does not report them. A metric the platform did not provide is hidden rather than shown as zero.
- **Latest analysis** - the most recent performance analysis with its model and token counts, and Approve or Dismiss while pending. A Generate analysis action when there is none.
- **Script panel** - a read-only render of the linked draft's script, or a link-to-draft control when unlinked.
- **Analyze** and **Repurpose** actions. Repurpose is enabled only when the post is linked to a draft that has a script.

## Full Analysis

Route: `/analytics/analysis`

- **Platform tabs**, with the initial tab taken from a `platform` query parameter.
- **Time range filter** - last week, last month (default), last two months, last six months, or all time.
- **Show or hide dismissed** toggle.
- Renders the most recent approved weekly brief and content gap analysis, each with model, token counts, and review actions, followed by the history for the selected filters.

## AI Insights

| Type | Trigger | Mode |
|------|---------|------|
| `performance_analysis` | Analyze on a post row, breakout card, chart bar, or post detail | Streaming |
| `content_gap` | Content gaps button on Analytics | Streaming |
| `weekly_brief` | Weekly brief button on Analytics | Streaming |
| `caption_hook_hashtag` | Generate captions in the editor's suggestion panel | Structured JSON |
| Repurpose | Repurpose on post detail, enabled only when the post is linked to a draft that has a script | Structured JSON |

The first four are `suggestion_type` values written to `ai_suggestions`. Repurpose results become `backlog` content pieces, which are the durable artifact, and the response itself is discarded.

- **Claude client** - `src/lib/ai/client.ts` exposes a lazily constructed `getAnthropic()`. Eager construction would throw at import time when `ANTHROPIC_API_KEY` is unset and break unrelated route modules.
- **Prompts** - `src/lib/ai/prompts.ts` holds `AI_MODEL`, defined here so client components can display it without bundling the SDK, and a frozen `SYSTEM_PROMPT` emitted as a single `cache_control: { type: "ephemeral" }` block on every request. Extended thinking is deliberately off so text streams immediately. Zod schemas validate each request's context before it reaches the model. `buildStreamingPrompt` and `buildStructuredPrompt` dispatch by type.
- **`POST /api/ai/stream`** - inserts a `pending` suggestion row before streaming so its id can be returned in an `X-Suggestion-Id` header. Streams text deltas as plain text, filling content and token counts on the final message. For `performance_analysis` with a linked piece it injects the piece's title, concept, and a 4000-character script summary. For `content_gap` it extracts a trailing "Top 3 insights" section into `actionItems`. A failure appends a visible marker to the stream.
- **`POST /api/ai/suggestions`** - structured output against a JSON schema, returning the saved record with 201. **`GET`** lists suggestions filtered by `contentPieceId`, `contentVariantId`, `status`, `type`, or `platform`, where `platform=none` means a NULL platform, distinct from omitting the parameter.
- **`PATCH /api/ai/suggestions/[id]`** - sets `approved` or `dismissed` plus `reviewedAt`. 404 when nothing matches. Approving records a decision; it does not copy content anywhere.
- **`POST /api/ai/repurpose`** - reads the script server-side from the content piece, flattens it, truncates to 6000 characters, and returns `{ ideas: [{ title, angle, description, outline }] }`. 400 when the draft has no script. Results are deliberately not persisted.
- **`StreamingText`** - decodes a stream incrementally with a blinking cursor, handling React StrictMode double-mounts via a `stream.locked` guard and releasing the reader on cleanup.
- **`MarkdownBody`** - markdown with GFM support, including tables in their own scroll container.
- **`SuggestionCard`** - headline, markdown body, action items, status badge once reviewed, and a footer with the model and total token count. Approve and Dismiss appear only while pending.
- **`AISuggestionSheet`** - a centered dialog showing a spinner while draining the stream, then the finished card with a pinned Approve and Dismiss footer. The close control is hidden while streaming.
- **`SuggestionPanel`** - the editor's right rail. Generate captions fires one request per existing variant in parallel, and prompts to add a variant first when there are none. Suggestions and History tabs split pending from reviewed with live counts.
- **Repurpose** - returns 4 to 6 new video ideas reusing the subject matter with a changed angle, point of view, or format. All ideas are pre-selected. Each saved idea becomes a `backlog` content piece tagged "Repurposed", with its script pre-seeded as a description paragraph, an "Outline" heading, and the beats as a bullet list. Partial failures are tolerated and reported.

## Cross-cutting

- **CSRF guard** - every mutating route runs `checkSameOrigin`, rejecting requests whose `Origin` host does not match the request `Host`. A request with no `Origin` is allowed, so command line tools keep working.
- **No auth** - single user, local only, by design.
- **Tests** - beside the code they cover: the platform modules, the shared platform helpers, the aggregation helpers, the chart scaling, and the prompt builders. No test opens a network or database connection.

---

## Database Schema

### Enums

```
content_status:    idea | draft | draft_complete | filming | filming_complete | editing | editing_complete | scheduled | posted
content_category:  backlog | idea | in_progress | complete
platform:          youtube | tiktok | instagram
suggestion_type:   performance_analysis | content_gap | caption_hook_hashtag | best_posting_time | weekly_brief
suggestion_status: pending | approved | dismissed
```

### content_pieces

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | defaultRandom |
| title | text NOT NULL | |
| concept | text | short pitch or angle; set by the repurpose flow |
| series_tag | text | grouping label |
| script_notes | jsonb | Tiptap editor JSON |
| status | content_status NOT NULL | default `idea` |
| category | content_category NOT NULL | default `idea` |
| scheduled_for | timestamptz | |
| published_at | timestamptz | |
| created_at / updated_at | timestamptz NOT NULL | defaultNow |
| INDEX | | on status, category, scheduled_for |

### content_variants

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| content_piece_id | uuid FK to content_pieces | cascade delete |
| platform | platform NOT NULL | |
| platform_data | jsonb NOT NULL | default `{}` |
| published_at | timestamptz | |
| created_at / updated_at | timestamptz NOT NULL | |
| INDEX | | on content_piece_id, platform |
| UNIQUE | (content_piece_id, platform) | one variant per platform per piece |

### platform_credentials

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| platform | platform NOT NULL UNIQUE | one row per platform |
| access_token | text NOT NULL | |
| refresh_token | text | |
| token_expires_at | timestamptz | |
| metadata | jsonb NOT NULL | channelId/channelName, userId/username, openId/displayName |
| last_synced_at | timestamptz | |
| created_at / updated_at | timestamptz NOT NULL | |

### platform_posts

The published catalog pulled by sync.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| platform | platform NOT NULL | |
| external_id | text NOT NULL | the platform's post id |
| title | text | |
| caption | text | |
| permalink | text | URL |
| thumbnail_url | text | URL; the image is never downloaded |
| published_at | timestamptz | |
| content_variant_id | uuid FK to content_variants, nullable, UNIQUE | optional association; set null on delete |
| last_fetched_at | timestamptz | |
| created_at / updated_at | timestamptz NOT NULL | |
| INDEX | (platform) | |
| UNIQUE | (platform, external_id) | |

### analytics_snapshots

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| platform_post_id | uuid FK to platform_posts NOT NULL | cascade delete |
| platform | platform NOT NULL | denormalized for query performance |
| snapshot_date | timestamptz NOT NULL | one row per post per day |
| views / likes / comments / shares / saves / reach / impressions | integer | default 0 |
| engagement_rate | real | (likes + comments + shares + saves) / reach * 100 |
| watch_time_seconds | integer | YouTube and TikTok |
| platform_metrics | jsonb | platform-specific extras, such as `reachProxy` |
| created_at | timestamptz NOT NULL | |
| INDEX | (platform_post_id, snapshot_date) | |

### scratchpad

A single global row, created on first read.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| content | text NOT NULL | default `""`; stringified editor JSON |
| created_at / updated_at | timestamptz NOT NULL | |

### ai_suggestions

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| type | suggestion_type NOT NULL | |
| content_piece_id | uuid FK nullable | set null on delete |
| content_variant_id | uuid FK nullable | set null on delete |
| platform_post_id | uuid FK nullable | set when the suggestion targets a synced post |
| platform | platform nullable | the tab it was generated from; NULL means unscoped |
| content | jsonb NOT NULL | `{ headline, body, actionItems? }` |
| status | suggestion_status NOT NULL | default `pending` |
| input_tokens / output_tokens | integer | cost tracking |
| model | text | |
| created_at | timestamptz NOT NULL | |
| reviewed_at | timestamptz | set on approve or dismiss |
| INDEX | | on status, type, platform |

---

## File Structure

```
├── drizzle.config.ts
├── vitest.config.ts
├── components.json
├── .env.local                          # gitignored
├── .env.example
├── src/
│   ├── app/
│   │   ├── layout.tsx                  # root: fonts, metadata
│   │   ├── globals.css                 # design tokens, platform colors
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx              # sidebar + main + toaster
│   │   │   ├── page.tsx                # redirect to /analytics
│   │   │   ├── analytics/
│   │   │   │   ├── page.tsx
│   │   │   │   ├── analysis/page.tsx
│   │   │   │   └── posts/[id]/page.tsx
│   │   │   ├── calendar/page.tsx
│   │   │   ├── drafts/
│   │   │   │   ├── page.tsx            # kanban + scratchpad
│   │   │   │   └── [id]/page.tsx       # content piece editor
│   │   │   └── settings/page.tsx
│   │   └── api/
│   │       ├── ai/
│   │       │   ├── repurpose/route.ts
│   │       │   ├── stream/route.ts
│   │       │   └── suggestions/
│   │       │       ├── route.ts
│   │       │       └── [id]/route.ts
│   │       ├── analytics/
│   │       │   ├── route.ts
│   │       │   └── sync/route.ts
│   │       ├── content/
│   │       │   ├── route.ts
│   │       │   └── [id]/
│   │       │       ├── route.ts
│   │       │       └── variants/route.ts
│   │       ├── platform-posts/[id]/route.ts
│   │       ├── scratchpad/route.ts
│   │       └── settings/credentials/
│   │           ├── route.ts
│   │           └── [platform]/route.ts
│   ├── components/
│   │   ├── ui/                         # shadcn primitives
│   │   ├── layout/                     # Sidebar, TopNav
│   │   ├── analytics/                  # dashboard, charts, cards, table, detail
│   │   ├── calendar/                   # CalendarView, MonthView, card
│   │   ├── drafts/                     # board, lane, row, new, scratchpad
│   │   ├── editor/                     # Tiptap, toolbar, header, title, viewer
│   │   ├── variants/                   # VariantEditor + one per platform
│   │   ├── ai/                         # sheet, card, panel, streaming, markdown, repurpose
│   │   └── settings/                   # SettingsClient, PlatformCredentialCard
│   ├── lib/
│   │   ├── db/                         # index.ts, schema.ts, queries/analytics.ts
│   │   ├── analytics/                  # aggregate.ts, chart-scale.ts
│   │   ├── platforms/                  # youtube, instagram, tiktok, types, test-utils
│   │   ├── ai/                         # client.ts, prompts.ts
│   │   ├── editor/prosemirror.ts
│   │   ├── http/same-origin.ts
│   │   └── utils.ts
│   ├── hooks/useAISuggestions.ts
│   └── types/index.ts                  # statuses, categories, platform labels
```

---

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Auth | None | Local-only personal application; no public exposure |
| CSRF | Same-origin header check on mutating routes | Cheap protection without an auth system |
| Status model | Two axes: `category` and `status` | Board placement and production stage change independently |
| Kanban layout | Horizontal swimlanes by category | Keeps the nine production statuses off the board axis |
| Platform tokens | Stored in the database, entered via Settings | Tokens rotate; configuration does not |
| YouTube auth | API key plus explicit channel ID | Public metrics only; keys cannot identify the caller and never expire |
| TikTok reach | View count proxy | Reach is not exposed by the Display API |
| Instagram impressions | Left at 0 | Deprecated by Meta in favor of views |
| Sync model | Pull the full catalog into `platform_posts` | Linking a post to a draft is optional and retroactive; undrafted posts are still tracked |
| Background sync | None; on demand | Avoids scheduling complexity at personal scale |
| Editor storage | Editor JSON | Preserves editor state; no markup to execute |
| Platform variant data | `jsonb` per variant | No migration needed to add platform-specific fields |
| AI mode | Suggest-only | A human approves every change; no autonomous mutations |
| Streaming | Long-form suggestions only | Short structured outputs do not benefit |
| Repurpose output | Not persisted as suggestions | The saved content pieces are the durable artifact |
| Scratchpad | Single global row | One user, one notepad |
