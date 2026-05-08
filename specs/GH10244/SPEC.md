# Shift-click bulk selection + bulk delete in Warp Drive (GH-10244)

## Summary

Warp Drive (saved objects: workflows, notebooks, env-var collections, Plans)
lacks multi-select. Users want familiar file-list ergonomics: Shift-click for
contiguous range, Cmd/Ctrl-click for individual toggle, plus a compact
selection bar with bulk Delete and bulk Move. This spec defines the V1
selection model, the bulk action set, keyboard parity, and how selection
interacts with filtering.

## Problem

- Drive items (especially AI-generated Plans) accumulate quickly.
- The only way to delete N items today is one at a time.
- There is no way to move multiple items into a folder in one gesture.
- The interaction model differs from every other "list of saved things" UI
  the user already knows.

## Goals

- Multi-select via Shift-click (range), Cmd/Ctrl-click (toggle).
- `Cmd+A` to select all visible/filtered, `Esc` to clear.
- Compact selection bar at ≥2 selected with Delete, Move to…, and Cancel.
- Bulk Delete with confirmation; bulk Move with folder picker.
- Selection persists across filter narrowing so a search-then-select-all flow
  works correctly.
- Keyboard parity for arrow navigation, `Shift+Arrow` extension, `Space`
  toggle, `Enter` open.

## Non-Goals

- No bulk edit of metadata (rename, retag) in V1; deferred to V1.5.
- No server-side admin bulk tools.
- No in-app undo window for Plan deletion (Plans are higher-stakes than chat
  history; undo is intentionally NOT mirrored from #10457).
- The single-item behaviors of **open, run, and share** are unchanged in V1 (open via `Enter`/double-click, run via existing run affordance, share via existing share affordance). Plain click does not trigger open/run/share. See B1 below for the **explicit, authoritative reconciliation** between plain click's existing single-row selection effect and the new anchor-setting effect; if the prose under Non-Goals here ever appears to conflict with B1, B1 wins.
- Not the same flow as PR #10457 (chat history bulk delete) — Drive selection
  is its own model.

## Behavior Contract

### B1. Single-click selects one row

**Authoritative plain-click contract.** A plain (unmodified) left click on a
Drive row does the following, in order, and nothing else:

1. Clears any existing multi-selection (i.e., the selection set is reset to
   contain only the clicked row).
2. Sets the clicked row as the single-row selection.
3. Sets the clicked row as the new "anchor" for future Shift extension.
4. Does **not** open the item.
5. Does **not** run the item.
6. Does **not** share the item.
7. Does **not** show the selection bar (size = 1; the bar requires ≥2 per B6).

**Reconciliation with prior single-row behavior.** Pre-V1 Drive already
treated plain click as a selection-and-focus gesture on the row (the row
became visually selected and arrow-key navigation continued from there).
V1 does **not** change what plain click does to the visible selection state
of a single row — the row is still selected and focused exactly as before.
V1 only adds two non-visible side effects on the same gesture:

- (a) Any rows that were members of a prior multi-selection set are now
  removed from the set. (Pre-V1 there was no multi-selection set, so this
  side effect had nothing to clear.)
- (b) The clicked row is recorded as the anchor for use by subsequent
  Shift-click and Shift+Arrow extension.

Both (a) and (b) are non-visible internal-state effects only. Plain click
does **not** trigger open, run, or share — those continue to use their
pre-V1 affordances (Enter / double-click / explicit run / explicit share
buttons). Therefore, from the user's standpoint, plain click on a row
behaves identically to pre-V1 except that prior multi-selection is cleared
(which only matters when the user has built a multi-selection in this
session). This is the single authoritative contract for plain click;
any earlier or later prose that appears to disagree is superseded by this
section.

### B2. Cmd/Ctrl-click toggles a single row

`Cmd-click` on macOS / `Ctrl-click` on Windows/Linux toggles the clicked row
in or out of the selection set. Anchor remains at the previous anchor; if no
anchor exists, the toggled row becomes the anchor.

### B3. Shift-click extends a contiguous range

`Shift-click` selects every row from the anchor through the clicked row,
inclusive. Anchor stays put on subsequent shift-clicks (so multiple
shift-clicks adjust the range from the same anchor). First shift-click with
no prior anchor sets the anchor to the clicked row.

### B4. Cmd/Ctrl+A selects all matching

`Cmd+A` (mac) / `Ctrl+A` (win/linux) selects every row in the currently
visible filtered set. With no filter active, it selects everything in the
current Drive view.

**Focus scoping (REQUIRED — load-bearing for `Cmd/Ctrl+A`).** All
selection-related shortcuts in this spec
(`Cmd/Ctrl+A`, `Esc`, `Space`, `Enter`, `Up`/`Down`, `Shift+Up`/`Shift+Down`,
plain/Shift/Cmd-Ctrl click handlers) only fire when **the Drive row list
itself owns keyboard focus**. The single most important consequence is
that **`Cmd/Ctrl+A` MUST NOT hijack the user's text selection** while
focus is inside the filter / search input or any other text input within
the Drive panel — when focus is inside a text input, `Cmd/Ctrl+A` falls
through to the platform-native "select all text" behavior on that input.
Implementations that route `Cmd/Ctrl+A` to row selection while a text
input owns focus are **non-conforming**.

The shortcuts do **not** intercept input when focus is inside any
text-input surface within the Drive panel — specifically:

- The filter / search input at the top of Drive.
- The folder name input inside the Move-to picker.
- Any inline rename affordance.
- Any modal text input (e.g., a future filter-by-tag input).

Concretely, the row list installs its keymap on a focus context
`WarpDriveRowList`. While focus is in any text input, `Cmd/Ctrl+A` falls
through to its native "select all text" behavior; `Esc` falls through to
its native "dismiss/clear input" behavior; `Space` types a space; `Enter`
submits the input. When the user `Tab`s out of the text input back into the
row list (or clicks a row), `WarpDriveRowList` regains focus and the
selection shortcuts resume firing.

Implementation note: this matches Warp's existing keymap-context pattern
(see verified `app/src/code/editor/` and command-palette focus-context
guards). The row-list keymap MUST be installed on a context distinct from
any descendant text-input contexts.

### B5. Esc clears

`Esc` clears selection and anchor and dismisses the selection bar.

### B6. Selection bar appears at ≥2 selected

When selection size is ≥2, a compact bar pins to the top of the Drive panel
with: `<N> selected · Delete · Move to… · Cancel`. The bar disappears when
selection drops to 0 or 1. At exactly 1 selected, normal single-row affordances
apply (no bulk bar).

### B7. Bulk Delete

Triggered from the selection bar Delete action.

**Hidden-selection authoritative rule (load-bearing).** Bulk Delete
**MUST NOT** proceed when the selection contains items hidden by the
active filter without first showing the user (a) the total count being
deleted, (b) the visible/hidden breakdown, and (c) a `[Show hidden]`
affordance to inspect the hidden subset before confirming. Suppressing,
re-using, or short-circuiting this confirmation — even after the user
previously dismissed a similar modal in the same session — is
**non-conforming**. This rule exists because the visible row list
materially understates the destructive scope when a filter is active;
the confirmation modal is the *only* moment the user can audit the true
scope. The exact copy and affordances per scenario are defined below.

The modal copy depends on whether the current selection includes items
hidden by an active filter:

- **All selected are visible (hidden count = 0):**
  `Delete <N> items? This cannot be undone.` `[Delete] [Cancel]`
- **Some selected are hidden by filter (visible > 0, hidden > 0):**
  `Delete <total> items, including <hidden> not currently visible due to
  filter? This cannot be undone.` followed by an explicit count breakdown
  on its own line: `(<visible> visible, <hidden> hidden by filter)`.
  `[Delete] [Show hidden] [Cancel]`
- **All selected are hidden (visible = 0, hidden > 0):**
  `All <hidden> selected items are hidden by the current filter. Delete
  anyway? This cannot be undone.` `[Delete] [Show hidden] [Cancel]`

The `[Show hidden]` action opens a secondary modal listing the names of every
selected item (visible and hidden), with the hidden subset visually flagged
(e.g. a "hidden by filter" tag). The user MAY deselect individual items from
this secondary view; deselections take effect on dismissal and the primary
confirmation is re-rendered with updated counts.

Other Bulk Delete behavior:

- No 5-second undo window (deliberately differs from chat-history bulk delete).
- Per-item failures are collected and surfaced in a result toast:
  `Deleted X. Y failed.` with a link to view failed items.

#### B7.2. Plan delete recovery model (V1 decision)

The earlier wording deferred Plan recovery to a separate decision. V1
locks in the following recovery model so user-facing copy and
implementation semantics are not ambiguous:

- **Plans (and all other Drive item types) are hard-deleted** when bulk
  Delete is confirmed. There is no in-app undo and no client-visible soft
  tombstone in V1.
- The bulk-delete confirmation copy uses the exact phrase **"This cannot
  be undone."** (already present in B7) — this is now a load-bearing
  promise that V1 implementations MUST honor. Any future soft-tombstone
  feature that materially changes recovery MUST also update this copy.
- **Server-side retention** is out of V1 user-facing scope. If the
  backend already retains soft tombstones for compliance / admin
  recovery, that capability is **not** surfaced in any V1 UI, telemetry
  field, or copy string. (Tracked under Open Questions for V1.5 product
  decision on whether to expose admin recovery.)
- This decision is intentionally stricter than chat-history bulk delete
  (#10457): chat history has a 5-second undo; Drive has none. Users
  pasting Plan IDs into bug reports / Linear are protected by the
  confirmation modal alone.

#### B7.1. Hidden-selection invariant (security / safety)

Bulk destructive operations on selections that include items hidden by an
active filter MUST display the hidden count in the confirmation. Implementations
MUST NOT proceed with a bulk delete when `hidden_count > 0` without first
showing the breakdown defined above. Skipping the breakdown — even when the
user has previously dismissed it for the same selection — is non-conforming.

This invariant exists because the visual list does NOT reflect the true scope
of the destructive action when a filter is active; the confirmation is the
sole moment the user can audit what they are about to delete.

### B8. Bulk Move to folder

Triggered from the selection bar Move to… action. Opens the existing folder
picker.

When the current selection includes items hidden by filter, the folder picker
header MUST display the same count breakdown wording used in B7
(`Move <total> items, including <hidden> not currently visible due to filter`
plus `(<visible> visible, <hidden> hidden by filter)`) and offer the same
`[Show hidden]` affordance. Move is less destructive than Delete but must
still surface the scope.

- Items already in the destination folder are a silent no-op (not an error).
- Mixed-type selections are allowed; folder picker is the same.

### B9. Selection survives filtering

Items selected before a filter/search narrows the visible set REMAIN
selected even when filtered out of view. The selection bar shows
`<visible> + <hidden> selected` so the user can see the hidden count and
decide whether to clear before acting. Bulk actions still apply to all
selected items, visible or not, subject to the B7 / B8 confirmation
requirements.

### B10. Keyboard navigation

- `Up` / `Down` move focus (single-select replaces previous selection).
- `Shift+Up` / `Shift+Down` extend selection contiguously from anchor.
- `Space` toggles the focused row's membership in the multi-selection.
- `Enter` opens the focused item.
- `Cmd+A`, `Esc` as defined above.

## Settings / API surface

No new user-facing settings. Internal additions:

- `WarpDriveSelection` model holding `{ anchor: Option<RowId>, set: HashSet<RowId> }`.

### Server-side authorization and bounded work (REQUIRED for bulk paths)

**Scope.** This section applies equally to (a) a new server-side batch
endpoint, if introduced, and (b) the existing per-item endpoints invoked
in a loop. There is no path — neither client-fanout nor batch — by which
bulk operations may bypass the rules below.

**Item-level authorization (REQUIRED).** Bulk delete and bulk move MUST
preserve the per-item authorization and ownership checks already
enforced by the single-item endpoints. A batch endpoint that "trusts the
batch" (i.e., authorizes once per request and then operates on every id)
is **non-conforming**. Specifically:

- **Per-item auth check.** For each item id in the request, the server
  re-runs the same ownership / team-membership / role check that the
  single-item endpoint performs. There is no "trust-the-batch" shortcut.
- **Partial-deny semantics.** If a subset of ids fail the auth check, the
  server processes the authorized subset and returns a structured result
  enumerating `succeeded[]`, `denied[]` (with a non-leaking reason code
  such as `not_found_or_unauthorized`), and `failed[]` (other errors).
  The client surfaces denials in the same failed-items toast surface
  defined in B7.
- **No id-leak.** The denial reason MUST NOT distinguish "item exists but
  you don't own it" from "item does not exist" — both collapse to
  `not_found_or_unauthorized` to avoid leaking the existence of items
  the caller cannot see.
- **Cross-team / cross-workspace.** A bulk delete or move request that
  includes any id outside the caller's accessible scope MUST treat that
  id as `not_found_or_unauthorized`; the request is **not** rejected
  wholesale (so a single bad id does not nuke the whole batch).
- **Audit log.** Each per-item delete or move operation is audited
  individually as if it had been a single-item call — there is no batch
  audit row that hides the per-item subjects.

### Client-side batching constraints (REQUIRED)

When the client falls back to per-item calls (no batch endpoint
available), the client iterator is bounded by ALL of the following — not
just a chunk size:

| Constraint | V1 limit | Rationale |
|---|---|---|
| Total selection size | hard cap **2000** items per bulk action | Beyond this, the UI shows an "Selection too large — narrow the filter and try again." error and refuses the action. Prevents a stray `Cmd+A` over a huge Drive from generating runaway traffic. |
| Chunk size | **200** ids per request when a batch endpoint is available; **1** per request when falling back to per-item endpoints | Matches existing per-item endpoint shape. |
| Concurrency (parallelism) | at most **4** concurrent in-flight requests | Caps client-driven server load and keeps UI responsive. |
| Inter-request rate | at most **20 requests / second** sustained, smoothed via a token-bucket limiter on the client | Protects shared backends and avoids per-user IP throttling. |
| Backoff | on `429` or `5xx`, exponential backoff with jitter starting at 250ms, max 4 retries per id | Standard polite-client pattern. |
| Cancellation | the operation is cancellable from the result toast and from `Esc` in the progress indicator; in-flight ids complete; queued ids are dropped | User remains in control. |

**Bounded total work (applies to BOTH paths).** The 2000-item hard cap,
4-concurrent-in-flight limit, 20 req/s rate cap, and exponential backoff
defined above are NOT specific to the per-item fallback. They are
hard caps on **total bulk-action server work**, applied identically when
a batch endpoint is used:

- The total selection size cap of 2000 applies to the batch payload as
  well — a batch request with >2000 ids is refused client-side BEFORE
  hitting the wire and is also rejected server-side as a defense-in-depth.
- When a batch endpoint accepts up to 200 ids per call, the same 4-way
  concurrency and 20 req/s sustained-rate caps apply to the *batched*
  call sequence (so the maximum theoretical throughput is identical to
  the per-item path: bounded concurrency × bounded rate).
- Backoff on `429` / `5xx` applies to the batch request as a whole and
  retries the entire chunk's ids that have not yet succeeded server-side.
- The cancellation contract from the table applies in both paths: in
  flight requests complete, queued chunks are dropped, and partially
  succeeded ids are surfaced in the result toast.

A server-side batch endpoint, when available, supersedes the per-item
loop for wire efficiency but MUST still enforce both the per-item
authorization rules and the total-work caps above. Chunking alone is
**not** sufficient; the work caps are an independent, additive
constraint.

## Acceptance Criteria

- A1: Shift-click between two rows selects every row in between, inclusive,
  and the anchor remains at the original anchor row.
- A2: Cmd/Ctrl-click toggles individual rows in/out without disturbing the
  rest of the set.
- A3: `Cmd+A` selects all currently visible rows when a filter is active and
  all rows otherwise.
- A4: `Esc` clears selection and dismisses the selection bar.
- A5: Selection bar appears when selection is ≥2 and disappears at 0 or 1.
- A6: Bulk delete shows the confirm modal and removes all selected items;
  per-item failures surface in a result toast.
- A6a: When all selected items are visible, bulk-delete confirmation reads
  `Delete <N> items? This cannot be undone.` with no breakdown line.
- A6b: When some selected items are hidden by filter, bulk-delete confirmation
  reads `Delete <total> items, including <hidden> not currently visible due
  to filter? This cannot be undone.` AND displays the breakdown line
  `(<visible> visible, <hidden> hidden by filter)`. The dialog MUST expose a
  `[Show hidden]` action.
- A6c: When ALL selected items are hidden (visible = 0), bulk-delete
  confirmation reads `All <hidden> selected items are hidden by the current
  filter. Delete anyway? This cannot be undone.` and exposes `[Show hidden]`.
- A6d: `[Show hidden]` opens a secondary view listing names of every selected
  item, with hidden items flagged. The user can deselect individual items
  from this view; on dismissal the primary confirmation re-renders with
  updated counts.
- A6e: Implementations MUST NOT proceed with bulk delete when `hidden_count > 0`
  without first showing the breakdown defined above (B7.1 invariant).
- A7: Bulk move opens the folder picker and moves all selected items.
- A7a: When the move selection includes hidden items, the folder picker
  header surfaces the same count breakdown and `[Show hidden]` affordance.
- A8: Selection persists when the user types into the filter / clears the
  filter.
- A9: Keyboard navigation full path: arrow movement, `Shift+Arrow` extension,
  `Space` toggle, `Enter` open.
- A_focus_scoping_text_input: While the Drive filter input has focus,
  `Cmd/Ctrl+A` selects the input text (does NOT select all rows); `Esc`
  clears the filter input (does NOT clear row selection); `Space` and
  `Enter` route to the input. After tabbing back to the row list,
  `Cmd/Ctrl+A` again selects all matching rows.
- A_server_auth_per_item: A bulk delete request containing 5 ids the
  caller owns and 2 ids the caller does NOT own returns 5 successes
  and 2 `not_found_or_unauthorized` denials; the failed-items toast
  surfaces the 2 denials. The 5 owned items are deleted; the 2
  non-owned items are unaffected.
- A_client_chunking_caps: A `Cmd+A` over a Drive containing 2500 items
  (no filter active) refuses to start the bulk action and surfaces
  "Selection too large — narrow the filter and try again." A 1500-item
  selection is accepted; the client issues at most 4 concurrent
  in-flight requests at any time and at most 20 requests per second
  sustained.
- A_plan_no_in_app_undo: After confirming bulk delete on a selection
  containing Plans, no undo affordance appears anywhere in the Drive UI;
  the result toast shows `Deleted X. Y failed.` only.

## Implementation Pointers

Verified paths (via `git ls-files`):

- Drive surfaces:
  - `app/src/settings_view/warp_drive_page.rs` — settings-view list of saved
    objects.
  - `app/src/search/command_palette/warp_drive/data_source.rs` — palette data
    source for Drive items.
  - `app/src/search/command_palette/warp_drive/mod.rs` and the
    `*_search_item.rs` siblings for typed rows.
  - `app/src/integration_testing/warp_drive/mod.rs`,
    `app/src/integration_testing/warp_drive/assertion.rs` — existing test
    harness for Drive behavior.

Likely change shape:

1. New module `app/src/settings_view/warp_drive/selection.rs` (new module)
   holding the selection state, anchor logic, and reducer for click/keyboard
   events.
2. Wire selection into the row click handler and keyboard input on the Drive
   list view.
3. Add the selection bar component above the list with Delete / Move to… /
   Cancel.
4. Bulk delete / move plumb through existing per-item APIs; chunked client
   loop bounded to 200 items.
5. Persist selection across filter changes by keeping the selection set keyed
   on stable row IDs, not on visible position.

## Tests

- T1: Shift-click selects an inclusive contiguous range from anchor.
- T2: Cmd/Ctrl-click toggles a single row without affecting others.
- T3: Cmd-click then Shift-click compound — Shift extends from existing
  anchor, not from the most recent Cmd-click row.
- T4: `Cmd+A` with a filter selects only visible filtered rows.
- T5: `Esc` clears selection and dismisses the selection bar.
- T6: Selection bar appears at 2 selected, disappears at 1 and at 0.
- T7: Bulk delete with a partial-failure mock surfaces the failed items.
- T_confirm_visible_only: With no filter active (or filter active but
  hidden_count = 0), the bulk-delete confirmation reads
  `Delete <N> items? This cannot be undone.` and does NOT include a
  breakdown line.
- T_confirm_with_hidden: With a filter active and hidden_count > 0 (and
  visible_count > 0), the confirmation reads
  `Delete <total> items, including <hidden> not currently visible due to
  filter? This cannot be undone.` and includes a breakdown line of the form
  `(<visible> visible, <hidden> hidden by filter)`.
- T_confirm_all_hidden: With visible_count = 0 and hidden_count > 0, the
  confirmation reads `All <hidden> selected items are hidden by the current
  filter. Delete anyway? This cannot be undone.`
- T_show_hidden_button: Clicking `[Show hidden]` opens a secondary view
  listing the names of every selected item, with hidden items visually
  flagged.
- T_deselect_from_hidden_view: From the secondary view the user can
  deselect individual items; on dismissal the primary confirmation
  re-renders with updated counts (and reverts to the no-hidden wording if
  the deselections eliminate the hidden subset).
- T_move_with_hidden: Bulk Move with hidden_count > 0 surfaces the same
  count breakdown and `[Show hidden]` affordance in the folder picker
  header.
- T8: Bulk move into an existing folder; items already in destination are
  silent no-ops.
- T9: Selection persists after typing into the filter and after clearing it.
- T10: Keyboard nav full path — arrows, `Shift+Arrow`, `Space`, `Enter`,
  `Cmd+A`, `Esc`.
- T_focus_scope_filter_input: Selection shortcuts do NOT fire while the
  filter input owns focus; they DO fire after focus returns to the row
  list.
- T_focus_scope_move_picker_input: Selection shortcuts do NOT fire while
  the Move-to picker's text input owns focus.
- T_server_auth_partial_deny: Mock server denies a subset of ids;
  client surfaces `denied[]` items in the failed-items toast under the
  reason code `not_found_or_unauthorized`; succeeded items are removed
  from the local view.
- T_server_auth_no_id_leak: Mock server denial reason is identical for
  "item exists, caller unauthorized" and "item does not exist"; client
  treats them identically and shows the same generic copy.
- T_client_total_cap_blocks: Selection of 2001 items refuses to start
  the bulk action and surfaces the size-cap toast.
- T_client_concurrency_cap: With 1000 selected ids and per-item
  endpoints, at no point are more than 4 requests in flight; sustained
  rate stays under 20 req/s.
- T_client_backoff_on_429: A mocked 429 response triggers exponential
  backoff with jitter starting at 250ms; the id is retried up to 4
  times before being surfaced as failed.
- T_plan_hard_delete_no_undo: After confirming bulk delete on a Plan
  selection, no undo affordance appears; result toast renders the
  default failure copy only.

## Open Questions

- Should bulk delete for Plans support a chat-history-style 5-second in-app
  undo? Plans are more precious than chat messages. Recommendation: NO
  in-app undo by default; instead, record a 30-day server-side soft
  tombstone that admins can recover. Final decision deferred to engineering.
- Should mixed-type selections (workflow + notebook + Plan) all share the
  same bulk-delete confirm copy, or should each type get its own count
  breakdown in the modal? Recommendation: single combined count for V1;
  itemized breakdown if user testing surfaces confusion.

## Telemetry

Extend the existing Drive action event with two fields:

- `bulk: bool` — whether the action was a bulk action (≥2 items).
- `count: u32` — number of items affected.

No new event names. Same applies to `delete` and `move` action events.
