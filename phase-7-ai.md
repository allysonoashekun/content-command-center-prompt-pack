# Phase 7: AI Integration

## What this phase delivers

Claude reading the stored performance data and returning analyses, captions, and repurposed ideas.

Everything is suggest-only. The AI never changes content on its own. It produces a suggestion, the operator approves or dismisses it, and the decision is recorded. The prompts enforce this.

This phase uses the Anthropic key loaded in `P2.1` and is the only phase that costs money to run. Each analysis processes a few thousand words. Building and verifying this phase costs a few cents in total.

## The four suggestion types

| Type | Triggered from | Returned | Saved |
|------|----------------|----------|-------|
| Performance analysis | A post row, breakout card, chart bar, or post detail page | Streamed | Yes |
| Content gap | The analytics toolbar | Streamed | Yes |
| Weekly brief | The analytics toolbar | Streamed | Yes |
| Captions, hooks and hashtags | The editor's suggestion panel | Complete, as structured data | Yes |

Repurposing is a separate flow. Its output becomes new drafts rather than a saved suggestion.

Long analyses stream because the first visible token arriving quickly matters. Short structured results do not stream, because the structure matters more than the wait.

---

### P7.1 - Build the Claude client and prompt module

Requires: P6.7
Halts: no

OBJECTIVE

A lazily constructed Anthropic client, and one module holding every prompt with a validated schema for the context each requires.

PROMPT

```text
--- PROMPT P7.1 ---
Read SPEC.md section "AI Insights", including the pinned model name.

1. Create src/lib/ai/client.ts exporting getAnthropic(), which builds
   the Anthropic client the first time it is called and reuses it after.

   Do not construct the client when the file is imported. If
   ANTHROPIC_API_KEY is missing, an eagerly-built client throws the
   moment any route importing this file loads, which breaks unrelated
   routes. Building it on demand lets each route return a clean error
   instead.

2. Create src/lib/ai/prompts.ts. It must not import the database.

   - Export AI_MODEL, set to the model named in SPEC.md. Define it here
     rather than in client.ts so client components can display it
     without importing the SDK.
   - Export a single frozen SYSTEM_PROMPT describing the assistant as a
     content strategy advisor for a solo creator publishing to YouTube,
     TikTok, and Instagram. Send it as one block marked with
     cache_control ephemeral on every request: it is identical every
     time, so caching it avoids paying to re-read it.
   - Define a Zod schema per suggestion type describing the context that
     type requires, and validate context against it before it reaches
     the model.
   - Write one prompt builder per type: performance analysis, content
     gap, weekly brief, captions, and repurposing. Each returns the
     system blocks and the user message.
   - Export buildStreamingPrompt and buildStructuredPrompt, dispatching
     by type and throwing on an unknown type.
   - Export the JSON schema for structured suggestion output, and a
     parser that validates a model response against it.
   - Export extractTopInsights, which pulls a trailing "Top 3 insights"
     section out of a content gap response into a list of action items,
     returning the remaining text as the body.
   - Export display titles for each suggestion type.

   Do not enable extended thinking. It delays the first visible token,
   and these responses are read as they arrive.

Do not create routes or components.
--- END PROMPT P7.1 ---
```

EXPECTED RESULT

Two new files under `src/lib/ai/`. Nothing calls them yet.

VERIFICATION

1. Grep the imports of `prompts.ts` and confirm it imports neither the Anthropic SDK nor anything from `src/lib/db`.
2. Import `client.ts` in a Node process with `ANTHROPIC_API_KEY` unset and confirm the import does not throw. If importing throws without a key, every AI route fails to load rather than returning an error.
3. Call `buildStreamingPrompt` for a weekly brief with a small sample context and print the resulting system blocks and user message without sending them to the model. Confirm the system block carries `cache_control` marked ephemeral.
4. Call `buildStreamingPrompt` with an unknown type and confirm it throws.
5. Pass a context missing a required field and confirm its Zod schema rejects it.
6. Confirm `AI_MODEL` matches the model named in `SPEC.md` and that extended thinking is not enabled anywhere.
7. `npm run build` exits 0.

GATE: Checks 1 and 2 pass.

---

### P7.2 - Build the streaming analysis route

Requires: P7.1
Halts: no

OBJECTIVE

A route that saves a pending suggestion, returns its id in a header, streams the generated text, and persists the full result with token counts.

PROMPT

```text
--- PROMPT P7.2 ---
Read SPEC.md section "AI Insights".

1. Create src/lib/editor/prosemirror.ts exporting proseMirrorToText,
   which flattens a Tiptap document to plain text, separating
   block-level nodes with newlines so the result stays readable.

2. Create POST /api/ai/stream, guarded by checkSameOrigin.

   - Accept a suggestion type, a context object, and optional
     contentPieceId, contentVariantId, platformPostId, and platform.
   - Validate the body, returning 400 on a bad shape and 400 separately
     when the context fails its type's schema.
   - Return 500 with a clear message when the API key is not configured.
   - For a performance analysis on a post linked to a draft, look up
     that draft and add its title, concept, and a script summary
     truncated to 4000 characters to the context, so the model can
     compare the result against the original intent. If that lookup
     fails, continue with the original context rather than failing the
     request.
   - Insert the suggestion row as pending BEFORE streaming starts, and
     return its id in an X-Suggestion-Id response header. The client
     needs that id to approve the result, and it cannot wait until the
     stream ends to receive it.
   - Stream text deltas to the client as plain text.
   - When the final message arrives, save the full text and the input
     and output token counts onto the row.
   - For a content gap analysis, run the response through
     extractTopInsights and store the action items alongside the body.
   - If generation fails mid-stream, append a visible marker to the
     stream rather than closing silently.

Do not build any user interface. Later tasks do that.
--- END PROMPT P7.2 ---
```

EXPECTED RESULT

Two new files. The route can be called but nothing calls it yet.

VERIFICATION

1. `POST /api/ai/stream` with type `weekly_brief` and a context built from the ten most recent posts per platform. Confirm the `X-Suggestion-Id` header is present in the response headers, before the body has finished arriving.
2. After the stream completes, query the row: `psql content_command_center -c "select type, status, input_tokens, output_tokens, model from ai_suggestions order by created_at desc limit 1;"`. Status is `pending`, the body is filled, and both token counts are non-zero.
3. `POST /api/ai/stream` with a malformed body returns 400; with a well-formed body whose context fails its type schema, also 400, through a distinct branch.
4. Run a `content_gap` analysis and confirm the stored row has action items extracted from the trailing insights section, separate from the body.
5. Confirm the pending row is inserted before the stream opens. Quote the ordering.

GATE: Checks 1 and 2 pass. The header must be sent before the text finishes, or the client cannot approve the result.

---

### P7.3 - Build the structured suggestion routes

Requires: P7.2
Halts: no

OBJECTIVE

Routes for generating structured suggestions, listing saved ones, and recording an approve or dismiss decision.

PROMPT

```text
--- PROMPT P7.3 ---
Read SPEC.md section "AI Insights".

1. Create POST /api/ai/suggestions, guarded by checkSameOrigin.
   Generate a structured suggestion using the model's JSON schema output
   format with the schema from prompts.ts, validate the response, save
   it, and return the saved record with status 201.

2. Add GET /api/ai/suggestions, listing saved suggestions newest first,
   filterable by contentPieceId, contentVariantId, status, type, and
   platform.

   The platform filter takes one special value: platform=none means
   suggestions with no platform recorded, which is what a suggestion
   generated from the All tab looks like. That is different from
   omitting the parameter, which means no platform filtering at all.
   Return 400 for an invalid filter value.

3. Create PATCH /api/ai/suggestions/[id], guarded by checkSameOrigin,
   accepting a status of approved or dismissed. Set the status and the
   review timestamp, and return 404 when no row matches.

   Approving records a decision. It must not copy content anywhere or
   modify any draft.
--- END PROMPT P7.3 ---
```

EXPECTED RESULT

Two new route files.

VERIFICATION

1. `POST /api/ai/suggestions` for a draft with a platform variant returns 201 with a saved record whose content matches the declared JSON schema.
2. Record the draft and its variant row in full. `PATCH /api/ai/suggestions/[id]` to `approved`. Confirm the suggestion's status and `reviewed_at` changed, and that the draft row and the variant row are byte-for-byte unchanged. Approval records a decision; it does not act.
3. `GET /api/ai/suggestions?platform=none` and `GET /api/ai/suggestions` return different sets: the first only suggestions with a null platform, the second everything.
4. `GET /api/ai/suggestions?status=notastatus` returns 400.
5. `PATCH /api/ai/suggestions/00000000-0000-0000-0000-000000000000` returns 404.

GATE: Check 2 shows the draft and variant unmodified.

---

### P7.4 - Build the streaming display components

Requires: P7.3
Halts: no

OBJECTIVE

The shared display layer: a streaming hook, an incremental text renderer safe under double-mounting, a markdown body, a suggestion card, and the dialog that hosts them.

PROMPT

```text
--- PROMPT P7.4 ---
Read SPEC.md section "AI Insights".

1. Create a useStreamingSuggestion hook that POSTs to /api/ai/stream and
   exposes the response body, the suggestion id from the X-Suggestion-Id
   header, a streaming flag, an error, and functions to start, reset,
   and mark the stream finished.

2. Create StreamingText, which decodes a response stream incrementally
   and appends text as it arrives, with a blinking cursor while active.

   A stream can only be read once. React StrictMode mounts components
   twice in development, so guard against reading a stream that is
   already locked, and release the reader on cleanup. Without this the
   component throws on every second mount in development only, which is
   a confusing way to discover the problem.

3. Create MarkdownBody, rendering markdown with GitHub-flavored markdown
   support, including tables. Give tables their own horizontal scroll
   container so a wide table cannot stretch the page.

4. Create SuggestionCard, rendering a headline, the markdown body, any
   action items, a status badge once reviewed, and a footer showing the
   model name and total token count. Show Approve and Dismiss only while
   the suggestion is pending, calling PATCH /api/ai/suggestions/[id].

5. Create AISuggestionSheet, a centered dialog that shows a spinner and
   a generating message while the stream drains, then renders the
   finished card with Approve and Dismiss pinned at the bottom. Hide the
   close control while streaming so a half-written analysis is not
   dismissed by accident.

Do not wire these to any button yet. The next task does that.
--- END PROMPT P7.4 ---
```

EXPECTED RESULT

Five new files. Nothing renders them yet.

VERIFICATION

1. Read `StreamingText` and confirm it guards against reading an already-locked stream and releases its reader on cleanup. Quote both. This is what makes it survive React StrictMode's double mount in development.
2. Read `SuggestionCard` and confirm Approve and Dismiss render only when the status is `pending`. Quote the condition.
3. Confirm `MarkdownBody` wraps tables in their own horizontal scroll container.
4. Confirm `AISuggestionSheet` hides its close control while streaming.
5. Confirm `useStreamingSuggestion` reads the suggestion id from the `X-Suggestion-Id` header.
6. `npm run build` exits 0.

GATE: Checks 1 and 2 pass.

---

### P7.5 - Wire AI into the analytics dashboard

Requires: P7.4
Halts: no

OBJECTIVE

Analyze actions everywhere a post appears, weekly brief and content gap actions on the toolbar, and a carousel of headline insights from the most recent approved content gap analysis.

PROMPT

```text
--- PROMPT P7.5 ---
Read SPEC.md sections "AI Insights" and "Analytics Dashboard".

1. Add an Analyze action to each place a post appears: rows in the post
   table, breakout cards, bars on the per-post chart, and the post
   detail page. All four open the same AISuggestionSheet and request a
   performance analysis for that post, passing the linked draft's id
   when the post has one.

2. Add Weekly brief and Content gaps buttons to the analytics toolbar.
   Each streams into the same dialog. Pass the active platform tab as
   the suggestion's platform, and no platform when the All tab is
   selected.

3. Create TopInsightsCard: a carousel of the three headline action items
   from the most recent approved content gap analysis, with dot
   navigation and a link to the full analysis page. Render nothing when
   there is no approved analysis yet.

4. On the post detail page, show the most recent performance analysis
   for that post with its model and token counts, offering Approve and
   Dismiss while it is pending, and a Generate analysis action when
   there is none.

Reuse AISuggestionSheet everywhere. Do not build a second dialog.
--- END PROMPT P7.5 ---
```

EXPECTED RESULT

Updated analytics components and post detail page.

VERIFICATION

1. Trigger a performance analysis for a specific post through the route the Analyze action calls. Confirm a row is saved with that `platformPostId` and that the generated text refers to that post's own metrics.
2. Confirm all four Analyze entry points open `AISuggestionSheet` and that no second dialog component exists. Grep for other dialog definitions.
3. Generate a content gap analysis and leave it pending. Confirm `TopInsightsCard` renders nothing. Approve it. Confirm the card now renders three action items. It reads only approved analyses.
4. Confirm a toolbar analysis run from a single-platform tab stores that platform, and one run from the All tab stores none.
5. Query `ai_suggestions` and confirm each row created carries the type, status, platform, and token counts expected from the trigger that produced it.

GATE: Checks 2 and 3 pass.

---

### P7.6 - Build the suggestion surfaces

Requires: P7.5
Halts: no

OBJECTIVE

A caption-generating panel in the editor, and a page listing the full analysis history with filters.

PROMPT

```text
--- PROMPT P7.6 ---
Read SPEC.md sections "AI Insights" and "Full Analysis".

1. Create SuggestionPanel for the right rail of /drafts/[id]:

   - A Generate captions action that fires one structured request per
     existing platform variant, in parallel rather than in sequence.
   - When the draft has no variants, show a message asking the reader to
     add one first rather than sending a request with nothing to work
     from.
   - Two tabs, Suggestions and History, splitting pending from reviewed,
     each showing a live count.
   - Render each suggestion with SuggestionCard, so Approve and Dismiss
     behave as they do everywhere else.
   - Add the panel to the editor page beneath the variant editor.

2. Create the page at /analytics/analysis:

   - Platform tabs, with the initial tab taken from a platform query
     parameter so links from the dashboard open on the right one.
   - A time range filter: last week, last month as the default, last two
     months, last six months, or all time.
   - A toggle to show or hide dismissed suggestions.
   - Render the most recent approved weekly brief and the most recent
     approved content gap analysis, each with its model, token counts,
     and review actions.
   - Below those, the history of analyses for the selected filters.
--- END PROMPT P7.6 ---
```

EXPECTED RESULT

Two or three new components, one new page, and an updated editor page.

VERIFICATION

1. For a draft with two platform variants, trigger Generate captions. Confirm one suggestion row is saved per variant, each carrying its own `content_variant_id`.
2. Confirm the requests are issued in parallel rather than sequentially. Quote the call site.
3. Approve one suggestion and confirm the pending and reviewed counts each change by one, matching the two tabs.
4. Confirm a draft with no variants renders the add-a-variant message and issues no request.
5. Request `/analytics/analysis` and confirm 200. Confirm `?platform=youtube` opens on the YouTube tab.
6. Confirm the dismissed toggle and each time range change the returned set.

GATE: Checks 1, 2, and 4 pass.

---

### P7.7 - Build the repurpose flow

Requires: P7.6
Halts: no

OBJECTIVE

Turning a published post into four to six new drafts in the Backlog, with their outlines seeded into the script.

PROMPT

```text
--- PROMPT P7.7 ---
Read SPEC.md section "AI Insights", subsection on repurposing.

1. Add outlineToProseMirror to src/lib/editor/prosemirror.ts, building a
   Tiptap document from a description and a list of outline beats: the
   description as a paragraph, an "Outline" heading, then the beats as a
   bullet list. Use only nodes the editor's extension set supports, and
   never emit an empty text node, which the editor rejects.

2. Create POST /api/ai/repurpose, guarded by checkSameOrigin.

   - Require the linked draft's id. Read the script from the database
     server-side rather than accepting it from the client: it is the
     authoritative copy and it can be large.
   - Flatten the script to text and truncate it to 6000 characters.
   - Return 400 with a clear message when the draft has no script.
   - Request structured output and return { ideas: [{ title, angle,
     description, outline }] }.
   - Do not save the ideas. They are returned and discarded.

3. Create RepurposeModal with three states: loading, review, and saved,
   plus a retry path when generation fails.

   - Show each idea with its title, angle, and outline, all
     pre-selected.
   - Saving creates each selected idea as a new content piece in the
     Backlog category, tagged "Repurposed", with its outline seeded into
     the script through outlineToProseMirror.
   - Tolerate partial failure: report which ideas saved and which did
     not rather than discarding the successful ones.

4. Add a Repurpose action to the post detail page, enabled only when the
   post is linked to a draft that has a script.
--- END PROMPT P7.7 ---
```

EXPECTED RESULT

One updated helper file, one new route, one new component, and an updated post detail page.

VERIFICATION

1. `POST /api/ai/repurpose` for a post linked to a draft with a script returns four to six ideas, each with a title, angle, description, and outline.
2. Confirm no `ai_suggestions` row was created by that call. Repurposing deliberately saves drafts, not suggestions.
3. Save the ideas. Query `psql content_command_center -c "select title, category, series_tag from content_pieces where series_tag = 'Repurposed';"` and confirm each is in the Backlog category.
4. Load one of those pieces' `script_notes` into the Tiptap schema and confirm it parses without error. Confirm it contains a paragraph, an "Outline" heading, and a bullet list, and contains no empty text node.
5. `POST /api/ai/repurpose` for a draft with no script returns 400 with a clear message.
6. Confirm the Repurpose action is disabled for a post with no linked draft.

GATE: Checks 2 and 4 pass. An empty text node makes the saved draft unopenable in the editor.

---

## Checkpoint

```text
Commit all current changes with the message "Phase 7: AI integration",
tag the commit phase-7-complete, and push both the commit and the tag.
```

## Drift check

```text
Read SPEC.md sections "AI Insights" and "Full Analysis" and compare them
against the running application. List anything specified there that is
missing, and anything present that the specification does not describe.
```

## What you have now

The complete application: planning on a board and a calendar, scripts and per-platform variants, live metrics from three platforms, a dashboard that compares periods and finds breakouts, and an AI layer that reads all of it and suggests what to do next without changing anything on its own.

Phase 8 covers testing it, auditing it, and operating it.
