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

Read the schema page in full to learn: domain, **`Language::` (wiki body language; default Korean if missing)**, allowed source kinds, raw path, index categories, attribute conventions.

## Language policy (applies to every step below)

The wiki has a primary `Language::` from the schema. **All wiki body content this skill writes must be in that language**, regardless of the source's language. Body content includes: Summary paragraphs, Key Takeaways bullets, Entities & Concepts notes, Relation to Other Wiki Pages prose, entity Description paragraphs, Appearances notes, Related Concepts relationship strings, Sources note prose, daily-note log lines.

**Source verbatim** is preserved only in:
- `Raw Text::` blocks (one paragraph per block, exactly as the source reads — never translate)
- `{{embed: ((uid))}}` embeds in Summary / entity blocks where the original phrasing matters
- Direct quotations (mark with quotes + a `((uid))` citation)

For an English source ingested into a Korean wiki, you produce: Korean Summary that cites `((uid))` into the English Raw Text. The reader sees Korean synthesis with one click to the English original. For source-page titles, see [[Wiki Schema]]'s Language policy — canonical published names stay in their original language; entity/concept/analysis page titles use the wiki language.

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

Tell the user, **in the wiki's `Language::`** (Korean by default — even if the source is English):
- 3-5 bullet points of key takeaways
- What entities/concepts this introduces or updates (proper nouns and unfamiliar technical terms can stay in original — explain in the wiki language)
- Whether it contradicts anything already in the wiki. To check, run `roam_search_by_text(<entity>)` and `roam_search_for_tag(<entity>)` for each candidate, fetch the relevant pages, and read.

Ask (in the wiki language): **"강조하거나 덜 다루고 싶은 부분 있나요?"** (or the equivalent in the wiki's set language).

Wait for the user's response before proceeding.

### 5. Decide the page title

Title language follows the schema's Language policy:
- **Source pages** — use the canonical published name in its original language. Don't translate paper/product titles. Examples: `Attention Is All You Need`, `OpenAI GPT-5 Release Notes`. If a Korean version is widely used (e.g. localized release notes), use that.
- **Codebase domain** — keep prefixes (`Mod: …`, `API: …`, `Decision: …`, `Flow: …`) and use the wiki language for the descriptive part: `Mod: 인증 서비스` for a Korean wiki, `Mod: Auth Service` for English.
- **Entity / concept / analysis pages (later steps)** — use the wiki language. `[[트랜스포머]]` not `[[Transformer]]` for a Korean wiki. Add an `Aliases::` parent attribute block with one child per common alternate name so backlinks survive.

Check existence first: `roam_fetch_page_by_title(<title>)`. If the page already exists with content, decide whether to add to it or pick a more specific title. Then `roam_create_page(<title>)` for new pages.

### 6. Build the source page

The source page layout uses a **three-group top level**: `Type::` alone at depth 0, then a `Meta` parent for source metadata, then a `Notes` parent for synthesis and verbatim content. Every concept is still **its own block**; multi-value attributes use parent + children, never comma-joined strings. Roam's `Key:: value` attribute index and `:block/refs` index resolve regardless of depth, so nesting attributes one level under `Meta` does not affect attribute lookup, page-attribute table, or backlinks.

Top-level shape (each line is a separate block created via `roam_create_block`; capture every returned uid for later citation):

```
Type:: #wiki-source

Meta
  Source:: <URL or absolute file path under Raw path::>
  Source kind:: paper | article | transcript | code | other
  Tags:: #<topic1> #<topic2>          (1-3 short tags inline is fine; otherwise split below)

Notes
  Summary                                       (← WIKI LANGUAGE — body in Korean if Language::Korean)
    <paragraph 1 — one block, in wiki language>
    <paragraph 2 — one block, in wiki language>
    <paragraph 3 — one block, in wiki language>  (each cites Raw Text:: via ((uid)) where applicable)

  Key Takeaways                                 (← WIKI LANGUAGE)
    <takeaway 1 — one block>
    <takeaway 2 — one block>
    ...

  Entities & Concepts                           (← page titles per Language policy; notes in wiki language)
    [[Entity 1]]
      <one-line note in wiki language>
    [[Concept 2]]
      <one-line note in wiki language>
    ...

  Relation to Other Wiki Pages                  (← WIKI LANGUAGE)
    Connects to [[Other Page]]
      <reason in wiki language — child block>
    Updates the claim in [[Other Page A]]
      <which claim and how, in wiki language — child block>

  Raw Text                                      (← ORIGINAL SOURCE LANGUAGE — verbatim, never translate)
    <paragraph 1 of the source verbatim — one block>
    <paragraph 2 of the source verbatim — one block>
    ...
```

If you have many tags or many topical refs, expand the inline form into parent + children under `Meta`:

```
Meta
  Tags::
    #<topic1>
    #<topic2>
    #<topic3>
    ...
```

**Construction recipe:**

1. Create `Type:: #wiki-source` at depth 0 first (one `roam_create_block` call). This is the lone top-level attribute and the marker `wiki-lint` uses to recognize wiki-managed pages.
2. Create the `Meta` parent at depth 0, then create each metadata attribute (`Source::`, `Source kind::`, `Tags::`, etc.) as a child of `Meta`. You can pass them as a single `children=[…]` tree on the `Meta` create call.
3. Create the `Notes` parent at depth 0, then create the section blocks (`Summary`, `Key Takeaways`, `Entities & Concepts`, `Relation to Other Wiki Pages`, `Raw Text`) as children of `Notes`, each with their own content as a `children` tree.
4. **Raw Text uploading** — split the source into paragraph-sized units (or logical units for code/transcripts: function, scene, speaker turn). Pass them as the `children` of the `Raw Text` block in a single `roam_create_block` call. Capture the `created_uids` from the response. These uids are the citation handles for the rest of the wiki.
5. Write the `Summary` blocks **after** `Raw Text` is uploaded so summary paragraphs can `((uid))`-cite the original lines. Use `{{embed: ((uid))}}` when the original phrasing carries weight (a quotable claim, a precise number).

**Pre-write validation (mandatory).** Before EVERY `roam_create_block` / `roam_create_page` / `roam_add_todo` call, verify each `content` field — including every `content` inside the `children` tree — is a non-empty string. If a value is `undefined`, `null`, an empty string, or the literal string `"undefined"`/`"null"`, **drop that entry** (don't send it) and surface the gap to the user before proceeding. The auto `#ai` suffix concat means an undefined value renders as a literal `undefined #ai` block in Roam — `roam_delete_block` can clean it up but only when `X-Roam-Mutate: true`, so don't write it in the first place.

**Proofread before write (mandatory when `Language::` uses a non-Latin script — Korean, Japanese, Chinese, Cyrillic, Arabic, Devanagari).** LLM token-level generation occasionally produces a wrong-but-valid syllable in CJK/non-Latin output (e.g., `카테고리` → `쳤터고리`). UTF-8 stays intact through roam-mcp, so this is a generation issue, not an encoding one — the only durable fix is a self-review before the prose hits Roam.

Run **one batched proofread call per ingest**, covering every wiki-language prose block drafted for the upcoming write pass:

- Source page: Summary paragraphs, Key Takeaways bullets, Entities & Concepts one-line notes, Relation to Other Wiki Pages paragraphs
- Entity / concept pages (step 7): Description paragraphs, Appearances notes, Related Concepts relationship strings

To make the single batched call possible, **draft the entity / concept prose up front during step 6** (after Raw Text uids are captured) instead of just-in-time inside step 7. Hold every drafted block in memory keyed by its target write call.

**Excluded from proofread** (do not include these in the proofread payload):

- `Raw Text` blocks — source-verbatim; "correcting" them would corrupt the citation handle and falsify the audit trail
- Page titles, attribute keys (`Source::`, `Tags::`), `[[refs]]`, `((uids))`, URLs, file paths, code identifiers — tokens, not prose
- Source-language direct quotations and `{{embed: ((uid))}}` blocks

**Issues-only return format** (keeps output tokens minimal — most blocks pass without a rewrite):

```
For each block below, decide if it contains a likely token-level corruption
typo (e.g. wrong syllable, dropped character, swapped Hangul jamo, transposed
Kanji). If yes, return { "index": <N>, "original": "<…>", "corrected": "<…>" }.
If no, omit it. Return [] if all blocks are clean. Do NOT rewrite for style,
flow, or word choice — only fix obvious token-level errors.

Blocks:
[1] <block content>
[2] <block content>
...
```

Apply each returned correction to the corresponding draft block before issuing the `roam_create_block` calls. If the call returns `[]`, write as drafted.

**Skip the proofread step entirely when `Language::` is Latin-script** (English, Spanish, French, German, Portuguese, etc.) — token-level corruption is rare in Latin LLM output and the cost-benefit doesn't justify the extra call.

**Idempotency on re-ingest.** If you're re-ingesting an updated source, fetch the existing source page first and walk the existing `Raw Text` children. The page may use the new `Notes` > `Raw Text` layout or the legacy flat layout (Raw Text at page top level) — locate `Raw Text` by string + page rather than by direct-child position so the walk works either way. A depth-tolerant Datalog pull:

```clojure
[:find ?uid ?s
 :where
 [?p :node/title "<source title>"]
 [?rt :block/string "Raw Text"]
 [?rt :block/page ?p]
 [?rt :block/children ?c]
 [?c :block/uid ?uid]
 [?c :block/string ?s]]
```

Then:

1. Match each new paragraph against existing children by the first ~80 chars.
2. **Identical match** → skip; reuse the existing `((uid))` (any wiki page that already cites it stays valid).
3. **Existing block, slightly changed source text** (same paragraph, edited wording) → if `X-Roam-Mutate: true` is enabled, call `roam_update_block(uid=<existing>, content=<new>)` to keep the citation stable. If mutation is disabled, append the new version as a sibling and tag it `Supersedes:: ((<old-uid>))` so readers know which is current.
4. **New paragraph not in the existing tree** → append as a new child of the existing `Raw Text` block (single `roam_create_block` call with `parent_uid=<raw-text-uid>` and `children=[…]` for the new paragraphs only). Do **not** create a fresh `Raw Text` sibling — append under whichever `Raw Text` parent already exists, regardless of layout generation.
5. **Existing block whose source paragraph was deleted upstream** → if mutation is enabled, `roam_delete_block(uid=<existing>)` only when no other wiki page cites that uid (run `roam_search_for_tag(<source title>)` and check). When uncertain, leave it and append a new `Sources note:: this paragraph removed in <date> revision` child block instead.

The mutation calls require the `X-Roam-Mutate: true` header on your MCP entry; without it, fall back to the v2.0 append-only behavior (rule 4 only) and tell the user that re-ingest could not refresh existing blocks in place.

### 7. Update entity and concept pages

By the time you reach step 7, the proofread pass from step 6 has already corrected the drafted Description / Appearances notes / Related Concepts strings (when `Language::` is non-Latin). Use the corrected drafts as the `content` for the `roam_create_block` / `roam_create_page` calls below; do not re-generate the prose.

For each entity/concept this source touches:

- **Page exists** (`roam_fetch_page_by_title` returns a tree): read it, then add **only what is actually new**. For `Sources::` — locate the existing parent attribute block by string + page (it may sit at depth 0 on legacy pages or under `Meta` on new pages); append `[[<source title>]]` as a new child of that existing parent (do NOT create another `Sources::` sibling, regardless of where the existing one lives). If `Sources::` exists only as a single-value inline attribute (legacy shape) leave it and append the new ref under the same parent's grandparent location. Roam tracks edit time on every block via `:edit/time` — do NOT write `Updated:: <today>` blocks.
- **Page doesn't exist:** `roam_create_page(<entity>)`, then build the page using the same three-group top level (`Type::` / `Meta` / `Notes`) as a source page. One concept per block; multi-value attributes use parent + children:

```
Type:: #wiki-entity            (or #wiki-concept)

Meta
  Sources::
    [[<this source title>]]    (additional source pages will be added here as siblings on future ingests)
  Aliases::                    (optional — add when the canonical title is in the wiki language but the entity is also widely known by other names)
    <alternate name 1>
    <alternate name 2>
  Tags::
    #<topic1>
    #<topic2>

Notes
  Description                                 (← WIKI LANGUAGE)
    <paragraph 1 in wiki language, citing ((uid)) into Raw Text:: of the source>
    <paragraph 2 in wiki language>

  Appearances in Sources                      (← notes in WIKI LANGUAGE; uids unchanged)
    [[<source title>]]
      <one-line note in wiki language — child block>
      ((uid-of-key-quote))       (separate child block — the citation itself)

  Related Concepts                            (← WIKI LANGUAGE)
    [[<related entity 1>]]
      <relationship in wiki language — child block>
    [[<related entity 2>]]
      <relationship in wiki language — child block>
```

When the original phrasing carries weight, `{{embed: ((uid))}}` the source block instead of paraphrasing — the embed renders the original-language verbatim alongside your wiki-language synthesis.

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
