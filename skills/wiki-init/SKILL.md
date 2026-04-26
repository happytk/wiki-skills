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
2. **Which source kinds will you ingest?** Pick from `paper | article | transcript | code | book-excerpt | thread | dataset | notes | other`. Default is "all" — narrow it only if you want the schema to advertise a smaller set. (Each kind has its own Raw Text chunking rule; see `wiki-ingest`'s Source-kind taxonomy.)
3. **What categories should `[[Wiki Index]]` use?**
   - Research default: `Sources | Entities | Concepts | Analyses`
   - Codebase default: `Modules | APIs | Decisions | Flows` — see `codebase.md` in this skill's directory for detailed codebase guidance
   - Or specify custom
4. **Where should binary/oversized source files live on disk?** (absolute path, e.g. `~/wikis/ml-research/raw`) — text-extractable sources will be uploaded directly into Roam blocks; this folder is the fallback for PDFs, audio, datasets, etc.

### 2. Create the local `raw/` directory

`mkdir -p <user-supplied path>`. Skills will copy binary sources here and cite them via the `Source::` attribute on Roam pages.

### 3. Create the canonical Roam pages

For each of `Wiki Schema`, `Wiki Index`, `Wiki Overview`:

1. `roam_fetch_page_by_title(<title>)` — if it already has children, skip the create.
2. Otherwise `roam_create_page(<title>)`.
3. Populate via `roam_create_block` (use `children` to create the section tree in one call where possible). Capture the returned uids if you need them later.

**Convention notes for every page you create:**

- Roam has no YAML frontmatter. Page metadata lives as **flat top-level attribute blocks** of the form `Key:: value`. Roam indexes these regardless of depth, but flat top-level blocks show up in the right-sidebar attribute table.
- One idea per block. Never use `-` bullets or newlines inside a single block string to fake hierarchy. Use the `children` argument or chain via `parent_uid`.
- Tag write operations with a wiki namespace tag (`#wiki-meta`, `#wiki-source`, `#wiki-page`, `#wiki-entity`, `#wiki-analysis`, `#wiki-change-request`, `#wiki-ingest-queue`) so the auto-injected `#ai` is not the only filter available.

### 4. `[[Wiki Schema]]` content

Top-level blocks:

```
Type:: #wiki-meta
Domain:: <user's domain description>
Source kinds allowed::
  <one block per allowed kind, e.g. paper / article / transcript / ...>
Raw path:: <absolute local path to raw/ directory>
Created:: <today in ordinal format, e.g. April 25th, 2026>
```

If the user said "all", list every kind from the Source-kind taxonomy as a child block. If they narrowed it, list only the chosen ones.

Then a child tree:

```
Conventions
  Page references use [[Page Title]] (Roam-native; no slug munging)
  Block references use ((9-char-uid)). Embed inline excerpts with {{embed: ((uid))}}
  Tags: #tag and [[tag]] are equivalent; both produce :block/refs
  Each idea is its own block. Never simulate sub-bullets inside one block string
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
  Page metadata is a set of flat top-level attribute blocks (Key:: value)
  Required attributes on every wiki page: Type::, Updated::
  Daily-note titles MUST use the ordinal date format (April 25th, 2026)

Operation log
  Logs are appended to today's daily note as blocks tagged #wiki-log
  Ops also carry a sub-tag: #wiki-log #ingest, #wiki-log #query, #wiki-log #lint, #wiki-log #update
  Recall recent ops via roam_search_for_tag("wiki-log")

Mutation policy
  roam-mcp exposes no update or delete tool — wiki content is append-only via this plugin
  wiki-update proposes changes as {{[[TODO]]}} Revise: ... blocks tagged #wiki-change-request
  Apply approved changes manually in the Roam UI, then mark the TODO done
  roam_search_by_status("TODO") surfaces all pending TODOs

TODO sub-types (filter by tag)
  #wiki-change-request
    Source:: wiki-update or wiki-lint
    Action:: apply edit in Roam UI, then mark done
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

Keep the index shallow — `roam_fetch_page_by_title` truncates beyond 3 levels. Layout:

```
Type:: #wiki-meta
Updated:: <today>

<Category 1>
  (entries appended here by wiki-ingest as child blocks: [[Page Title]] — one-liner _(ingested April 25th, 2026)_)
<Category 2>
  ...
Maintenance
  (lint reports linked here)
```

Each category lives at depth 1; page entries land at depth 2.

### 6. `[[Wiki Overview]]` content

```
Type:: #wiki-meta
Updated:: <today>

Current Understanding
  No sources ingested yet.

Open Questions
  Add questions here as they arise.

Key Entities / Concepts
  Populated as pages are created.
```

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
- Every block this plugin writes is auto-tagged `#ai` by roam-mcp; filter on `#wiki-source`, `#wiki-entity`, `#wiki-ingest-queue`, etc. for wiki-specific views
- Mutations are append-only — `wiki-update` queues `#wiki-change-request` TODOs for you to apply in the Roam UI
