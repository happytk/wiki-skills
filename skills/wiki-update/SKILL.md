---
name: wiki-update
description: Use when revising existing Roam wiki pages because knowledge has changed, a new piece of information updates or contradicts existing content, or the user wants to direct an edit. Applies edits in place using roam_update_block / roam_delete_block / roam_move_block once the MCP entry has X-Roam-Mutate enabled; falls back to a TODO change-request queue when mutation is not available.
---

# Wiki Update

Revise wiki content. Always cite the source of new information. Always sweep downstream pages. Always log to today's daily note.

## Mutation gate

The five mutation tools — `roam_update_block`, `roam_delete_block`, `roam_move_block`, `roam_rename_page`, `roam_delete_page` — are **hidden by default** in roam-mcp. To enable them, the MCP entry that targets your Roam graph must include the header `X-Roam-Mutate: true`. Example `.mcp.json` snippet:

```json
{
  "mcpServers": {
    "roam": {
      "type": "http",
      "url": "http://localhost:8787/mcp",
      "headers": {
        "X-Roam-Graph": "<graph>",
        "X-Roam-Token": "roam-graph-token-...",
        "X-Roam-Mutate": "true"
      }
    }
  }
}
```

**At the start of every wiki-update run**, probe whether mutation is available — try a no-op call to a known-existing block (or check if `roam_update_block` is in your tool list). If the tools aren't visible:

- Tell the user to add `X-Roam-Mutate: true` to the MCP entry and reload (`launchctl bootout` + `launchctl bootstrap` if running via LaunchAgent), OR
- Fall back to **legacy queue mode**: queue every change as a `{{[[TODO]]}} Revise: … #wiki-change-request` block under the affected page (the v2.0 behavior). Make the fallback explicit in your report so the user knows mutation was unavailable.

## Pre-condition

Locate the wiki:
1. `roam_fetch_page_by_title("Wiki Schema")`
2. Fall back to `roam_search_by_text("Wiki Schema")`
3. If neither finds it, tell the user to run `wiki-init` first

Read the schema for the **`Language::`** (default Korean), conventions, and mutation policy.

## Language policy

All prose this skill writes — `Reason::` child blocks, `Source::` notes, `Updated by::` tags, downstream-effect explanations, daily-note log lines, and the new content of any `roam_update_block` call that REPLACES wiki body — must be in the wiki's `Language::`. The diff display you show the user (Current / Proposed / Reason / Source) is also in the wiki language, with source-original text appearing only when the `Current` or `Proposed` IS source-language verbatim (e.g., editing a `Raw Text::` block to fix a transcription error). Direct quotations from sources are preserved via `((uid))` citations.

## Process (mutation enabled)

### 1. Identify what to update

The user may provide:
- **Specific page titles** — work through them in order
- **New information** — read `[[Wiki Index]]` to find affected pages, then fetch them
- **A lint report** — open `[[Lint <date>]]` and walk its recommendations item by item

### 2. For each candidate change, present the diff

Fetch the current page via `roam_fetch_page_by_title`. For each proposed change, show:

> **Page:** `[[<title>]]`
> **Block:** `((<uid>))` (capture from the fetch)
> **Current:** `<exact existing block string>`
> **Proposed:** `<replacement string>`
> **Reason:** `<why this change is warranted>`
> **Source:** `<URL, file path, source page title, or description of where this information comes from>`
> **Action:** `update | delete | move | append`

**Always include `Source:`.** An edit without a source citation creates untraceability — future you won't know why the change was made.

Ask for confirmation **per change**. Do not batch-apply without per-change approval.

### 3. Apply the change in place

On confirmation, dispatch on the action:

| Action | Tool | Notes |
| --- | --- | --- |
| `update` | `roam_update_block(uid=<uid>, content=<proposed>)` | Replaces the block string. The `X-Roam-Ai-Tag` header still governs whether `#ai` is appended. |
| `delete-block` | `roam_delete_block(uid=<uid>)` | **Destructive** and cascades through all descendants. Confirm twice when the block has children — show the child count. |
| `move` | `roam_move_block(uid=<uid>, parent_uid=<new>, order=<n>)` or `(uid, page=<title>, order)` | Use to relocate a block to the right section. `order` is 0-indexed; omit to append last. |
| `append` | `roam_create_block(content=<new>, page=<title>, parent_uid=<section>)` | Pure addition; not really an "update" but reuse the same confirm flow when proposing additions next to existing content. |
| `rename-page` | `roam_rename_page(title=<old>, new_title=<new>)` (or pass `uid` instead of `title`) | Roam's ref index re-resolves automatically — every `[[old]]` reference now points at the renamed page. Confirm once; the change is graph-wide. |
| `delete-page` | `roam_delete_page(title=<title>)` (or `uid=<uid>`) | **Destructive — wipes the entire page including all blocks under it.** Always run a pre-check: `roam_search_for_tag(<title>)` for inbound refs, list every page that references it, and confirm twice (or three times if the page has 5+ inbound refs). Returns `dry_run: true` on success — surface that flag to the user. |

**Capture the citation source.** When updating a content block based on a new source, also append a child block under the updated block recording the citation:

```
Source:: <URL or [[Page]] or path>
Updated by:: wiki-update <today, ordinal>
```

(This is metadata about the *edit*, not a `Updated::` attribute on the page — Roam's `:edit/time` already records the edit timestamp.)

**Page rename safety.** Before `roam_rename_page`, scan inbound refs (`roam_search_for_tag(<old title>)`) and report the count to the user (e.g., "23 blocks across 7 pages reference this title — they will all auto-update"). Roam's ref index handles the rewiring; no follow-up edits to those blocks are needed. Update only the wiki-managed entries that contain the title as a string (e.g., the `[[Wiki Index]]` one-liner) using `roam_update_block` if those entries are out of sync after the rename.

### 4. Check for downstream effects

For each affected page, find references:

1. `roam_search_for_tag(<page title>)` → blocks that link to it via `[[…]]` or `#…`
2. `roam_search_by_text(<page title>)` → blocks that mention it textually but may not link

For every page surfaced (resolved via `:block/page → :node/title`):

- Read it via `roam_fetch_page_by_title`
- Does the change require an edit there too? (e.g., a stale claim now contradicted)
- If yes: present a per-change diff using the same confirm-and-apply flow. Each downstream page goes through step 2-3 individually.

### 5. Contradiction sweep

If the new information contradicts an existing claim, search for the contradicted claim across the whole wiki before applying:

```
roam_search_by_text("<phrase from contradicted claim>")
```

The same wrong claim may appear in more than one place. Apply per-block edits to all occurrences, not just the most obvious one. Per-change confirmation still applies.

### 6. Update `[[Wiki Index]]`

If the one-line summary for any updated page changed, find the index entry block (depth 2 child under the relevant category) and `roam_update_block` it directly. Don't append a sibling — that creates two entries for one page.

### 7. Update `[[Wiki Overview]]`

Re-read `[[Wiki Overview]]`. If the change shifts the synthesis:

- A claim under `Current Understanding` is now wrong → `roam_update_block` it (or `roam_delete_block` if simply obsolete)
- An open question was resolved → `roam_delete_block` the question and add a one-line resolution under `Current Understanding`
- A new question opened → append a child under `Open Questions` (additive)

Use the same per-change confirm flow.

### 8. Append to today's daily note

Always — do not ask permission, do not skip. Multi-value lists are parent + children (one block per page); the root block string holds only a short headline.

```
[[Wiki Schema]] update | <one-line headline> #wiki-log #wiki-update
  Reason:: <brief description of what changed and why>
  Source:: <URL or [[Page]] or description>
  Edits applied::
    update:: <count>
    delete:: <count>
    move:: <count>
    append:: <count>
  Pages edited::
    [[Page A]]
    [[Page B]]
  Downstream pages also edited::
    [[Page D]]
    [[Page E]]
```

### 9. Report to user

- Pages with applied edits: `[[Page A]]`, `[[Page B]]` (with edit counts)
- Downstream pages also touched: `[[Page D]]`, `[[Page E]]`
- Anything that fell through to legacy queue mode (mutation tool failure mid-run): list the queued TODOs

## Process (legacy queue mode — mutation NOT available)

Same as v2.0 behavior:

1-2. Identify candidates and present diffs (steps 1-2 above are unchanged).

3. **On confirmation**, append a TODO block under the affected page (do NOT call mutation tools — they're not available in this mode):

```
{{[[TODO]]}} Revise ((<uid>)): <one-line summary> #wiki-change-request
  Current:: <exact existing string>
  Proposed:: <replacement string>
  Reason:: <why>
  Source:: <URL or [[Page]] or path>
  Queued:: <today, ordinal>
```

4-5. Downstream sweep + contradiction sweep produce more TODOs, not edits.

6. Index/overview changes also queue as TODOs.

7. Daily-note log uses `Change requests queued::` instead of `Edits applied::`.

8. Final report tells the user **mutation mode was unavailable**, lists every queued TODO, and reminds them to apply via Roam UI (or enable `X-Roam-Mutate: true` and re-run).

`roam_search_by_status("TODO")` filtered by `#wiki-change-request` lists pending edits. This mode is the fallback only — prefer the mutation flow when the gate is open.

## Common Mistakes

- **Skipping the mutation probe** — assuming the tools are available without checking. The probe step at the top of the run determines whether you take the in-place path or the queue path.
- **Updating without `Source::` metadata** — the wiki loses its auditability. Always append `Source::` and `Updated by::` child blocks under the edited block (or carry the citation in the new block content itself).
- **`roam_delete_block` without checking children** — deletes cascade. Always confirm twice when the block has descendants; show the child count in the confirmation prompt.
- **Skipping the downstream sweep** — a successful in-place edit that contradicts an un-edited page creates silent inconsistency. Always run the sweep.
- **Skipping the daily-note log** — the daily-note log is the only audit trail.
- **Batch-applying without per-change confirmation** — show each diff individually. The user may accept some changes and reject others.
- **Comma-joining multi-page lists** — `Pages edited:: [[a]], [[b]]` defeats the outliner. Use a parent attribute block with one child per page.
- **Trying to rename pages via mutation tools** — there is no `roam_rename_page`. Direct the user to Roam's built-in rename in the UI.
