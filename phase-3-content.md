# Phase 3: Content

## What this phase delivers

The planning half of the application: a kanban board, a rich-text script editor, per-platform variant forms, and a scratchpad. Nothing here touches YouTube, TikTok, or Instagram.

## The two status axes

Every piece of content carries two independent state fields. This runs through the whole phase and every task in it depends on both existing.

| Field | Values | Controls | Changed by |
|-------|--------|----------|------------|
| `category` | Backlog, Idea, In Progress, Complete | Which lane the piece sits in | Dragging a card between lanes |
| `status` | Idea, Draft, Draft Complete, Filming, Filming Complete, Editing, Editing Complete, Scheduled, Posted | Production stage, shown as a badge | The status dropdown, or scheduling on the calendar |

A card never displays its own category; the lane it sits in already says that. It displays its status.

---

### P3.1 - Build the content API routes

Requires: P2.4
Halts: no

OBJECTIVE

Six API routes covering create, read, update, and delete for content and its platform variants, with a same-origin guard on every mutating route, plus three sample rows for the next task to render.

PROMPT

```text
--- PROMPT P3.1 ---
Read SPEC.md sections "Drafts & Kanban", "Platform Variants", and
"Database Schema". Read the route handler documentation shipped in
node_modules for the installed Next.js version before writing any route.

1. Create src/lib/http/same-origin.ts exporting checkSameOrigin. It
   compares the request Origin header's host against the Host header and
   returns a 403 response when they differ. A request with no Origin
   header is allowed, so command line tools keep working. Document why in
   a comment.

2. Create these routes, validating input with Zod:

   GET /api/content
     Lists pieces with their variants attached, newest first. Optional
     status, category, and platform query filters. Status and category
     become SQL conditions; platform filters in memory after the join.
     An invalid filter value returns 400.

   POST /api/content
     Creates a piece, defaulting the title to "Untitled". Returns 201.

   PATCH /api/content/[id]
     Partial update of title, concept, series tag, script notes, status,
     category, or scheduled date. An explicitly null value must clear the
     field rather than being ignored. Returns 404 when no row matches.

   DELETE /api/content/[id]
     Deletes the piece. Its variants are removed by the cascade already
     defined in the schema. Returns 204.

   POST /api/content/[id]/variants
     Creates or updates the variant for one platform. platformData is
     replaced wholesale, not merged.

   DELETE /api/content/[id]/variants
     Deletes one platform's variant, with the platform given in the
     request body.

3. Every route that changes data must call checkSameOrigin first and
   return its response when one is given.

4. Create three sample content pieces directly in the database with
   varied titles, categories, and statuses, so the next task has content
   to display.

Do not build any user interface in this task.
--- END PROMPT P3.1 ---
```

EXPECTED RESULT

One helper file, four route files, and three rows in `content_pieces`.

VERIFICATION

1. Run a full cycle against the dev server: `POST /api/content` returns 201 with an id; `PATCH /api/content/[id]` changing `category` returns 200; `GET /api/content` returns the piece with the new category; `DELETE /api/content/[id]` returns 204; a second `GET` no longer lists it.
2. `GET /api/content?status=notastatus` returns 400, not 500.
3. `PATCH /api/content/00000000-0000-0000-0000-000000000000` returns 404.
4. `psql content_command_center -c "select title, status, category from content_pieces;"` lists three rows.
5. Read each mutating route and confirm it calls `checkSameOrigin` before doing anything else.

GATE: The cycle in check 1 completes and check 5 finds no unguarded mutating route.

---

### P3.2 - Build the Kanban board

Requires: P3.1
Halts: no

OBJECTIVE

Four lanes on `/drafts` holding content by category, with cards draggable between lanes and the move persisted.

PROMPT

```text
--- PROMPT P3.2 ---
Read SPEC.md section "Drafts & Kanban" and the "two status axes"
description in that section.

Build three components using dnd-kit:

1. KanbanBoard, a client component that fetches GET /api/content on
   mount and renders one lane per category in the order defined in
   SPEC.md. Use a PointerSensor with an 8px activation distance so a
   click is not mistaken for a drag. Render a DragOverlay showing a
   floating copy of the card while it is being dragged. Show
   per-category skeleton placeholders while loading.

2. KanbanSwimlane, a droppable lane keyed by its category, with a header
   showing the category name and a live count, its own independent
   scroll, a background highlight while a card hovers over it, and a
   "Drop here" message when empty.

3. ContentRow, a draggable row showing: a drag handle revealed on hover,
   the title, the series tag, the scheduled date but only when the
   status is "scheduled", a small chip per platform variant, and the
   status badge.

On drop, update the piece's category in local state first, then PATCH
/api/content/[id]. If the request fails, restore the previous category
and show an error toast.

Render the board on the /drafts page.

Do not add click-through navigation, a delete control, or a New Content
button. Later tasks add those. Do not add reordering within a lane.
--- END PROMPT P3.2 ---
```

EXPECTED RESULT

Three new components and an updated drafts page. The three sample pieces appear in the lanes matching their categories.

VERIFICATION

1. `GET /api/content` returns the three pieces with their categories, and requesting `/drafts` returns 200 with all four lane names present in the body.
2. Simulate the drop path directly: `PATCH /api/content/[id]` with a new `category`, then `GET /api/content` and confirm the change persisted. This is the behavior the drag is wired to.
3. Read `KanbanBoard` and confirm the `PointerSensor` sets an 8px activation distance. Quote the line.
4. Read the drop handler and confirm the failure path restores the previous category and raises a toast. Quote the lines.
5. `npm run build` exits 0.

GATE: Checks 2, 3, and 4 pass. Check 4 is what makes a failed save visible rather than silent.

---

### P3.3 - Build the script editor

Requires: P3.2
Halts: no

OBJECTIVE

A page per content piece holding a Tiptap editor that autosaves the script to `script_notes`, reached by clicking a card.

PROMPT

```text
--- PROMPT P3.3 ---
Read SPEC.md section "Content Editor".

1. Create the page at /drafts/[id] as a server component that loads the
   piece from the database directly and returns a not-found response for
   an unknown id. Lay it out as a single full-height column containing
   the editor.

2. Create TiptapEditor as a client component using StarterKit,
   Placeholder, Image, Link, Underline, and CharacterCount.

   - The script is stored in content_pieces.script_notes as the editor's
     JSON document.
   - On every change, wait 800ms after typing stops, then PATCH
     /api/content/[id] with the document.
   - Show a save indicator reading Saving, Saved, or Unsaved, colored
     differently in each state.
   - Show live word and character counts in the footer.
   - Clear the pending save timer when the component unmounts.

3. Create EditorToolbar as a separate component taking the editor as a
   prop, so it can be reused later by another editor. Include: bold,
   italic, underline; heading level 1 and 2; bullet list, ordered list,
   blockquote; link and image, each collecting a URL with window.prompt;
   undo and redo, disabled when unavailable. Highlight a button when its
   formatting is active at the cursor.

4. In ContentRow, make clicking the row navigate to /drafts/[id].
   Exclude the drag handle from this, so dragging never navigates.

Do not add the header controls, the platform variant panel, or any AI
feature to this page. Later tasks add those.
--- END PROMPT P3.3 ---
```

EXPECTED RESULT

One new page and two new components, plus an updated `ContentRow`.

VERIFICATION

1. Request `/drafts/[id]` for a real id and confirm 200; request it for a nonexistent uuid and confirm 404.
2. `PATCH /api/content/[id]` with a Tiptap JSON document in `scriptNotes`, then query `psql content_command_center -c "select script_notes from content_pieces where id = '...';"` and confirm the document round-trips with its marks intact.
3. Read `TiptapEditor` and confirm the debounce is 800ms and that the pending timer is cleared on unmount. Quote both.
4. Read `ContentRow` and confirm the drag handle is excluded from the navigation click handler. Quote the line.
5. `npm run build` exits 0.

GATE: Check 2 shows formatting preserved and check 3 finds the unmount cleanup.

---

### P3.4 - Build the editor header controls

Requires: P3.3
Halts: no

OBJECTIVE

Five independently saving controls above the editor: back link, editable title, series tag, scheduled date, and status dropdown.

PROMPT

```text
--- PROMPT P3.4 ---
Read SPEC.md section "Content Editor", subsection "Header controls".

Create ContentEditorHeader as a client component holding five controls,
each saving independently by calling PATCH /api/content/[id]:

1. A back link returning to /drafts.

2. EditableTitle, as its own component: a borderless inline input that
   saves on blur, skips the request when the value has not changed,
   trims whitespace, and falls back to "Untitled" when emptied. Show a
   brief saving indicator while the request is in flight.

3. A free-text series tag input that saves on blur. Clearing it must
   send null, not an empty string.

4. A native date input for the scheduled date. Build the date from its
   local year, month, and day components rather than parsing the
   "YYYY-MM-DD" string directly, because that string is interpreted as
   UTC midnight and would land on the previous day for anyone behind
   UTC. Clearing the input sends null.

5. A status dropdown listing all nine production statuses with their
   display labels. Update local state first, then save; restore the
   previous value and show an error toast if the save fails.

Render the header at the top of the /drafts/[id] page.

Do not add a save button. Do not change the category from this header.
--- END PROMPT P3.4 ---
```

EXPECTED RESULT

Two new components and an updated editor page.

VERIFICATION

1. `PATCH /api/content/[id]` with `seriesTag: null` and confirm the column is null, not an empty string.
2. Set a scheduled date, then query `scheduled_for` and confirm the stored date is the calendar day intended in the machine's local timezone, not the day before. This is the check that catches the UTC parsing error.
3. `PATCH` a new `status` and confirm `category` is unchanged in the same row. The two axes must move independently.
4. Read the date control and confirm it constructs the date from local year, month, and day components rather than parsing the string. Quote the lines.
5. Read the status dropdown and confirm the failure path restores the previous value. Quote the lines.

GATE: Checks 2 and 3 pass. They are the two that catch real modeling errors.

---

### P3.5 - Add content creation and deletion

Requires: P3.4
Halts: no

OBJECTIVE

A New Content button that creates a piece and opens it, and a delete control on each card behind a confirmation dialog.

PROMPT

```text
--- PROMPT P3.5 ---
Read SPEC.md section "Drafts & Kanban".

1. Create NewContentButton as a client component that POSTs to
   /api/content with the title "Untitled" and navigates to
   /drafts/[id] for the returned piece. Disable it while the request is
   in flight. Show an error toast on failure. Render it in the drafts
   page TopNav actions slot.

   Do not add a dialog asking for a title first.

2. Add a delete control to ContentRow, revealed on hover, that opens a
   confirmation dialog. The dialog names the piece and states that its
   platform variants will also be deleted permanently.

3. On confirmation, remove the piece from local state first, then call
   DELETE /api/content/[id]. On failure, restore the piece to its
   original position in the list and show an error toast.

4. Clicking the delete control must not navigate to the editor.
--- END PROMPT P3.5 ---
```

EXPECTED RESULT

One new component and an updated `ContentRow`.

VERIFICATION

1. `POST /api/content` returns 201 with an id, and `GET /api/content` then lists a piece titled "Untitled".
2. Create a piece, add a variant to it, then `DELETE /api/content/[id]`. Confirm both `content_pieces` and `content_variants` row counts drop, proving the cascade.
3. Read `NewContentButton` and confirm it disables itself while the request is in flight. Quote the line.
4. Read the delete control and confirm the click handler stops propagation so it cannot navigate. Quote the line.
5. Read the confirmation dialog and confirm it names the piece and states that variants are deleted.

GATE: Check 2 shows the variant row removed with its parent.

---

### P3.6 - Build the platform variant forms

Requires: P3.5
Halts: no

OBJECTIVE

A tabbed panel beside the script with a per-platform form for YouTube, TikTok, and Instagram, each autosaving its whole field set.

PROMPT

```text
--- PROMPT P3.6 ---
Read SPEC.md section "Platform Variants".

1. Create VariantEditor as a client component showing an "Add" button
   for each platform that has no variant yet. Adding one POSTs to
   /api/content/[id]/variants with empty platform data and reveals a
   tab; the Add button for that platform then disappears. Each tab
   carries a small remove control that deletes the variant and its saved
   data. Show an empty state when no variants exist.

2. Create three form components using react-hook-form with Zod
   resolvers. Each watches the whole form and upserts to
   /api/content/[id]/variants 800ms after the last change.

   YouTube: Title with a 110-character limit and a live counter,
   Description as a multi-line field, and Tags entered comma-separated
   and stored as an array.

   TikTok: Caption with a 2200-character limit and a live counter,
   Hashtags separated by spaces or commas with any leading "#" removed
   and stored as an array, and an optional Sound name.

   Instagram: Caption with a 2200-character limit and a live counter,
   Media URLs entered one per line and stored as an array, and optional
   Alt text.

3. Add the panel to the right side of the /drafts/[id] page, beside the
   script editor.

Note that platformData is replaced wholesale on save, so every form must
send all of its fields on every save, not only the changed ones.

Do not add an AI suggestion panel. Phase 7 adds that.
--- END PROMPT P3.6 ---
```

EXPECTED RESULT

Four new components and an updated editor page.

VERIFICATION

1. `POST /api/content/[id]/variants` for TikTok with hashtags entered as `#one #two`, then query `platform_data` and confirm the hashtags are stored as an array of `["one","two"]` with the `#` stripped.
2. Save a YouTube variant with title, description, and tags, then save it again changing only the title, and confirm description and tags survive. This is the wholesale-replacement contract.
3. `DELETE /api/content/[id]/variants` with a platform in the body removes only that variant.
4. Read each form and confirm it submits its complete field set on every save, not only dirty fields. Quote the submit path.
5. `GET /api/content` returns each piece with its variants attached, so the board can render platform chips.

GATE: Checks 1 and 2 pass. Check 2 is what a partial-update form would fail.

---

### P3.7 - Build the scratchpad

Requires: P3.6
Halts: no

OBJECTIVE

A single application-wide notepad beside the board, autosaving, backed by exactly one database row.

PROMPT

```text
--- PROMPT P3.7 ---
Read SPEC.md section "Scratchpad".

1. Create GET and PATCH /api/scratchpad. GET returns the single
   scratchpad row, creating it if none exists. PATCH updates that row's
   content, creating the row if it is somehow missing. PATCH must call
   checkSameOrigin first and return 400 on invalid input.

2. Create ScratchpadPanel as a client component using the same Tiptap
   extension set as the script editor except CharacterCount, and reusing
   EditorToolbar.

   - Content is stored as the editor's JSON document, serialized to a
     string.
   - Save 800ms after the last change, with its own Saved, Saving,
     Unsaved indicator.
   - On load, if the stored content is not valid editor JSON, treat it
     as older plain text and convert its line breaks so it still opens.
   - Show a loading placeholder until the content has been fetched.

3. Place the panel in the right pane of the /drafts page, beside the
   board.
--- END PROMPT P3.7 ---
```

EXPECTED RESULT

One new route file and one new component.

VERIFICATION

1. `GET /api/scratchpad` twice, then `psql content_command_center -c "select count(*) from scratchpad;"` returns exactly 1. More than one row means the find-before-insert is wrong.
2. `PATCH /api/scratchpad` with a document containing a bullet list, then `GET` and confirm the list structure round-trips.
3. `PATCH /api/scratchpad` with an invalid body returns 400.
4. Read `ScratchpadPanel` and confirm the plain-text fallback path exists for content that is not valid editor JSON. Quote the lines.
5. `npm run build` exits 0.

GATE: Check 1 returns exactly one row.

---

## Checkpoint

```text
Commit all current changes with the message "Phase 3: content", tag the
commit phase-3-complete, and push both the commit and the tag.
```

## Drift check

```text
Read SPEC.md sections "Drafts & Kanban", "Scratchpad", "Content Editor",
and "Platform Variants" and compare each against the running application.
List anything specified there that is missing, and anything present that
the specification does not describe.
```

## What you have now

A working content planner: pieces created, moved between lanes, scripted, given per-platform captions, and deleted, with everything autosaving.

Phase 4 adds the calendar.
