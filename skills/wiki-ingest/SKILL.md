---
name: wiki-ingest
description: Use when adding a new source — paper, article, URL, file, transcript, code snippet — to a Roam-backed wiki. Extracts text into block-per-paragraph form so every excerpt earns a stable ((uid)) for citation. One ingest may touch 10-15 Roam pages.
---

# Wiki Ingest

Add a source to the Roam wiki. Read it, discuss with the user, write a source page (with extracted text uploaded as one paragraph per block), update entity/concept pages, run the backlink audit, and update the index, overview, and daily-note log.

## Pre-condition

Locate the wiki:

1. `roam_fetch_page_by_title("Wiki Schema")` — primary discovery.
2. If empty, fall back to `roam_search_by_text("Wiki Schema")` in case the user renamed it.
3. If neither finds it, tell the user to run `wiki-init` first.

Read the schema page in full to learn: domain, source types, raw path, index categories, attribute conventions.

## Process

### 1. Accept the source

The source can be:
- **File path** — read it directly. If binary or oversized, copy to the local `Raw path::` directory from the schema. If text-extractable, you'll upload its content into Roam in step 5.
- **URL** — use the `browse` skill to fetch it. Save binaries to `Raw path::`; for HTML/article content, hold the extracted text in memory for step 5.
- **Pasted text** — use what was provided directly.

### 2. Read the source in full

Read all content. For long sources, read in sections. Do not skip.

### 3. Surface takeaways — BEFORE writing anything

Tell the user:
- 3-5 bullet points of key takeaways
- What entities/concepts this introduces or updates
- Whether it contradicts anything already in the wiki. To check, run `roam_search_by_text(<entity>)` and `roam_search_for_tag(<entity>)` for each candidate, fetch the relevant pages, and read.

Ask: **"Anything specific you want me to emphasize or de-emphasize?"**

Wait for the user's response before proceeding.

### 4. Decide the page title

Use a natural Roam title. Examples:
- `Attention Is All You Need` (paper)
- `OpenAI GPT-5 Release Notes` (article)
- `Mod: Auth Service` (codebase, see `codebase.md`)

Check existence first: `roam_fetch_page_by_title(<title>)`. If the page already exists with content, decide whether to add to it or pick a more specific title. Then `roam_create_page(<title>)` for new pages.

### 5. Build the source page

The source page layout — top-level blocks created via `roam_create_block` (capture every returned uid; you'll need them for citations):

```
Type:: #wiki-source
Source:: <URL or absolute file path under Raw path::>
Source kind:: paper | article | transcript | code | other
Tags:: #<topic> #<topic>
Sources::                       (empty for source pages)
Updated:: <today, ordinal: April 25th, 2026>

Summary
  <2-3 paragraph synthesis in your own words. Each paragraph is its own block.>
  <Inline citations to Raw Text:: blocks via ((uid)).>

Key Takeaways
  <one bullet per block>
  <one bullet per block>
  ...

Entities & Concepts
  [[Entity 1]] — <one-line note>
  [[Concept 2]] — <one-line note>

Relation to Other Wiki Pages
  Connects to [[Other Page]] because <reason>
  Updates the claim in [[Other Page]] that <…>

Raw Text
  <paragraph 1 of the source verbatim, one block>
  <paragraph 2 of the source verbatim, one block>
  ...
```

**Construction recipe:**

1. Build a single nested `children` tree and pass it to `roam_create_block(content="Type:: #wiki-source", page=<title>, children=[…])` — but Roam's `Key:: value` attribute index works best when each attribute is its own top-level block, so create the attribute blocks one at a time at depth 0 first, then create the section parents (`Summary`, `Key Takeaways`, `Entities & Concepts`, `Relation to Other Wiki Pages`, `Raw Text`) each with their content as a `children` tree.
2. **Raw Text uploading** — split the source into paragraph-sized units (or logical units for code/transcripts: function, scene, speaker turn). Pass them as the `children` of the `Raw Text` block in a single `roam_create_block` call. Capture the `created_uids` from the response. These uids are the citation handles for the rest of the wiki.
3. Write the `Summary` blocks **after** `Raw Text` is uploaded so summary paragraphs can `((uid))`-cite the original lines. Use `{{embed: ((uid))}}` when the original phrasing carries weight (a quotable claim, a precise number).

**Idempotency.** If you're re-ingesting an updated source, fetch the existing source page first and string-match children before appending — match the first ~80 chars of each Raw Text paragraph to skip already-uploaded content.

### 6. Update entity and concept pages

For each entity/concept this source touches:

- **Page exists** (`roam_fetch_page_by_title` returns a tree): read it, add a child block under the relevant section, ensure the `Sources::` attribute lists this source's `[[<source title>]]`, update the `Updated::` attribute (append a new `Updated:: <today>` block — there is no in-place edit).
- **Page doesn't exist:** `roam_create_page(<entity>)`, then create attribute + section blocks:

```
Type:: #wiki-entity            (or #wiki-concept)
Sources:: [[<this source title>]]
Tags:: #<topic>
Updated:: <today>

Description
  <synthesis in your own words, citing ((uid))s into Raw Text:: of the relevant source pages>

Appearances in Sources
  [[<source title>]] — <one-line note>  ((uid-of-key-quote))

Related Concepts
  [[<related>]] — <relationship>
```

When the original phrasing matters, `{{embed: ((uid))}}` the source block instead of paraphrasing.

### 7. Backlink audit — do not skip

Roam already maintains the `:block/refs` index, but textual mentions that don't use `[[…]]` won't appear there. For each new entity/concept:

1. `roam_search_for_tag(<entity title>)` → blocks that already reference this entity. Resolve their `:block/page` titles; call this set `R`.
2. `roam_search_by_text(<entity title>)` → blocks that mention the entity textually. Resolve their pages; call this set `T`.
3. Pages in `T - R` mention the entity but don't link to it. For each, decide whether to append a child block under the relevant parent that explicitly references `[[<entity>]]`. Confirm with the user before bulk edits.

This is the step most commonly skipped. A compounding wiki's value comes from bidirectional links.

### 8. Update `[[Wiki Index]]`

Fetch `[[Wiki Index]]`. Find the category block (depth 1) that matches this source's category. Append a child block:

```
[[<page title>]] — <one-line summary> _(ingested <today, ordinal>)_
```

For any new entity/concept pages created in step 6, add those under their categories too. Skip categories that already contain a string-matching entry (idempotency).

### 9. Update `[[Wiki Overview]]`

Fetch `[[Wiki Overview]]`. If this source:
- Introduces a significant concept → append a child block under `Key Entities / Concepts` referencing `[[<concept>]]`
- Shifts the overall understanding → append a child block under `Current Understanding` (do not delete the old one — append-only)
- Raises a new question → append under `Open Questions`

Append a fresh `Updated:: <today>` attribute block at depth 0.

### 10. Log to today's daily note

Append (no `page` arg, defaults to today's daily note):

```
[[Wiki Schema]] ingest | <source title> #wiki-log #wiki-ingest
  Source:: <URL or path>
  Page:: [[<source title>]]
  Pages updated:: [[<entity1>]], [[<entity2>]], ...
  Backlinks added on:: [[<page A>]], [[<page B>]]
```

### 11. Report to user

- Source page: `[[<title>]]` (with `Raw Text::` containing N blocks, uids captured for citation)
- Entity/concept pages created or updated: <list with `[[Page]]` links>
- Pages that received new `[[<entity>]]` references in the backlink audit: <list>
- `[[Wiki Index]]` and `[[Wiki Overview]]` updated
- Logged to today's daily note as `#wiki-log #wiki-ingest`

## Common Mistakes

- **Skipping Raw Text upload** — the wiki loses its "auditable to source" property. Always upload extracted text when feasible.
- **One giant Raw Text block** — defeats `((uid))` citation. One paragraph (or logical unit) per block.
- **Skipping the backlink audit** — set-difference between `roam_search_for_tag` and `roam_search_by_text` is the whole point.
- **Forgetting to capture uids** — `roam_create_block` returns `uid` and `created_uids`; thread them into your subsequent citation blocks rather than re-fetching.
- **Faking sub-bullets with `-`** — Roam treats one block as one idea. Use `children` instead.
