---
name: wiki-update
description: Use when revising existing Roam wiki pages because knowledge has changed, a new piece of information updates or contradicts existing content, or the user wants to direct an edit. Because roam-mcp exposes no update or delete tool, this skill queues changes as `{{[[TODO]]}} … #wiki-change-request` blocks for the user to apply manually in Roam.
---

# Wiki Update

Revise wiki content. Always cite the source of new information. Always sweep downstream pages. Always log to today's daily note.

## ⚠ Mutation constraint

**roam-mcp has no update or delete tool.** Existing block strings cannot be changed by this skill. Instead, every proposed change becomes a `{{[[TODO]]}} Revise: …` block tagged `#wiki-change-request`, queued under the affected page. The user applies the change manually in the Roam UI and then marks the TODO done. `roam_search_by_status("TODO")` surfaces the pending queue at any time.

If a future MCP server exposes write transactions, this skill should be updated to apply changes directly. For now, the queue is the audit trail.

## Pre-condition

Locate the wiki:
1. `roam_fetch_page_by_title("Wiki Schema")`
2. Fall back to `roam_search_by_text("Wiki Schema")`
3. If neither finds it, tell the user to run `wiki-init` first

## Process

### 1. Identify what to update

The user may provide:
- **Specific page titles** — work through them in order
- **New information** — read `[[Wiki Index]]` to find affected pages, then fetch them
- **A lint report** — open `[[Lint <date>]]` and walk its recommendations item by item

### 2. For each page to update

Fetch the current page via `roam_fetch_page_by_title`. For each proposed change, present the diff to the user before queuing anything:

> **Page:** `[[<title>]]`
> **Block:** `((<uid>))` (capture from the fetch)
> **Current:** `<exact existing block string>`
> **Proposed:** `<replacement string>`
> **Reason:** `<why this change is warranted>`
> **Source:** `<URL, file path, source page title, or description of where this information comes from>`

**Always include `Source:`.** A change request without a source citation creates untraceability — future you won't know why the change was queued.

Ask for confirmation per change. Do not batch-apply without per-change approval.

### 3. Queue the change request

On confirmation, append a TODO block under the affected page (use `roam_create_block(content=…, page=<title>)` or `parent_uid` to nest under a specific section parent):

```
{{[[TODO]]}} Revise ((<uid>)): <one-line summary> #wiki-change-request
  Current:: <exact existing string>
  Proposed:: <replacement string>
  Reason:: <why>
  Source:: <URL or [[Page]] or path>
  Queued:: <today, ordinal>
```

If the change is **additive** (a new claim that doesn't conflict with existing content), you may instead create the new content directly as appended blocks under the relevant section, with a `Source::` attribute block alongside. Note clearly in your report which changes were queued vs. directly appended.

If the change is a **structural rename** of a page (e.g., `Mod: Auth Service` → `Mod: Identity Service`), queue the TODO under both the old and the new page (the new page is created via `roam_create_page` so it exists), and surface the rename to the user — they will need to use Roam's built-in page-rename in the UI for backlinks to update automatically.

### 4. Check for downstream effects

For each affected page, find references:

1. `roam_search_for_tag(<page title>)` → blocks that link to it via `[[…]]` or `#…`
2. `roam_search_by_text(<page title>)` → blocks that mention it textually but may not link

For every page surfaced (resolved via `:block/page → :node/title`):

- Read it via `roam_fetch_page_by_title`
- Does the proposed update change anything that page asserts?
- If yes: flag it explicitly — `[[Other Page]] may also need updating because <reason>`
- Offer to queue a change request on that page too, with the same per-change confirmation

### 5. Contradiction sweep

If the new information contradicts an existing claim, search for the contradicted claim across the whole wiki before queuing:

```
roam_search_by_text("<phrase from contradicted claim>")
```

The same wrong claim may appear in more than one place. Queue change requests on all occurrences, not just the most obvious one.

### 6. Update `[[Wiki Index]]`

If the one-line summary for any updated page changes, queue an update under `[[Wiki Index]]`:

```
{{[[TODO]]}} Revise index entry for [[<page>]] #wiki-change-request
  Current:: [[<page>]] — <old summary>
  Proposed:: [[<page>]] — <new summary>
  Source:: <where new summary comes from>
```

### 7. Update `[[Wiki Overview]]`

Re-read `[[Wiki Overview]]`. If the change shifts the overall synthesis (new understanding, resolved open question, changed key claim), propose edits using the same queue-and-confirm flow.

You may also append to `Open Questions` directly if the change raises a new question (additive changes don't need a TODO).

### 8. Append to today's daily note

Always — do not ask permission, do not skip:

```
[[Wiki Schema]] update | <list of pages with queued changes> #wiki-log #wiki-update
  Reason:: <brief description of what changed and why>
  Source:: <URL or [[Page]] or description>
  Change requests queued:: <count>
  Directly appended:: <count, if any>
```

### 9. Report to user

- Pages with queued change requests: `[[Page A]]`, `[[Page B]]`
- Pages with additive changes applied directly: `[[Page C]]`
- Downstream pages flagged for review: `[[Page D]]`, `[[Page E]]`
- Lookup query for pending changes anytime: `roam_search_by_status("TODO")` filtered by `#wiki-change-request`

Remind the user: **apply queued changes in the Roam UI**, then mark each `{{[[TODO]]}}` done. The skill cannot do this for them.

## Common Mistakes

- **Queuing without a Source::** — change requests without provenance rot fast. Always include where the new information came from.
- **Skipping the downstream sweep** — a queued change on one page that contradicts an unqueued page creates silent inconsistency once the user applies the first one.
- **Skipping the daily-note log** — the daily-note log is the only audit trail.
- **Batch-queuing without per-change confirmation** — show each diff individually. The user may accept some changes and reject others.
- **Trying to delete content via `roam_create_block`** — there is no delete. Mark obsolete content with `Status:: obsolete` (an attribute block) and reference the corrected page.
