# Seed Data

Optional. Fills `platform_posts` and `analytics_snapshots` with a realistic fictional catalog so phases 6 and 7 have data to build and verify against.

## When to run it

Run it after `P5.6`, the first task that writes to these tables, or at any point afterward. Run it when any of the following is true:

- No platform was connected in `P1.2`, so sync returns nothing.
- The connected accounts are new, so there is no 90-day history to chart.
- Phase 6 verification needs a catalog large enough to exercise axis capping and breakout detection, which real data may not provide.

Skip it when a synced catalog is already present and sufficient.

It writes only to `platform_posts` and `analytics_snapshots`, never to credentials, drafts, or anything the operator has written, and it labels every row it creates so it can be removed cleanly.

## The prompt

```text
--- PROMPT SEED DATA ---
Read SPEC.md sections "Database Schema: platform_posts" and "Database
Schema: analytics_snapshots".

Write a script at scripts/seed.ts that inserts a fictional but realistic
catalog directly into the database, then run it and report what it
created.

Requirements:

- About 150 posts spread across youtube, tiktok, and instagram, with
  publish dates distributed over the last 18 months, denser in recent
  months.
- Set external_id to a clearly fictional value prefixed "seed-" so every
  seeded row can be identified and deleted later. Never collide with a
  real platform id.
- Plausible titles and captions for a content creator. YouTube posts get
  titles; TikTok and Instagram posts get captions and no title, so the
  caption fallback in the query layer is exercised.
- Thumbnail URLs may point at a placeholder image service or be null.
  Do not download any image.
- 90 days of daily snapshots per post, with views accumulating over time
  rather than fluctuating randomly. A post's view count must never
  decrease from one day to the next.
- Engagement rates in a believable range, mostly 2 to 8 percent, varying
  per platform.
- Reach populated for instagram. For youtube and tiktok set reach equal
  to views and record reachProxy: "view_count" in platform_metrics, as
  the real modules do. Leave saves at 0 for tiktok and impressions at 0
  for instagram.
- watch_time_seconds populated for youtube and tiktok only.
- Include 3 or 4 deliberate outliers at 20 to 50 times the median view
  count, so the chart axis capping in P6.1 has something to trigger on,
  and several posts between 1.5 and 3 times their platform average so
  breakout detection has candidates.
- The script must be safe to run twice: delete existing rows whose
  external_id starts with "seed-" before inserting.

Also write scripts/unseed.ts that deletes every post whose external_id
starts with "seed-", along with their snapshots, and nothing else.

Do not modify any application code.
--- END PROMPT SEED DATA ---
```

## Verification

1. Report seeded post and snapshot counts by platform, the snapshot date range, and the highest and median view count. The highest must be at least 20 times the median, or the axis cap in `P6.5` has nothing to trigger on.
2. Confirm no TikTok or Instagram row has a title, so the caption fallback in `P6.2` is exercised.
3. Confirm view counts are monotonic per post: `select post_id from analytics_snapshots s1 join analytics_snapshots s2 on s1.post_id = s2.post_id and s2.snapshot_date = s1.snapshot_date + 1 where s2.views < s1.views;` returns no rows.
4. Run `scripts/seed.ts` a second time and confirm the row counts are unchanged.
5. `GET /api/analytics?view=overview` returns populated summaries, engagement, views, breakouts, and cadence.

## Removing it

```text
Run scripts/unseed.ts. Confirm every post with a "seed-" external id has
been removed along with its snapshots, and that no other row was
affected. Report the row counts before and after.
```

Seeded and real posts coexist safely, since they never share an external id. Remove the seed data before relying on the dashboard, because a chart mixing real and fictional numbers reports nothing useful.
