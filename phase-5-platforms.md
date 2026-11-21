# Phase 5: Platform Integrations

## What this phase delivers

The connection between the application and the operator's real accounts. By the end, the application can pull the entire published catalog from YouTube, TikTok, and Instagram into the database, with view, like, comment, and engagement numbers for every post.

This is where the credentials validated in `P1.3` are loaded into the application. `P5.5` reads them from `.env.credentials.local` and stores them through the credentials API, so no manual entry is required.

## What sync does

Sync pulls every published post from each connected platform into `platform_posts` and records one snapshot of that post's metrics per day.

Two consequences matter for the implementation. Posts never drafted in the application are still tracked; sync pulls the whole catalog, and linking a synced post back to a draft is a separate operation covered in `P5.7`. Only numbers and text are stored, never media; thumbnails and permalinks are kept as URLs.

The three platform modules are the parts of the application covered by automated tests, added in phase 8.

---

### P5.1 - Build the platform contract and the YouTube module

Requires: P4.3
Halts: no

OBJECTIVE

One normalized metric shape every platform module returns, and the YouTube module implementing it.

PROMPT

```text
--- PROMPT P5.1 ---
Read SPEC.md sections "Platform Sync" and "Database Schema:
analytics_snapshots".

1. Create src/lib/platforms/types.ts defining:

   - NormalizedMetrics: the metric shape every platform module returns,
     mirroring the analytics_snapshots columns, including externalId,
     views, likes, comments, shares, saves, reach, impressions, optional
     watchTimeSeconds, engagementRate, and an open platformMetrics
     object for platform-specific extras.
   - FetchedPost: externalId, title, caption, permalink, thumbnailUrl,
     publishedAt, and metrics. URLs only; never media data.
   - RefreshedToken: accessToken, absolute expiresAt or null, and an
     optional rotated refreshToken.
   - computeEngagementRate: (likes + comments + shares + saves) / reach
     * 100, rounded to two decimals, returning 0 when reach is 0 so the
     result is never Infinity or NaN.
   - PlatformApiError: an Error carrying the platform, HTTP status, and
     the platform's own message.

2. Create src/lib/platforms/youtube.ts against the YouTube Data API v3.

   - validateCredentials(apiKey, channelId) reads the channel with
     part=snippet,statistics and returns the channel id and name. It
     throws when no channel matches. A channel ID is required because an
     API key cannot identify the caller.
   - Fetch the catalog by resolving the channel's uploads playlist and
     paging playlistItems at 50 per page. Do not use search.list: it
     caps at roughly 500 results and costs 100 quota units per request,
     against 1 for playlistItems.
   - Take the publish time from contentDetails.videoPublishedAt, not
     snippet.publishedAt, which is when the video was added to the
     playlist.
   - Fetch statistics with videos.list in chunks of 50 ids per request.
   - Stop paging when the next page token is absent, unchanged, or the
     page is empty, and throw after a hard page limit rather than
     looping forever.
   - Skip any video the statistics call did not return, rather than
     writing a zeroed snapshot over real numbers.
   - The Data API exposes neither reach nor impressions. Use view count
     as the reach proxy, set impressions to 0, and record
     reachProxy: "view_count" in platformMetrics so the dashboard can
     label it honestly.

Do not write the credentials API, the settings page, or the sync route.
--- END PROMPT P5.1 ---
```

EXPECTED RESULT

Two new files under `src/lib/platforms/`. Nothing calls them yet.

VERIFICATION

1. Write a temporary script that reads `YOUTUBE_API_KEY` and `YOUTUBE_CHANNEL_ID` from `.env.credentials.local`, calls the module's `fetchPosts`, and prints the post count plus the title, view count, and publish date of the three most recent. Run it. Delete it afterward. Skip this check if YouTube was not supplied.
2. The count should be consistent with the number of public videos on the channel, and the titles should belong to it. A channel with more than 50 videos also confirms paging, since the first request returns 50.
3. Read the module and confirm `search.list` is not used anywhere.
4. Confirm the publish time is read from `contentDetails.videoPublishedAt`.
5. Confirm `platformMetrics` records `reachProxy: "view_count"`.
6. `npm run build` exits 0.

GATE: Check 1 returns posts belonging to the operator's channel, or YouTube was not supplied.

---

### P5.2 - Build the Instagram module

Requires: P5.1
Halts: no

OBJECTIVE

An Instagram module producing the same shapes as YouTube, including the engagement metrics that are not public.

PROMPT

```text
--- PROMPT P5.2 ---
Read SPEC.md section "Platform Sync". Implement
src/lib/platforms/instagram.ts against the same contract as the YouTube
module, using host https://graph.instagram.com.

- validateCredentials(accessToken) reads /me with fields id, username,
  and account_type, returning all three.
- Page the user's media with a limit of 100, following the cursor at
  paging.cursors.after. Stop when the cursor is absent or unchanged, and
  enforce a hard page limit.
- Fetch per-media insights with metric=reach,likes,comments,shares,saved,
  views.
- Run those insight requests through a concurrency pool of 4 rather than
  sequentially or all at once. Sequential is too slow for a large
  catalog; unbounded triggers rate limiting.
- A post whose insights request fails must not fail the whole run.
- Meta has deprecated impressions in favor of views, so leave impressions
  at 0 rather than inventing a value.
- refreshLongLivedToken(token) calls /refresh_access_token with
  grant_type=ig_refresh_token. Instagram refreshes using the current
  access token, so no app secret is needed at runtime and there is no
  separate refresh token.

Return the same FetchedPost and NormalizedMetrics shapes as YouTube.
--- END PROMPT P5.2 ---
```

EXPECTED RESULT

One new file. Nothing calls it yet.

VERIFICATION

1. Write a temporary script that reads `INSTAGRAM_ACCESS_TOKEN` from `.env.credentials.local`, calls `fetchPosts`, and prints the post count plus the caption, view count, reach, and engagement rate of the three most recent. Run it. Delete it afterward. Skip if Instagram was not supplied.
2. Reach must be a real number rather than a copy of the view count; Instagram reports it directly. Reach of zero on every post means the insights requests are failing while the media requests succeed.
3. Read the module and confirm the insights requests run through a concurrency pool of 4.
4. Confirm a failing insights request for one post does not abort the run.
5. Confirm `impressions` is left at 0.

GATE: Check 1 returns posts, or Instagram was not supplied.

---

### P5.3 - Build the TikTok module

Requires: P5.2
Halts: no

OBJECTIVE

A TikTok module producing the same shapes as the other two, with token refresh that returns TikTok's rotated refresh token.

PROMPT

```text
--- PROMPT P5.3 ---
Read SPEC.md section "Platform Sync". Implement
src/lib/platforms/tiktok.ts against the same contract as the other two
modules, using host https://open.tiktokapis.com.

- validateCredentials(accessToken) reads /v2/user/info/ requesting
  open_id, union_id, display_name, and username.
- Fetch the catalog by POSTing to /v2/video/list/ with the requested
  fields in the query string and max_count plus cursor in the body. The
  maximum page size is 20. Metadata and metrics arrive together.
- Stop paging when the response reports no more results, or when the
  cursor does not advance, and enforce a hard page limit.
- The Display API exposes no reach metric, so use view count as the
  proxy and record that in platformMetrics, as the YouTube module does.
- The Display API does not report saves. Leave saves at 0.
- refreshAccessToken(refreshToken) POSTs to /v2/oauth/token/ as
  form-encoded data using TIKTOK_CLIENT_KEY and TIKTOK_CLIENT_SECRET
  from the environment. TikTok rotates the refresh token on every
  refresh, so the returned refresh token must be included in the result
  for the caller to persist.

Return the same FetchedPost and NormalizedMetrics shapes as the other
modules.
--- END PROMPT P5.3 ---
```

EXPECTED RESULT

One new file. All three platform modules now exist and produce identical shapes.

VERIFICATION

1. Write a temporary script that reads `TIKTOK_ACCESS_TOKEN` from `.env.credentials.local`, calls `fetchPosts`, and prints the video count plus the caption and view count of the three most recent. Run it. Delete it afterward. Skip if TikTok was not supplied.
2. If the request fails because the token has expired, that is correct behavior for a token older than 24 hours rather than a module fault. Call `refreshAccessToken` with the stored refresh token, write the returned access token, expiry, and rotated refresh token back into `.env.credentials.local`, and retry.
3. Compare the return types of `fetchPosts` across all three modules and confirm all three produce identical `FetchedPost` and `NormalizedMetrics` shapes, with no platform-specific fields leaking out.
4. Confirm `refreshAccessToken` returns the rotated refresh token to its caller. TikTok invalidates the old one, so a refresh that discards the new token breaks the next renewal.
5. Confirm `saves` is left at 0.

GATE: Checks 3 and 4 pass. Check 4 is the difference between TikTok working for a day and working indefinitely.

---

### P5.4 - Build the credentials API

Requires: P5.3
Halts: no

OBJECTIVE

Routes for storing, reading, and removing platform credentials, where storing validates against the live platform first and reading never returns a token.

PROMPT

```text
--- PROMPT P5.4 ---
Read SPEC.md section "Settings".

Create these routes, all guarded by checkSameOrigin where they change
data:

GET /api/settings/credentials
  Returns metadata only for every stored credential: platform, metadata,
  lastSyncedAt, tokenExpiresAt, a hasRefreshToken boolean, and
  timestamps. Access tokens and refresh tokens must never appear in a
  response.

POST /api/settings/credentials
  Accepts platform, accessToken, optional refreshToken, optional
  tokenExpiresAt, and an optional channelId used by YouTube.

  - Validate the input shape with Zod and return 400 with the specific
    problem when it fails.
  - Call the matching platform module's validateCredentials before
    saving anything. If the platform rejects the credential, return 400
    carrying the platform's own error message, and store nothing.
  - Store the metadata the platform returned: channel id and name for
    YouTube, user id and username for Instagram, open id and display
    name for TikTok.
  - Upsert on platform, so there is exactly one row per platform.
  - When no expiry is supplied and the platform is Instagram, default
    the expiry to 60 days from now. Instagram long-lived tokens last
    about that long and the settings form has no expiry field, so
    without this default the automatic refresh has nothing to compare
    against and never runs.
  - Return metadata only, never the token.

DELETE /api/settings/credentials/[platform]
  Removes that platform's row. Returns 400 for an unknown platform name
  and 204 on success.
--- END PROMPT P5.4 ---
```

EXPECTED RESULT

Two new route files. No credentials stored yet.

VERIFICATION

1. `GET /api/settings/credentials` returns an empty list.
2. `POST /api/settings/credentials` with platform `youtube`, the deliberately invalid API key `not-a-real-key`, and channel ID `UCinvalid` returns 400 carrying a message originating from Google, not a generic failure.
3. `GET /api/settings/credentials` still returns an empty list, proving the rejected credential was not stored.
4. `psql content_command_center -c "select count(*) from platform_credentials;"` returns 0.
5. `DELETE /api/settings/credentials/notaplatform` returns 400.
6. Read the POST handler and confirm `validateCredentials` is called before any write.

GATE: Checks 3 and 6 pass. A credential must never be stored before the platform accepts it.

---

### P5.5 - Build the settings page and load the credentials

Requires: P5.4
Halts: no

OBJECTIVE

A settings page showing connection state per platform, and every credential the operator supplied in `P1.2` stored in the database through that API.

PROMPT

```text
--- PROMPT P5.5 ---
Read SPEC.md section "Settings".

1. Create a PlatformCredentialCard component rendering one platform's
   fields:

   YouTube: API key and Channel ID, noting that the key reads public
   statistics and never expires.
   Instagram: long-lived access token and an optional refresh token,
   noting that a Business or Creator account is required.
   TikTok: access token, optional refresh token, and an optional
   expires-in value in seconds. Convert that number of seconds into an
   absolute timestamp before saving; do not send the raw seconds.

2. Saving calls POST /api/settings/credentials. While the request is in
   flight, disable the form. If the platform rejects the credential,
   display the platform's own message on the card rather than a generic
   error.

3. When a platform is connected, show instead of the form: a connected
   badge with the account identity returned by the platform, the last
   sync time as a relative description or "never", and the token expiry,
   noting that it auto-refreshes on sync when a refresh token is stored.
   Include a Remove button that calls the DELETE route.

4. Create the settings page rendering one card per platform, reading
   current state from GET /api/settings/credentials.

5. Load the operator's credentials into the database. Read
   .env.credentials.local and, for every platform whose values are
   present, POST them to /api/settings/credentials:

   youtube: accessToken from YOUTUBE_API_KEY, channelId from
   YOUTUBE_CHANNEL_ID.
   instagram: accessToken from INSTAGRAM_ACCESS_TOKEN.
   tiktok: accessToken from TIKTOK_ACCESS_TOKEN, refreshToken from
   TIKTOK_REFRESH_TOKEN, and tokenExpiresAt computed from
   TIKTOK_EXPIRES_IN as an absolute timestamp.

   Skip any platform whose values are empty. If TikTok is rejected
   because its access token has expired, call refreshAccessToken with
   the stored refresh token first, write the rotated values back into
   .env.credentials.local, and retry once.

   Report which platforms connected and the account identity each one
   returned. Do not print any token.

The API never returns tokens, so the page must not attempt to display or
prefill them.
--- END PROMPT P5.5 ---
```

EXPECTED RESULT

Two new components, an updated settings page, and one `platform_credentials` row per platform the operator supplied.

VERIFICATION

1. `GET /api/settings/credentials` returns one entry per supplied platform, each with the account identity the platform returned. That identity could only appear if the credential worked.
2. The response body contains no `accessToken` and no `refreshToken` field, at any nesting level.
3. `psql content_command_center -c "select platform, metadata, token_expires_at from platform_credentials;"` shows one row per supplied platform.
4. `POST /api/settings/credentials` with a deliberately wrong token returns 400 with the platform's own message and leaves the existing row untouched.
5. Request `/settings` and confirm 200 with a card per platform.
6. Read the TikTok card and confirm it converts expires-in seconds to an absolute timestamp before saving.

GATE: Check 1 returns at least one connected platform and check 2 finds no token in the response.

---

### P5.6 - Build the sync orchestration

Requires: P5.5
Halts: no

OBJECTIVE

One route that pulls the full catalog from every connected platform and records today's metrics for every post.

PROMPT

```text
--- PROMPT P5.6 ---
Read SPEC.md section "Platform Sync".

Create POST /api/analytics/sync, guarded by checkSameOrigin, with an
extended maximum duration since a large catalog takes minutes.

For each stored credential:

1. If its token expires within 7 days, refresh it before fetching.
   Instagram refreshes with its own access token. YouTube and TikTok use
   their refresh tokens. Persist the new access token, the new expiry,
   and any rotated refresh token before continuing.

2. Fetch the full catalog through that platform's module.

3. Upsert each post keyed on (platform, external_id). On update, do not
   overwrite an existing content_variant_id: the link between a post and
   a draft is set by hand and must survive every future sync.

4. Upsert one snapshot per post per day, keyed on the post and today's
   date, so syncing twice in one day overwrites rather than duplicating.

5. Stamp the credential's last_synced_at.

Run the platforms with Promise.allSettled so one failure cannot block the
others. Return:

{
  platforms: [{ platform, new, updated, total }],
  totals: { posts, new, updated },
  errors: [{ platform, message }],
  newPosts: [ ...enough detail to display and link a post... ]
}

Return an empty result rather than an error when no credentials are
stored.

Do not build a sync button or modal. Phase 6 adds those.
--- END PROMPT P5.6 ---
```

EXPECTED RESULT

One new route file. Running it fills two tables that have been empty since phase 2.

VERIFICATION

1. `POST /api/analytics/sync` returns 200. Record the response and the row counts: `psql content_command_center -c "select platform, count(*) from platform_posts group by platform;"` and the same for `analytics_snapshots`.
2. The five most recent post titles belong to the operator's accounts.
3. `POST /api/analytics/sync` a second time. It must report zero new posts, and the `analytics_snapshots` row count must not increase. One snapshot per post per day means the second run overwrites the first. A doubling count means the daily key is wrong, which would distort every chart in phase 6.
4. Read the upsert and confirm `content_variant_id` is excluded from the update set.
5. Read the orchestration and confirm the platforms run under `Promise.allSettled`.
6. With no credentials stored the route returns an empty result rather than an error. Verify by reading the branch if credentials are present.

GATE: Check 3 shows no snapshot growth on the second run.

---

### P5.7 - Link a synced post to a draft

Requires: P5.6
Halts: no

OBJECTIVE

A route connecting a published post to the platform variant of the draft it came from, marking that draft posted.

PROMPT

```text
--- PROMPT P5.7 ---
Read SPEC.md section "Platform Sync", subsection on association.

Create PATCH /api/platform-posts/[id], guarded by checkSameOrigin,
accepting { contentPieceId } as either a UUID or null.

When given a piece id:
  - Return 404 if the post or the piece does not exist.
  - Find the variant for this post's platform on that piece, creating it
    with empty platform data if there is none.
  - The post-to-variant link is one to one. Release any other post
    currently attached to that variant before attaching this one.
  - Attach the post to the variant.
  - Set the piece's status to "posted".

When given null:
  - Clear the post's association and leave the piece untouched.

Do not build a user interface for linking. Phase 6 adds that.
--- END PROMPT P5.7 ---
```

EXPECTED RESULT

One new route file.

VERIFICATION

1. Pick a synced post and a content piece. `PATCH /api/platform-posts/[id]` with that `contentPieceId`. Confirm the post's `content_variant_id` is set, the variant was found or created for the post's platform, and the piece's status is now `posted`.
2. Link a second post to the same piece and platform. Confirm the first post was released rather than the request failing. Two posts claiming one variant would mean the database uniqueness rule is doing the work and the second link would error instead of replacing.
3. `PATCH` with `contentPieceId: null`. Confirm the association is cleared and the piece's status is unchanged.
4. `PATCH` with a nonexistent post id returns 404, and with a nonexistent piece id returns 404.
5. Run a sync again and confirm the link from check 1 survived it.

GATE: Checks 2 and 5 pass. Check 5 confirms the sync upsert preserves the link.

---

## Checkpoint

```text
Commit all current changes with the message "Phase 5: platform
integrations", tag the commit phase-5-complete, and push both the commit
and the tag.
```

## Drift check

```text
Read SPEC.md sections "Platform Sync" and "Settings" and compare them
against the running application. List anything specified there that is
missing, and anything present that the specification does not describe.
```

## What you have now

A database holding the published catalog from every connected platform, with a metrics snapshot per post, and the ability to connect a published post back to its draft.

None of it is visible yet. Phase 6 builds the dashboard that reads it.

If no platform was connected, or the connected accounts are new, run `seed-data.md` before phase 6 so the dashboard has data to display.
