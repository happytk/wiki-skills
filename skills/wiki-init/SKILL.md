---
name: wiki-init
description: Use when bootstrapping a new personal wiki for any knowledge domain — research, codebase documentation, reading notes, competitive analysis, or any long-term knowledge accumulation project. Targets a Roam Research graph via the roam-mcp MCP server.
---

# Wiki Init

Bootstrap an LLM-maintained wiki inside a Roam Research graph using the [roam-mcp](https://github.com/happytk/roam-mcp) MCP server.

## Pre-flight

1. **Verify roam-mcp is connected.** Call `roam_find_pages_modified_today` once. If it errors, stop and tell the user to install/configure roam-mcp:
   - `ROAM_API_TOKEN` (must start with `roam-graph-token-`)
   - `ROAM_GRAPH_NAME` (exact case from the URL, e.g. `roamresearch.com/#/app/<graph-name>`)

2. **Check for an existing wiki.** Call `roam_fetch_page_by_title("Wiki Schema")`. If it returns a populated tree, ask whether to reinitialize (which will append a new schema block — there is no delete) or just continue with the existing wiki.

## Process

### 1. Gather configuration (one question at a time)

Ask:
1. **What is the domain/purpose?** (one sentence)
2. **What is the wiki's primary language?** (Korean / English / other) — this is the language for **wiki body content**: summaries, takeaways, entity descriptions, query answers, lint reports. Source text in `Raw Text::` always stays in the original language (verbatim). Default: Korean.
3. **Which source kinds will you ingest?** Pick from `paper | article | transcript | code | book-excerpt | thread | dataset | notes | other`. Default is "all" — narrow it only if you want the schema to advertise a smaller set. (Each kind has its own Raw Text chunking rule; see `wiki-ingest`'s Source-kind taxonomy.)
4. **What categories should `[[Wiki Index]]` use?**
   - Research default: `Sources | Entities | Concepts | Analyses`
   - Codebase default: `Modules | APIs | Decisions | Flows` — see `codebase.md` in this skill's directory for detailed codebase guidance
   - Or specify custom
5. **Where should binary/oversized source files live on disk?** (absolute path, e.g. `~/wikis/ml-research/raw`) — text-extractable sources will be uploaded directly into Roam blocks; this folder is the fallback for PDFs, audio, datasets, etc.

### 2. Create the local `raw/` directory

`mkdir -p <user-supplied path>`. Skills will copy binary sources here and cite them via the `Source::` attribute on Roam pages.

### 3. Create the canonical Roam pages

For each of `Wiki Schema`, `Wiki Index`, `Wiki Overview`:

1. `roam_fetch_page_by_title(<title>)` — if it already has children, skip the create.
2. Otherwise `roam_create_page(<title>)`.
3. Populate via `roam_create_block` (use `children` to create the section tree in one call where possible). Capture the returned uids if you need them later.

**Convention notes for every page you create:**

- Roam has no YAML frontmatter. Page metadata lives as `Key:: value` blocks. Roam's attribute index resolves these regardless of depth — backlinks, the page-attribute table, and `:block/refs` all work the same whether `Source::` sits at the page top level or one level under a `Meta` parent.
- **Three-group top level for content pages** (`#wiki-source`, `#wiki-entity`, `#wiki-concept`, `#wiki-page`, `#wiki-analysis`): `Type::` alone at depth 0, then a `Meta` parent grouping every other attribute (`Source::`, `Sources::`, `Source kind::`, `Tags::`, `Aliases::`, `Category::`, etc.), then a `Notes` parent grouping every content section (`Summary`, `Key Takeaways`, `Description`, `Raw Text`, etc.). This makes meta and body visually distinct in the outliner without sacrificing index behavior.
- Pure-navigation `#wiki-meta` pages (`Wiki Index`, `Wiki Overview`) stay flat — `Type::` at depth 0, then their categories/sections at depth 0 — because they are dashboards, not entries, and an extra `Notes` wrapper would add a click for every read.
- One idea per block. Never use `-` bullets or newlines inside a single block string to fake hierarchy. Use the `children` argument or chain via `parent_uid`.
- Tag write operations with a wiki namespace tag (`#wiki-meta`, `#wiki-source`, `#wiki-page`, `#wiki-entity`, `#wiki-analysis`, `#wiki-change-request`, `#wiki-ingest-queue`) so filtering doesn't depend on the `#ai` auto-tag (which roam-mcp adds by default but can be disabled per MCP entry via the `X-Roam-Ai-Tag: false` header — see the README).

### 4. `[[Wiki Schema]]` content

Build the schema page using the three-group top level: `Type::` alone, `Meta` for the schema's own configuration attributes, `Notes` for the long-form conventions doc that downstream skills read.

```
Type:: #wiki-meta

Meta
  Domain:: <user's domain description>
  Language:: Korean      (or English / 다국어 — the language used for wiki BODY content; Raw Text:: stays in source language)
  Source kinds allowed::
    <one block per allowed kind, e.g. paper / article / transcript / ...>
  Raw path:: <absolute local path to raw/ directory>
  Created:: <today in ordinal format, e.g. April 25th, 2026>
```

If the user said "all", list every kind from the Source-kind taxonomy as a child block under `Source kinds allowed::`. If they narrowed it, list only the chosen ones.

Then under a `Notes` parent at depth 0, append the conventions tree:

```
Notes
  Conventions
    Page references use [[Page Title]] (Roam-native; no slug munging)
    Block references use ((9-char-uid)). Embed inline excerpts with {{embed: ((uid))}}
    Tags: #tag and [[tag]] are equivalent; both produce :block/refs
    Each idea is its own block. Never simulate sub-bullets inside one block string
    Page hierarchy (Type / Meta / Notes)
      Content pages (#wiki-source / #wiki-entity / #wiki-concept / #wiki-page / #wiki-analysis)
        use a three-group top level: Type:: at depth 0, then a Meta parent
        grouping every other attribute (Source::, Sources::, Source kind::, Tags::,
        Aliases::, Category::, ...), then a Notes parent grouping every content
        section (Summary, Key Takeaways, Description, Raw Text, ...)
      Roam's Key:: value attribute index resolves regardless of depth, so nesting
        attributes one level under Meta does not affect the right-sidebar
        attribute table, ((uid)) resolution, or :block/refs backlinks
      Pure-navigation #wiki-meta pages ([[Wiki Index]], [[Wiki Overview]]) stay
        flat — they are dashboards, not entries — Type:: at depth 0 followed by
        their categories/sections at depth 0, no Notes wrapper
      Existing pages on the legacy flat layout keep working; wiki-lint flags them
        as advisory only. Re-grouping is opt-in, page by page
    Outliner discipline
      A block holds ONE idea. Never pack a comma-separated list of refs or
      a structured `Key: a; Key: b` payload into one string
      When an attribute has multiple values, write it as a parent attribute
      block with one CHILD block per value:
        Sources::
          [[Source A]]
          [[Source B]]
          [[Source C]]
      Multi-fact sections are also parent + children:
        Pages updated::
          [[Entity 1]]
          [[Entity 2]]
      A short single ref or 1-3 short tags may stay inline (Tags:: #ml #transformer)
    Required attribute on every wiki page: Type:: (top-level — depth 0, never under Meta)
    Last-edited time is tracked automatically by Roam (:edit/time on every block) — DO NOT write Updated:: blocks; they accumulate without a mutate API and Roam already exposes the canonical edit time
    Daily-note titles MUST use the ordinal date format (April 25th, 2026)
    Language policy
      The wiki has a primary Language:: attribute (default Korean)
      All wiki BODY content is written in that language regardless of source
      language: Summary, Key Takeaways, Description, Appearances notes,
      Related Concepts, Reason::, Source:: prose, query Answer paragraphs,
      Open Questions, lint report explanations, Fix:: prose, daily-note
      log lines
      Original-language verbatim is preserved ONLY in:
        Raw Text:: blocks (one paragraph per block, source verbatim)
        {{embed: ((uid))}} embeds and ((uid)) inline citations
        Direct quotes — when quoting, mark with quotes and cite the uid
      Page titles
        Source pages (papers/articles/products): use the canonical published
          name. If a Korean version is widely used, that; otherwise the
          original (e.g. "Attention Is All You Need" stays English)
        Entity / concept / analysis pages: use the wiki language
          (e.g. [[트랜스포머]], not [[Transformer]])
        For entities widely known by a non-wiki-language name, add an
          Aliases:: parent attribute block on the page with one child per
          alternate name. Other pages can ref either name; the wiki
          canonicalizes to the wiki-language title
      Code identifiers, Roam syntax ([[]], ((uid)), Key::), tag names,
      file paths, URLs are NEVER translated — they are tokens, not prose

  Operation log
    Logs are appended to today's daily note as blocks tagged #wiki-log
    Ops also carry a sub-tag: #wiki-log #ingest, #wiki-log #query, #wiki-log #lint, #wiki-log #update
    Recall recent ops via roam_search_for_tag("wiki-log")

  Mutation policy
    roam-mcp exposes 5 mutation tools, gated by the X-Roam-Mutate: true header on the MCP entry:
      roam_update_block(uid, content)              replace block string
      roam_delete_block(uid)                       delete block + descendants (cascades)
      roam_move_block(uid, parent_uid|page, order) relocate block
      roam_rename_page(title|uid, new_title)       rename page; Roam refs auto-rewire
      roam_delete_page(title|uid)                  delete entire page (DESTRUCTIVE)
    When the header is set, wiki-update applies edits in place per-confirmation
    When the header is not set, wiki-update falls back to legacy queue mode:
      every change becomes a {{[[TODO]]}} Revise: ... block tagged #wiki-change-request
      user applies the edits manually in Roam UI, then marks the TODO done
    Probe at the start of every wiki-update / wiki-lint run to choose the path
    roam_search_by_status("TODO") surfaces pending change requests in legacy mode

  TODO sub-types (filter by tag)
    #wiki-change-request
      Source:: wiki-update or wiki-lint, only emitted in legacy queue mode (no X-Roam-Mutate)
      Action:: apply edit in Roam UI, then mark done — or enable X-Roam-Mutate and re-run
    #wiki-ingest-queue
      Source:: wiki-ingest follow-ups, wiki-query gap offers, or user manually
      Action:: run wiki-ingest with no source → Mode C (queue runner)
      Once ingested, wiki-ingest appends Ingested as:: [[<page>]] under the TODO

  Source kinds (allowed values for Source kind:: on source pages)
    paper        — academic papers, ADRs, RFCs (paragraph-per-block)
    article      — blog posts, news, doc pages (paragraph-per-block)
    transcript   — podcast/video/interview/meeting (one block per speaker turn)
    code         — source files, snippets (one block per logical unit)
    book-excerpt — book chapter, manual section (paragraph-per-block, EXCERPTS only)
    thread       — Twitter/X, HN, mailing list (one block per post/reply)
    dataset      — CSV/JSON sample, OpenAPI spec (structural summary + sample rows)
    notes        — meeting notes, email, chat export (paragraph or message)
    other        — record the chunking decision in a Chunking note:: child

  Copyright
    Upload only what is reasonable as fair-use excerpt. Never paste a whole book,
    full paywalled paper, or other restricted content into Raw Text::. The wiki
    is your synthesis, not a mirror — cite Source:: and excerpt key passages.

  Index categories
    <one block per category the user chose>
```

If the domain is a codebase, append a child block that mirrors the README-vs-wiki rule from `codebase.md`.

### 5. `[[Wiki Index]]` content

`Wiki Index` is a pure-navigation page — keep it flat, with `Type::` at depth 0 and category blocks directly at depth 0 below it (no `Notes` wrapper). The Conventions section in `[[Wiki Schema]]` records this exemption so wiki-lint won't flag it.

Keep the index shallow — `roam_fetch_page_by_title` truncates beyond 3 levels.

```
Type:: #wiki-meta

<Category 1>
  (entries appended here by wiki-ingest as child blocks: [[Page Title]] — one-liner _(ingested April 25th, 2026)_)
<Category 2>
  ...
Maintenance
  (lint reports linked here)
```

Each category lives at depth 1; page entries land at depth 2.

### 6. `[[Wiki Overview]]` content

`Wiki Overview` is the other dashboard page — same exemption as Wiki Index. Type:: at depth 0, then sections at depth 0.

```
Type:: #wiki-meta

Current Understanding
  No sources ingested yet.

Open Questions
  Add questions here as they arise.

Key Entities / Concepts
  Populated as pages are created.
```

**Section ordering convention:** all three content sections (`Current Understanding`, `Open Questions`, `Key Entities / Concepts`) are **reverse-chronological** — newest entries on top. `wiki-ingest`, `wiki-query`, and `wiki-update` write new children with `roam_create_block(..., order=0)` so the latest item is always the first thing the reader sees. Past entries are never deleted just because they're old (append-only history); they simply move down as new ones are prepended.

### 7. Log the init operation

Append a block to today's daily note (call `roam_create_block` with NO `page` arg — defaults to today's daily note). Use the ordinal date format if you need to reference the daily note explicitly. Multi-value attributes are parent + children, never comma-joined into one string.

```
[[Wiki Schema]] init | <domain> #wiki-log #wiki-init
  Categories::
    <Category 1>
    <Category 2>
    ...
  Raw path:: <path>
```

### 8. Confirm

Tell the user:
- Wiki initialized in Roam graph `<graph name>`
- Canonical pages: `[[Wiki Schema]]`, `[[Wiki Index]]`, `[[Wiki Overview]]`
- Local raw directory: `<path>` (binary/oversized sources go here; text-extractable sources will be uploaded into Roam blocks by `wiki-ingest`)
- Add sources by running `wiki-ingest` with a URL, file path, or pasted text
- Save sources for later by dropping `{{[[TODO]]}} Ingest: <url> #wiki-ingest-queue` blocks anywhere in Roam, then running `wiki-ingest` with no arguments to process the queue
- Run `wiki-lint` periodically to keep the wiki healthy and surface ingest-queue progress
- By default every block this plugin writes is auto-tagged `#ai` by roam-mcp; set the MCP entry's `X-Roam-Ai-Tag: false` header to turn that off. Either way, filter on `#wiki-source`, `#wiki-entity`, `#wiki-ingest-queue`, etc. for wiki-specific views
- Mutations are append-only — `wiki-update` queues `#wiki-change-request` TODOs for you to apply in the Roam UI
