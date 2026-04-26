---
name: wiki-ingest
description: Use when adding a new source — paper, article, URL, file, transcript, code snippet — to a Roam-backed wiki. Extracts text into block-per-paragraph form so every excerpt earns a stable ((uid)) for citation. Also serves as a queue runner — process items the user (or earlier sessions) dropped under `#wiki-ingest-queue`. One ingest may touch 10-15 Roam pages.
---

# Wiki Ingest

Add a source to the Roam wiki. Read it, discuss with the user, write a source page (with extracted text uploaded as one paragraph per block), update entity/concept pages, run the backlink audit, and update the index, overview, and daily-note log.

The skill operates in three modes:

- **Mode A — Ingest now.** User provides a URL, file path, or pasted text. Run the full ingest flow.
- **Mode B — Add to queue.** User wants to capture a source for later (no immediate ingest). Append a `{{[[TODO]]}} … #wiki-ingest-queue` block and stop.
- **Mode C — Process queue.** No source supplied (or `--queue`). Pull pending `#wiki-ingest-queue` TODOs, present them, ingest the chosen ones in sequence.

## Pre-condition

Locate the wiki:

1. `roam_fetch_page_by_title("Wiki Schema")` — primary discovery.
2. If empty, fall back to `roam_search_by_text("Wiki Schema")` in case the user renamed it.
3. If neither finds it, tell the user to run `wiki-init` first.

Read the schema page in full to learn: domain, allowed source kinds, raw path, index categories, attribute conventions.

## Process — Mode A (ingest now)

### 1. Accept the source

The source can be:
- **File path** — read it directly. If binary or oversized, copy to the local `Raw path::` directory from the schema. If text-extractable, you'll upload its content into Roam in step 5.
- **URL** — use the `browse` skill to fetch it. Save binaries to `Raw path::`; for HTML/article content, hold the extracted text in memory for step 5.
- **Pasted text** — use what was provided directly.

### 2. Read the source in full

Read all content. For long sources, read in sections. Do not skip.

### 3. Classify the source kind

Pick a value for `Source kind::` from the taxonomy below. The kind drives **how `Raw Text::` is chunked** in step 5.

| Source kind | Examples | Raw Text chunking | Notes |
| --- | --- | --- | --- |
| `paper` | arXiv PDF, journal article, ADR, RFC | one block per paragraph; if section headers exist, each section becomes a parent block with paragraph children | PDF binary stays in `raw/`; only extracted text is uploaded |
| `article` | blog post, news, doc page | one block per paragraph | strip HTML chrome (nav, footer, comments) before splitting |
| `transcript` | podcast, video, interview, meeting | **one block per speaker turn**, prefixed with the speaker name; long monologues split at paragraph boundaries within the turn | audio/video without an existing transcript is out of scope — transcribe externally first |
| `code` | source file, config file, snippet | **one block per logical unit** (top-level function, class, struct, exported decl); imports/preamble in one combined block | each block prefixed with `path:line-range` so callers can recover location |
| `book-excerpt` | book chapter, manual section | one block per paragraph | excerpt only — never upload a full book; respect copyright (see [[Wiki Schema]]) |
| `thread` | Twitter/X, HN comment thread, mailing list | **one block per post or reply**, with the author/timestamp as the block prefix; reply chains use parent/child nesting | preserve the reply hierarchy via Roam's outliner |
| `dataset` | CSV/JSON sample, OpenAPI spec, GraphQL schema | structural summary (schema, fields, cardinalities) + a handful of representative rows; **never paste the full file** | full file goes to `raw/`; the wiki page captures shape, not data |
| `notes` | meeting notes, email, Slack/Discord export | one block per paragraph or message; if the source is a conversation, treat it like `transcript` | typically arrives via paste |
| `other` | anything not matching above | LLM judgment, but record the chunking decision | add a `Chunking note::` attribute block explaining the choice |

### 4. Surface takeaways — BEFORE writing anything

Tell the user:
- 3-5 bullet points of key takeaways
- What entities/concepts this introduces or updates
- Whether it contradicts anything already in the wiki. To check, run `roam_search_by_text(<entity>)` and `roam_search_for_tag(<entity>)` for each candidate, fetch the relevant pages, and read.

Ask: **"Anything specific you want me to emphasize or de-emphasize?"**

Wait for the user's response before proceeding.

### 5. Decide the page title

Use a natural Roam title. Examples:
- `Attention Is All You Need` (paper)
- `OpenAI GPT-5 Release Notes` (article)
- `Mod: Auth Service` (codebase, see `codebase.md`)

Check existence first: `roam_fetch_page_by_title(<title>)`. If the page already exists with content, decide whether to add to it or pick a more specific title. Then `roam_create_page(<title>)` for new pages.

### 6. Build the source page

The source page layout — every concept is **its own block**. Multi-value attributes use parent + children, never comma-joined strings.

Top-level blocks (each line below is a separate block created via `roam_create_block`; capture every returned uid for later citation):

```
Type:: #wiki-source
Source:: <URL or absolute file path under Raw path::>
Source kind:: paper | article | transcript | code | other
Tags:: #<topic1> #<topic2>          (1-3 short tags inline is fine; otherwise split below)

Summary
  <paragraph 1 — one block>
  <paragraph 2 — one block>
  <paragraph 3 — one block>          (each cites Raw Text:: via ((uid)) where applicable)

Key Takeaways
  <takeaway 1 — one block>
  <takeaway 2 — one block>
  ...

Entities & Concepts
  [[Entity 1]]
    <one-line note about how this source relates to the entity — one child block>
  [[Concept 2]]
    <one-line note — one child block>
  ...

Relation to Other Wiki Pages
  Connects to [[Other Page]]
    <reason — child block>
  Updates the claim in [[Other Page A]]
    <which claim, and how — child block>

Raw Text
  <paragraph 1 of the source verbatim — one block>
  <paragraph 2 of the source verbatim — one block>
  ...
```

If you have many tags or many topical refs, expand the inline form into parent + children:

```
Tags::
  #<topic1>
  #<topic2>
  #<topic3>
  ...
```

**Construction recipe:**

1. Build a single nested `children` tree and pass it to `roam_create_block(content="Type:: #wiki-source", page=<title>, children=[…])` — but Roam's `Key:: value` attribute index works best when each attribute is its own top-level block, so create the attribute blocks one at a time at depth 0 first, then create the section parents (`Summary`, `Key Takeaways`, `Entities & Concepts`, `Relation to Other Wiki Pages`, `Raw Text`) each with their content as a `children` tree.
2. **Raw Text uploading** — split the source into paragraph-sized units (or logical units for code/transcripts: function, scene, speaker turn). Pass them as the `children` of the `Raw Text` block in a single `roam_create_block` call. Capture the `created_uids` from the response. These uids are the citation handles for the rest of the wiki.
3. Write the `Summary` blocks **after** `Raw Text` is uploaded so summary paragraphs can `((uid))`-cite the original lines. Use `{{embed: ((uid))}}` when the original phrasing carries weight (a quotable claim, a precise number).

**Pre-write validation (mandatory).** Before EVERY `roam_create_block` / `roam_create_page` / `roam_add_todo` call, verify each `content` field — including every `content` inside the `children` tree — is a non-empty string. If a value is `undefined`, `null`, an empty string, or the literal string `"undefined"`/`"null"`, **drop that entry** (don't send it) and surface the gap to the user before proceeding. The auto `#ai` suffix concat means an undefined value renders as a literal `undefined #ai` block in Roam — `roam_delete_block` can clean it up but only when `X-Roam-Mutate: true`, so don't write it in the first place.

**Idempotency on re-ingest.** If you're re-ingesting an updated source, fetch the existing source page first and walk the existing `Raw Text::` children:

1. Match each new paragraph against existing children by the first ~80 chars.
2. **Identical match** → skip; reuse the existing `((uid))` (any wiki page that already cites it stays valid).
3. **Existing block, slightly changed source text** (same paragraph, edited wording) → if `X-Roam-Mutate: true` is enabled, call `roam_update_block(uid=<existing>, content=<new>)` to keep the citation stable. If mutation is disabled, append the new version as a sibling and tag it `Supersedes:: ((<old-uid>))` so readers know which is current.
4. **New paragraph not in the existing tree** → append as a new child of `Raw Text::` (single `roam_create_block` call with `children=[…]` for the new paragraphs only).
5. **Existing block whose source paragraph was deleted upstream** → if mutation is enabled, `roam_delete_block(uid=<existing>)` only when no other wiki page cites that uid (run `roam_search_for_tag(<source title>)` and check). When uncertain, leave it and append a new `Sources note:: this paragraph removed in <date> revision` child block instead.

The mutation calls require the `X-Roam-Mutate: true` header on your MCP entry; without it, fall back to the v2.0 append-only behavior (rule 4 only) and tell the user that re-ingest could not refresh existing blocks in place.

### 7. Update entity and concept pages

For each entity/concept this source touches:

- **Page exists** (`roam_fetch_page_by_title` returns a tree): read it, then add **only what is actually new**. For `Sources::` — if it's currently a parent attribute block, append `[[<source title>]]` as a new child of that existing parent (do NOT create another `Sources::` sibling). If it's a single-value inline attribute (legacy shape) leave it and append a new child `[[<source title>]]` to the page-level Sources:: parent if you can identify one; otherwise leave the legacy block and append the new ref under the relevant section instead. Roam tracks edit time on every block via `:edit/time` — do NOT write `Updated:: <today>` blocks.
- **Page doesn't exist:** `roam_create_page(<entity>)`, then create attribute + section blocks. One concept per block; multi-value attributes use parent + children:

```
Type:: #wiki-entity            (or #wiki-concept)
Sources::
  [[<this source title>]]      (additional source pages will be added here as siblings on future ingests)
Tags::
  #<topic1>
  #<topic2>

Description
  <paragraph 1 — one block, citing ((uid)) into Raw Text:: of the source>
  <paragraph 2 — one block>

Appearances in Sources
  [[<source title>]]
    <one-line note — child block>
    ((uid-of-key-quote))         (separate child block for the citation)

Related Concepts
  [[<related entity 1>]]
    <relationship — child block>
  [[<related entity 2>]]
    <relationship — child block>
```

When the original phrasing matters, `{{embed: ((uid))}}` the source block instead of paraphrasing.

### 8. Backlink audit — do not skip

Roam already maintains the `:block/refs` index, but textual mentions that don't use `[[…]]` won't appear there. For each new entity/concept:

1. `roam_search_for_tag(<entity title>)` → blocks that already reference this entity. Resolve their `:block/page` titles; call this set `R`.
2. `roam_search_by_text(<entity title>)` → blocks that mention the entity textually. Resolve their pages; call this set `T`.
3. Pages in `T - R` mention the entity but don't link to it. For each, decide whether to append a child block under the relevant parent that explicitly references `[[<entity>]]`. Confirm with the user before bulk edits.

This is the step most commonly skipped. A compounding wiki's value comes from bidirectional links.

### 9. Update `[[Wiki Index]]`

Fetch `[[Wiki Index]]`. Find the category block (depth 1) that matches this source's category. Append a child block:

```
[[<page title>]] — <one-line summary> _(ingested <today, ordinal>)_
```

For any new entity/concept pages created in step 7, add those under their categories too. Skip categories that already contain a string-matching entry (idempotency).

### 10. Update `[[Wiki Overview]]`

Fetch `[[Wiki Overview]]`. If this source:
- Introduces a significant concept → append a child block under `Key Entities / Concepts` referencing `[[<concept>]]`
- Shifts the overall understanding → append a child block under `Current Understanding` (do not delete the old one — append-only)
- Raises a new question → append under `Open Questions`

Roam tracks the page's last-edit time via `:edit/time` on the appended blocks; no manual `Updated::` block needed.

### 11. Surface follow-up suggestions and offer to queue

Before logging, look for ingest-worthy leads this source surfaced:

- **Cited references that aren't in the wiki yet** — papers, blog posts, RFCs the source links to
- **Sections you intentionally skipped** (long appendix, supplementary material) that may be worth ingesting later
- **Adjacent sources** worth pulling in (the same author's other paper, a follow-up version, a competing approach)

Present the list and ask: **"Queue any of these to `#wiki-ingest-queue` for later?"**

For each accepted lead, append a queue TODO under the source page (or the user's preferred location):

```
{{[[TODO]]}} Ingest: <one-line description> #wiki-ingest-queue
  URL:: <url or local path, if known>
  Source kind:: paper | article | ...   (best guess; user can edit)
  Reason:: <why this is worth ingesting>
  Suggested by:: [[<this source page>]]
  Queued:: <today, ordinal>
```

If the user adds their own queue items unprompted (e.g., pasting a list of URLs), accept the same format with whatever subset of children they provide. The minimum is `{{[[TODO]]}} <title or url> #wiki-ingest-queue`.

### 12. Log to today's daily note

Append (no `page` arg, defaults to today's daily note). Multi-value lists are parent attribute blocks with one child per item — never comma-joined into a single block string.

```
[[Wiki Schema]] ingest | <source title> #wiki-log #wiki-ingest
  Source:: <URL or path>
  Page:: [[<source title>]]
  Pages updated::
    [[<entity1>]]
    [[<entity2>]]
    ...
  Backlinks added on::
    [[<page A>]]
    [[<page B>]]
    ...
  Queue items added::
    [[<follow-up 1>]]
    [[<follow-up 2>]]
    (or "none")
```

If this run was triggered from a `#wiki-ingest-queue` TODO (Mode C, see below), also append the originating queue uid as a child block: `Closes queue item:: ((<queue-todo-uid>))`.

### 13. Close the queue item (Mode C only)

If this ingest was triggered by a queue TODO, the item must be marked done so it stops appearing in `roam_search_by_status("TODO")` filtered by `#wiki-ingest-queue`. The path depends on whether mutation is enabled.

**With `X-Roam-Mutate: true` (preferred):**

1. Append two audit child blocks under the queue TODO:
   ```
   Ingested as:: [[<source page title>]]
   Ingested on:: <today, ordinal>
   ```
2. Read the queue TODO's current root string (you already have it from the Mode C step 3 listing).
3. Replace `{{[[TODO]]}}` with `{{[[DONE]]}}` in that string and call `roam_update_block(uid=<queue-todo-uid>, content=<new string>)`. The audit children remain attached.

After both calls, the queue entry is closed automatically — no user action needed.

**Without mutation (legacy):** append the audit children only, then tell the user: *"Mark `((<queue-todo-uid>))` as done in Roam UI."* (Without `X-Roam-Mutate`, `roam_update_block` is hidden.)

#### Closing without ingesting

When the user decides a queue item is no longer worth ingesting (source unavailable, content turned out to be irrelevant, etc.) instead of running the full Mode A flow:

1. Append a single child block under the TODO recording the reason:
   ```
   Closed without ingest:: <reason in one line>
   Closed on:: <today, ordinal>
   ```
2. With mutation: `roam_update_block` to flip `{{[[TODO]]}}` → `{{[[DONE]]}}` (same as above).
3. Without mutation: ask the user to mark it done in Roam UI.

This keeps the audit trail (queue history, decisions to skip) without polluting the active TODO list.

### 14. Report to user

- Source page: `[[<title>]]` (with `Raw Text::` containing N blocks, uids captured for citation)
- Entity/concept pages created or updated: <list with `[[Page]]` links>
- Pages that received new `[[<entity>]]` references in the backlink audit: <list>
- `[[Wiki Index]]` and `[[Wiki Overview]]` updated
- Follow-up queue items added: <list, or "none">
- If Mode C: closed queue items — list each `((<uid>))` and whether the close was by ingest or skip; if mutation was unavailable, note which TODOs still need a manual UI check
- Logged to today's daily note as `#wiki-log #wiki-ingest`

## Process — Mode B (add to queue without ingesting)

When the user says "queue this for later" / "add to ingest queue" / passes a URL with `--queue`:

1. Locate the wiki (same Pre-condition).
2. Decide **where to drop the queue TODO** — under the user's daily note (default), or under a related source page if context makes it obvious. Ask if unclear.
3. Append:

   ```
   {{[[TODO]]}} Ingest: <title or "<url>"> #wiki-ingest-queue
     URL:: <url or local path>
     Source kind:: paper | article | ...   (if known; otherwise omit)
     Reason:: <why> (if user provided one)
     Suggested by:: manual
     Queued:: <today, ordinal>
   ```

4. Confirm to the user: queued at `((<uid>))`, run `wiki-ingest` with no source argument to process the queue when ready.
5. Append a `#wiki-log #wiki-queue` block to today's daily note recording the addition.

No source page is created in Mode B. Skip steps 1-12 of Mode A.

## Process — Mode C (process the queue)

When `wiki-ingest` is invoked with no source (or `--queue`):

1. Locate the wiki.
2. `roam_search_by_status("TODO")` then filter the result for blocks whose `:block/refs` include `wiki-ingest-queue`. (If the search returns more than ~50 items, ask whether to narrow by date or `Suggested by::`.)
3. Present the list to the user with: title (root block string), URL, Source kind, Suggested by, Queued. Number the items.
4. Ask: **"Process which? (numbers, comma-separated, or `all`, or `cancel`.)"**
5. For each chosen item, run Mode A starting at step 2 ("Read the source in full"), passing the queue TODO's URL/path as the source. In step 13, close the queue item with the `Ingested as::` / `Ingested on::` children.
6. After the batch, report: processed N items, M still pending, K skipped/cancelled.
7. Append one `#wiki-log #wiki-queue` block to today's daily note summarizing the run.

## Common Mistakes

- **Skipping Raw Text upload** — the wiki loses its "auditable to source" property. Always upload extracted text when feasible.
- **One giant Raw Text block** — defeats `((uid))` citation. One paragraph (or logical unit) per block.
- **Skipping the backlink audit** — set-difference between `roam_search_for_tag` and `roam_search_by_text` is the whole point.
- **Forgetting to capture uids** — `roam_create_block` returns `uid` and `created_uids`; thread them into your subsequent citation blocks rather than re-fetching.
- **Faking sub-bullets with `-`** — Roam treats one block as one idea. Use `children` instead.
- **Comma-joining multi-value attributes** — `Sources:: [[a]], [[b]], [[c]]` defeats the outliner. Write the parent `Sources::` block, then one child block per ref. Same for `Pages updated::`, `Backlinks added on::`, `Tags::` (when more than 2-3).
- **Wrong chunking for the source kind** — pasting a transcript paragraph-per-block instead of speaker-turn-per-block, or dumping a code file as a single Raw Text block. Re-read the Source kind taxonomy before writing.
- **Skipping follow-up queue offer** — every paper/article cites adjacent work. Surface it and let the user decide whether to queue. Otherwise the wiki stops compounding.
- **Closing a queue item without `Ingested as::` / `Closed without ingest::`** — every Mode C run must thread either an "ingested as" or a "closed without ingest" audit child block under the TODO so the audit trail is intact.
- **Telling the user to flip `{{[[TODO]]}}` → `{{[[DONE]]}}` in Roam UI when `X-Roam-Mutate: true` is enabled** — the skill must call `roam_update_block` to do it itself. Manual instructions belong only in legacy mode.
- **Sending undefined/null content to a write tool** — every block content (root and children) must be a non-empty string. Iterating over an entity/source list with empty slots produces blocks like `undefined #ai` that you cannot delete. Validate before each write call; drop missing entries and tell the user.
- **Writing `Updated::` blocks** — Roam tracks edit time via `:edit/time` on every block, exposed in the Roam UI and via `roam_datomic_query`. Manual `Updated::` blocks accumulate (no mutate API) and add no information. Don't write them; lint reads `:edit/time` instead.
- **Creating a new `Sources::` parent on every ingest** — when a page already has `Sources::`, append the new ref as a child of the existing parent. A second sibling `Sources::` block is the bug pattern that produces noisy duplicates.
