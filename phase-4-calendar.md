# Phase 4: Calendar

## What this phase delivers

A month calendar showing scheduled content, where dragging a piece onto a day sets both its `scheduled_for` date and its status to Scheduled.

No new tables and no new API routes. This phase reuses the content routes from phase 3 and the same drag-and-drop library as the board, applied to a different shape.

---

### P4.1 - Build the calendar shell and month grid

Requires: P3.7
Halts: no

OBJECTIVE

A month grid on `/calendar` spanning whole weeks, with month navigation and a Today control.

PROMPT

```text
--- PROMPT P4.1 ---
Read SPEC.md section "Calendar".

1. Create CalendarView as a client component holding the calendar state.

   - It keeps a reference date, starting at today.
   - A "Today" button resets the reference date to today.
   - Left and right chevron buttons step one month back or forward.
   - A label reads the month and full year, for example "August 2026".
   - It fetches GET /api/content once on mount and holds the result in
     state. Show a skeleton placeholder while that request is in flight.

2. Create MonthView, rendering the grid.

   - Weeks start on Monday. Use date-fns with a Monday week start
     consistently; a mismatch here shifts every date by one column.
   - The grid spans from the start of the week containing the first of
     the month to the end of the week containing the last of the month,
     so it is always full weeks: up to six rows of seven columns.
   - Rows have a minimum height but grow to fill available space, and
     the grid scrolls vertically if it overflows.
   - A weekday header row labels the seven columns.
   - Days outside the displayed month are visually muted.
   - The week row containing today gets a subtle background tint.
   - Today's date number is shown inside a filled circle.

3. Render CalendarView on the /calendar page.

Do not render content on the grid yet, and do not add drag-and-drop.
Later tasks add both.
--- END PROMPT P4.1 ---
```

EXPECTED RESULT

Two new components and an updated calendar page.

VERIFICATION

1. Request `/calendar` and confirm 200, with the current month and full year present in the body and Monday first in the weekday header.
2. Grep every week boundary calculation in the calendar components and confirm each passes the same Monday week-start option to date-fns. A single inconsistent call shifts every date by one column.
3. Confirm the rendered grid contains a multiple of seven day cells, between 28 and 42.
4. `npm run build` exits 0.

GATE: Check 2 finds no inconsistent week start.

---

### P4.2 - Show scheduled content on the grid

Requires: P4.1
Halts: no

OBJECTIVE

Scheduled pieces rendered inside their day cell, capped at three with an overflow count, and each day made a drop target.

PROMPT

```text
--- PROMPT P4.2 ---
Read SPEC.md section "Calendar".

1. Create DraggableContentCard, rendering a piece as a compact card
   showing its title, with a grab cursor. Clicking it navigates to
   /drafts/[id]. Accept a prop that renders it as a non-interactive
   copy, for use by the drag overlay in the next task.

2. In each day cell of MonthView:

   - Select the pieces whose scheduled date falls on that day, comparing
     in local time rather than UTC.
   - Sort them by scheduled time, earliest first.
   - Render at most three cards. If more exist, render a "+N more"
     count beneath them.
   - Make the cell a drop target keyed by its date in "yyyy-MM-dd" form.
     The key format matters: the next task reads the date back out of it.
   - Highlight the cell while a dragged card is held over it.

Do not add DndContext, dragging, or any save behavior. The next task adds
those.
--- END PROMPT P4.2 ---
```

EXPECTED RESULT

One new component and an updated `MonthView`.

VERIFICATION

1. Schedule four pieces on the same date via `PATCH /api/content/[id]`, then request `/calendar` and confirm three cards render for that day plus a "+1 more" count.
2. Query `scheduled_for` for a scheduled piece and confirm the day cell it renders into matches that date in local time, not the day before. A piece appearing one day early means the comparison is running in UTC.
3. Read the day-cell selection and confirm the date comparison uses local-time components. Quote the lines.
4. Confirm each day cell's droppable id is formatted `yyyy-MM-dd`. The next task parses it back.

GATE: Checks 2 and 4 pass.

---

### P4.3 - Add drag to reschedule

Requires: P4.2
Halts: no

OBJECTIVE

Dropping a card on a day writes both the new `scheduled_for` date and a status of `scheduled`, rolling both back together on failure.

PROMPT

```text
--- PROMPT P4.3 ---
Read SPEC.md section "Calendar".

1. In CalendarView, wrap the grid in a DndContext using a PointerSensor
   with an 8px activation distance, so clicking a card to open it is not
   mistaken for a drag.

2. Add a DragOverlay rendering a slightly rotated copy of the card being
   dragged.

3. On drop:

   - The drop target's id is the day in "yyyy-MM-dd" form. Build a Date
     from its year, month, and day components rather than parsing the
     string directly, for the same timezone reason as the editor's date
     field. Time of day is not part of the drop; the date lands at local
     midnight.
   - Update the piece in local state first, setting both its scheduled
     date and its status to "scheduled".
   - Then PATCH /api/content/[id] with both fields.
   - If the request fails, restore both the previous scheduled date and
     the previous status, and show an error toast. Restoring only one of
     the two leaves the interface disagreeing with the database.
   - Ignore a drop with no target, and a drop on the day the piece is
     already on.

Do not add resizing, multi-day spans, or time-of-day scheduling.
--- END PROMPT P4.3 ---
```

EXPECTED RESULT

An updated `CalendarView`.

VERIFICATION

1. Simulate the drop path: `PATCH /api/content/[id]` with both `scheduledFor` and `status: "scheduled"`, then query the row and confirm both fields changed. A drop that writes only the date looks correct on the calendar and wrong everywhere else.
2. Read the drop handler and confirm it sends both fields in one request. Quote the lines.
3. Read the failure path and confirm it restores both the previous date and the previous status. Quote the lines.
4. Confirm the drop handler builds its Date from the id's year, month, and day components rather than parsing the string.
5. Confirm the handler returns early for a drop with no target and for a drop on the piece's current day.

GATE: Checks 2 and 3 pass. Restoring one field but not the other is the failure this gate exists to catch.

---

## Checkpoint

```text
Commit all current changes with the message "Phase 4: calendar", tag the
commit phase-4-complete, and push both the commit and the tag.
```

## Drift check

```text
Read SPEC.md section "Calendar" and compare it against the running
application. List anything specified there that is missing, and anything
present that the specification does not describe.
```

## What you have now

Two views of the same records: a board organized by stage and a calendar organized by date, either of which can move a piece.

Everything so far is self-contained. Phase 5 connects the application to YouTube, TikTok, and Instagram.
