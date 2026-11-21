# Phase 6: Analytics Dashboard

## What this phase delivers

The dashboard: performance across every platform in one place, growth against the previous period, charts over time, breakout detection, and a sortable catalog of everything published.

The work is layered deliberately. Calculations are written first as pure functions with no database code, so they can be tested alone. Then the queries that feed them, then the API, then the page. Each layer is verifiable before the next is built, which matters more here than anywhere else in the pack, because a wrong number on a chart looks exactly like a right one.

**No AI features are built in this phase.** The Analyze actions, the Top Insights carousel, and the analysis panel belong to phase 7, and the prompts here say so explicitly.

If the connected accounts are new or small, the dashboard will be sparse and some charts will have too few points to cap an axis. That is correct behavior. Run `seed-data.md` first if a full dashboard is needed to verify this phase.

---

### P6.1 - Build the analytics calculations

Requires: P5.7
Halts: no

OBJECTIVE

Pure functions turning daily metric snapshots into everything the dashboard shows, importing nothing from the database layer, with unit tests.

PROMPT

```text
--- PROMPT P6.1 ---
Read SPEC.md section "Analytics Dashboard".

Create src/lib/analytics/aggregate.ts. It must import nothing from the
database layer. Every function takes data as an argument and returns a
result.

Define a JoinedSnapshot type describing one snapshot joined to its post
and, when linked, the draft behind it. Then implement:

- selectLatestPerPost: reduce many snapshots to the most recent one per
  post. Current metrics are always the newest snapshot; everything
  except the time series builds on this.
- aggregateEngagementOverTime: one point per calendar day, averaging
  each platform's engagement rate across posts snapshotted that day.
- aggregateViewsOverTime: one point per calendar day, summing each
  platform's views. Bucket both by UTC day, consistently.
- summarizePlatforms(current, previous): totals, averages, and the
  percent change against the preceding equal-length window. A delta is
  null when there is no prior data, which is different from a delta of
  zero. Omit platforms with no rows entirely rather than showing zeros.
- toPostMetricRows: flatten joined snapshots into display rows.
- sortPostMetrics: sort by any column, ascending or descending, without
  mutating the input, comparing text and numbers appropriately.
- computeBreakoutPosts: posts at or above 1.5 times their own platform's
  average views or average engagement rate, ranked by whichever multiple
  is larger, limited to five. Eligibility is filtered on publishedAt,
  not on snapshot date: an old post re-synced today still has a snapshot
  dated today, so filtering by snapshot date would call an old post
  recent. A post with no publishedAt is not eligible.
- computePostingCadence: posts published within the range, and that
  count expressed per week. The per-week rate is null unless both ends
  of the range are known.

Create src/lib/analytics/chart-scale.ts:

- niceCeil: round a number up to a readable axis bound.
- computeAxisCap(values): return { max, clamped, clippedCount }. Cap the
  axis at the 95th percentile with a little headroom, but only when
  there are at least 6 values and the peak exceeds that percentile by
  more than 3 times. Take the percentile at the nearest rank rather than
  interpolating, since interpolation drags the outlier back in. If
  rounding pushes the cap past the peak so nothing would be clipped,
  return the full-range axis instead.

Set up Vitest if it is not already configured, and write unit tests
beside both files covering: the latest-snapshot reduction, a null delta
against an absent previous period, breakout eligibility by publish date,
a cap that triggers, and a spread that is uneven but not extreme enough
to cap.

Do not create database queries, API routes, or components.
--- END PROMPT P6.1 ---
```

EXPECTED RESULT

Two source files, two test files, and a passing test run.

VERIFICATION

1. `npm test` exits 0 with every test passing.
2. Grep the import lines of `aggregate.ts` and `chart-scale.ts` and confirm neither imports anything from `src/lib/db`. A single database import here makes these functions untestable in isolation and unusable in browser code.
3. Confirm a test exists asserting a null delta against an absent previous window, distinct from a delta of zero.
4. Confirm a test exists asserting breakout eligibility is filtered on `publishedAt` rather than snapshot date.
5. Confirm one test triggers the axis cap and one has an uneven spread that correctly does not cap.

GATE: Checks 1 and 2 pass.

---

### P6.2 - Build the analytics database queries

Requires: P6.1
Halts: no

OBJECTIVE

The query layer that loads snapshots and delegates every calculation to `aggregate.ts`.

PROMPT

```text
--- PROMPT P6.2 ---
Read SPEC.md section "Analytics Dashboard".

Create src/lib/db/queries/analytics.ts.

1. loadSnapshots(filters): one shared query joining analytics_snapshots
   to platform_posts, then left-joining content_variants and
   content_pieces so a linked draft comes back with the row. Filter by
   platform and by snapshot date range when given.

   For the display title, fall back in this order: the post title, then
   a caption snippet truncated to 80 characters, then "Untitled post".
   Instagram and TikTok posts have captions rather than titles, so
   without this fallback most rows would be blank.

2. Thin wrappers, each loading snapshots and passing them to the
   matching pure function from aggregate.ts:
   getEngagementOverTime, getViewsOverTime, getBreakoutPosts,
   getPostingCadence, getAllPostMetrics.

3. getPlatformSummaries: load the current window and reduce it to the
   latest snapshot per post, then load the immediately preceding window
   of equal length and reduce it the same way, and pass both to
   summarizePlatforms so it can compute the deltas.

Keep all calculation in aggregate.ts. This file loads data and delegates.
--- END PROMPT P6.2 ---
```

EXPECTED RESULT

One new file. Nothing calls it yet.

VERIFICATION

1. Call `getAllPostMetrics` with no filters. Report the row count and five sample rows with titles and view counts. No row may have a blank title. A blank Instagram or TikTok title means the caption fallback is missing.
2. Call `getPlatformSummaries` for the last 30 days. Report whether each delta is a number or null. If every delta is null, the previous window is not being loaded.
3. Confirm this file contains no arithmetic that belongs in `aggregate.ts`; each wrapper loads and delegates.
4. `npm run build` exits 0.

GATE: Check 1 returns no blank titles.

---

### P6.3 - Build the analytics API

Requires: P6.2
Halts: no

OBJECTIVE

One read-only route serving the dashboard in two modes: overview and post catalog.

PROMPT

```text
--- PROMPT P6.3 ---
Read SPEC.md section "Analytics Dashboard".

Create GET /api/analytics.

Validate every query parameter with Zod: platform, dateFrom, dateTo,
view, sortBy, and sortDir. Return 400 with a clear message for an
invalid value rather than letting it reach the database.

view=overview returns { summaries, engagement, views, breakouts, cadence }
with the five queries run in parallel.

view=posts returns { posts }, sorted server-side by the requested column
and direction.

This route only reads data, so it needs no same-origin guard.
--- END PROMPT P6.3 ---
```

EXPECTED RESULT

One new route file.

VERIFICATION

1. `GET /api/analytics?view=overview` returns 200 with all five keys: `summaries`, `engagement`, `views`, `breakouts`, `cadence`.
2. `GET /api/analytics?view=posts&sortBy=views&sortDir=desc` returns posts in descending view order.
3. `GET /api/analytics?platform=myspace` returns 400, not 500. A rejected parameter is a bad request, not a server fault.
4. `GET /api/analytics?view=posts&sortBy=nosuchcolumn` returns 400.
5. Confirm the five overview queries run in parallel rather than sequentially.

GATE: Check 3 returns 400.

---

### P6.4 - Build the dashboard shell and metric cards

Requires: P6.3
Halts: no

OBJECTIVE

The analytics page: platform tabs, date range selection, per-platform growth cards, posting cadence, and breakout posts.

PROMPT

```text
--- PROMPT P6.4 ---
Read SPEC.md section "Analytics Dashboard".

1. Create AnalyticsClient as a client component owning the page state:

   - Platform tabs: All, YouTube, TikTok, Instagram. The active tab
     drives every request.
   - Date range: last 7, 30, or 90 days, defaulting to 30, plus a custom
     from-and-to pair.
   - Fetch both the overview and the post list from /api/analytics
     whenever the platform or range changes.
   - Show a skeleton layout while loading.
   - Show an empty state when there is no data, reading: "No data yet -
     connect a platform in Settings and run a sync".

2. Create PlatformMetricsCard, one per platform that has data in range:

   - Lead with views growth and engagement growth against the preceding
     equal-length window, shown with direction and color, and rendered
     as a dash when there is no prior period to compare against. A dash
     and a zero mean different things and must not look the same.
   - Below those, average engagement rate and post count.

3. Create PostingCadenceCard showing posts in range and average posts
   per week. Render it only when a single platform is selected, since
   cadence across platforms is not a meaningful number.

4. Create BreakoutPosts, listing up to five recent posts that reached at
   least 1.5 times their platform's average, each badged with its
   multiple and linking to the post and to its permalink on the
   platform.

5. Render AnalyticsClient on the /analytics page.

Do not add charts, the post table, the sync button, or any Analyze
action. Later tasks and phase 7 add those.
--- END PROMPT P6.4 ---
```

EXPECTED RESULT

Four new components and an updated analytics page.

VERIFICATION

1. Request `/analytics` and confirm 200.
2. `GET /api/analytics?view=overview&dateFrom=...&dateTo=...` for the last 30 days. Independently compute each platform's total views and post count with a direct SQL query over the latest snapshot per post, and confirm the two agree.
3. Read `PlatformMetricsCard` and confirm a null delta renders as a dash and a zero delta renders as `0%`, through distinct branches. Quote the lines.
4. Confirm `PostingCadenceCard` renders only when a single platform is selected.
5. Confirm the empty state string matches the specification exactly.
6. `npm run build` exits 0.

GATE: Checks 2 and 3 pass. Conflating null and zero is the error this gate catches.

---

### P6.5 - Build the charts

Requires: P6.4
Halts: no

OBJECTIVE

Three charts, with the per-post chart capping its axis visibly rather than letting one outlier flatten every other bar.

PROMPT

```text
--- PROMPT P6.5 ---
Read SPEC.md section "Analytics Dashboard". Use Recharts.

1. EngagementChart: a line per platform, engagement rate over time.
   Shown on every tab.

2. ViewsChart: daily total views per platform. Shown only on the All
   tab.

3. PostViewsChart: one bar per post, ordered by publish date. Shown only
   when a single platform is selected. Each bar is split into an
   engagement segment and the remaining views, so volume and interaction
   are readable in one shape.

   Apply computeAxisCap to the view counts. When the axis is capped:
   - Draw clipped bars with a torn or broken top edge so it is visible
     that the bar continues past the axis.
   - Label each clipped bar with its true value.
   - State in the chart header that the axis is capped and how many
     posts exceed it.

   A chart that silently truncates data is worse than one that does not
   truncate at all, so the capping must be visible.

4. Give each chart an empty state for when there is nothing to plot, and
   use the platform color tokens defined in P2.3 consistently across all
   three.

Do not add click-to-analyze behavior to any chart. Phase 7 adds that.
--- END PROMPT P6.5 ---
```

EXPECTED RESULT

Three new chart components rendered on the analytics page.

VERIFICATION

1. Take the view counts the per-post chart would plot for one platform, run them through `computeAxisCap`, and report `max`, `clamped`, and `clippedCount`. Confirm the component renders a clipped treatment and a true-value label for exactly `clippedCount` bars, and states the cap in its header when `clamped` is true.
2. Confirm `ViewsChart` renders only on the All tab and `PostViewsChart` only when a single platform is selected. Quote the conditions.
3. Confirm all three charts read the platform color tokens defined in `P2.3` rather than hard-coded colors.
4. Confirm each chart has an empty state for zero data points.
5. `npm test` and `npm run build` both exit 0.

GATE: Check 1 shows the chart's clipped rendering matching `computeAxisCap`'s output. A cap applied but not communicated fails this gate.

---

### P6.6 - Build the post catalog and sync

Requires: P6.5
Halts: no

OBJECTIVE

A sortable, filterable table of every published post, and the Sync button and modal that drive the route from `P5.6`.

PROMPT

```text
--- PROMPT P6.6 ---
Read SPEC.md sections "Analytics Dashboard" and "Platform Sync".

1. PerformanceTable: the post catalog.
   - Every column sortable except platform, with text columns
     defaulting to ascending and metrics to descending.
   - A platform multi-select filter in the header, shown on the All tab.
   - A sticky header over a scrolling list with a maximum height.
   - Thumbnails loaded from the platform by URL.
   - Each row links to the post detail page and to the post on its
     platform.
   - A badge on rows linked to a draft, jumping to that draft.

2. LinkToIdeaMenu: a control that loads completed content pieces when
   opened, not before, and links or unlinks through PATCH
   /api/platform-posts/[id]. It is used from the table and from the sync
   modal, so build it once.

3. SyncModal: a spinner while the sync runs, then a summary showing
   totals, per-platform new and updated counts, and any per-platform
   error inline. List newly discovered posts with a link control beside
   each. Include a "Run in background" action that closes the modal and
   shows a toast when the sync finishes.

4. Add a Sync button to the analytics toolbar. Hold the sync request in
   AnalyticsClient rather than in the modal, so closing the modal does
   not cancel it.

5. After a sync completes, re-resolve the date range before refetching.
   A snapshot stamped a moment ago can otherwise fall outside a range
   whose end was fixed when the page loaded, and the new data would not
   appear.

Do not add per-row Analyze actions. Phase 7 adds those.
--- END PROMPT P6.6 ---
```

EXPECTED RESULT

Three new components and an updated analytics page.

VERIFICATION

1. `GET /api/analytics?view=posts&sortBy=views&sortDir=desc` returns the highest-view post first, and `sortDir=asc` reverses it.
2. Record `platform_posts` row counts, `POST /api/analytics/sync`, and confirm the counts change as the response reports. Confirm no more than one snapshot per post exists for today: `select post_id, count(*) from analytics_snapshots where snapshot_date = current_date group by post_id having count(*) > 1;` must return no rows.
3. `PATCH /api/platform-posts/[id]` through the link path, then confirm the corresponding row carries a `content_variant_id` and the draft's status is `posted`.
4. Read `AnalyticsClient` and confirm the sync request is held there, not in `SyncModal`, so closing the modal cannot cancel it. Quote the lines.
5. Read the post-sync refetch and confirm the date range is re-resolved before refetching. Quote the lines.
6. Confirm `LinkToIdeaMenu` loads its options on open rather than on mount.

GATE: Checks 2, 4, and 5 pass. Check 5 is why freshly synced data appears at all.

---

### P6.7 - Build the post detail page

Requires: P6.6
Halts: no

OBJECTIVE

A page for a single published post showing its full metrics and, when linked, the script that produced it.

PROMPT

```text
--- PROMPT P6.7 ---
Read SPEC.md section "Post Detail".

1. Create the page at /analytics/posts/[id] as a server component,
   loading the post, its most recent snapshot, and any linked draft in
   parallel. Return not-found for an unknown id.

2. Create PostDetailClient rendering:

   - A header with the thumbnail, title, a link to the post on its
     platform, the platform badge, the publish date, and the linked
     draft when there is one.
   - A metrics grid: views, likes, comments, shares, saves, reach,
     engagement rate, and watch time when present, labeled with the date
     of the snapshot they came from. Hide saves for TikTok, which does
     not report them. Hide watch time when it is absent. Do not display
     a metric the platform did not provide as zero.
   - A script panel showing the linked draft's script read-only, or a
     control to link this post to a draft when it is unlinked. Build the
     read-only renderer as its own component reusing the Tiptap
     extension set from P3.3.

Do not add Analyze or Repurpose actions, and do not display AI analyses.
Phase 7 adds all of those.
--- END PROMPT P6.7 ---
```

EXPECTED RESULT

One new page and two new components.

VERIFICATION

1. Request `/analytics/posts/[id]` for a real post and confirm 200; for a nonexistent id, confirm 404.
2. Query the post's most recent snapshot and confirm every value the page displays matches it, including the snapshot date label.
3. Open a TikTok post and confirm no saves figure is rendered. Open a post missing watch time and confirm the field is absent rather than showing zero.
4. Confirm a linked post renders the draft's script read-only, and an unlinked post renders a link control instead.
5. `npm run build` exits 0.

GATE: Checks 2 and 3 pass. A missing measurement and a measurement of zero must not render the same.

---

## Checkpoint

```text
Commit all current changes with the message "Phase 6: analytics
dashboard", tag the commit phase-6-complete, and push both the commit and
the tag.
```

## Drift check

```text
Read SPEC.md sections "Analytics Dashboard" and "Post Detail" and compare
them against the running application. List anything specified there that
is missing, and anything present that the specification does not
describe. Ignore anything the specification marks as an AI feature; those
are built in phase 7.
```

## What you have now

A working analytics dashboard: growth against the previous period, engagement and views over time, breakout detection, a sortable catalog, and a detail page pairing each published post with its script.

Phase 7 adds the layer that reads this data and interprets it.
