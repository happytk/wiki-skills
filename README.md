# wiki-skills (Roam Research)

A Claude Code plugin implementing [Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — a persistent, compounding knowledge base maintained by your LLM — backed by a [Roam Research](https://roamresearch.com/) graph instead of filesystem markdown.

Instead of RAG (re-deriving answers from raw documents every time), this system builds and maintains a wiki: a structured, interlinked set of Roam pages and blocks that gets richer with every source you add and every question you ask. Source text is uploaded paragraph-per-block so every excerpt earns a stable `((uid))` that any other page can cite directly.

## Prerequisites

This plugin talks to Roam through the [`roam-mcp`](https://github.com/happytk/roam-mcp) MCP server. Configure it before installing:

| Variable | Purpose |
| --- | --- |
| `ROAM_API_TOKEN` | Roam Backend API token. Must start with `roam-graph-token-`. |
| `ROAM_GRAPH_NAME` | Graph name as it appears in `roamresearch.com/#/app/<graph-name>`. |

Then add roam-mcp to your Claude Code MCP servers list and confirm `roam_find_pages_modified_today` works before running `wiki-init`.

## Installation

```bash
/plugin marketplace add kfchou/wiki-skills
/plugin install wiki-skills@kfchou/wiki-skills
```

## Skills

| Skill | Description |
|---|---|
| `wiki-init` | Bootstrap a new Roam-backed wiki for any domain — creates `[[Wiki Schema]]`, `[[Wiki Index]]`, `[[Wiki Overview]]`. |
| `wiki-ingest` | Add a source (paper, URL, file, transcript) — uploads extracted text paragraph-per-block under `Raw Text::`, creates entity/concept pages, runs the backlink audit. |
| `wiki-query` | Ask a question against the wiki — answers cite via `[[Page]]` and `((uid))` block refs into the original `Raw Text::` excerpts. |
| `wiki-lint` | Health audit via Datalog: orphans, stale claims, broken refs, missing pages, contradictions. |
| `wiki-update` | Queue revisions as `{{[[TODO]]}} #wiki-change-request` blocks (roam-mcp has no update/delete API; you apply changes manually in Roam). |

## How It Works

### Storage model

```
Roam graph
├── [[Wiki Schema]]          ← conventions, raw path, mutation policy (how skills find the wiki)
├── [[Wiki Index]]           ← content catalog: every page, one-line summary, by category (max 2 levels deep)
├── [[Wiki Overview]]        ← evolving synthesis of everything known
├── [[<source titles>]]      ← one page per source, with Raw Text:: paragraphs as ((uid))-citable blocks
├── [[<entity / concept pages>]]
├── [[<analysis pages>]]     ← saved query answers
└── Daily notes              ← operation log: every wiki-* operation appends a #wiki-log block
```

Plus a local `raw/` directory on disk for binary or oversized sources (PDFs, audio) that can't live as Roam blocks. The path is recorded in `[[Wiki Schema]]`.

### Page metadata

Roam has no YAML frontmatter. Each page starts with **flat top-level attribute blocks** of the form `Key:: value`:

```
Type:: #wiki-source
Source:: https://arxiv.org/abs/1706.03762
Source kind:: paper
Tags:: #ml #transformer
Updated:: April 25th, 2026
```

…followed by section blocks (`Summary`, `Key Takeaways`, `Entities & Concepts`, `Raw Text`, …) as siblings.

### Cross-references

- `[[Page Title]]` — Roam-native page reference (no slug munging)
- `((uid))` — block reference for citing a specific paragraph or quote
- `{{embed: ((uid))}}` — embed the original block inline

`wiki-query` and `wiki-update` use `((uid))` aggressively so every claim traces back to source text in `Raw Text::`.

### Typical Workflow

```
wiki-init          → bootstrap a new wiki
wiki-ingest        → add sources one at a time (uploads text into Raw Text:: blocks; repeat)
wiki-query         → ask questions; save good answers back as pages
wiki-lint          → periodic health check (every 5-10 ingests)
wiki-update        → queue revisions as #wiki-change-request TODOs
```

### Key Behaviors

- **`wiki-ingest`** surfaces key takeaways and asks what to emphasize *before* writing anything. Extracts source text into paragraph-per-block units under `Raw Text::` so every excerpt has a stable `((uid))`. After creating a source page, it runs a backlink audit using the set-difference of `roam_search_for_tag` and `roam_search_by_text` to add explicit `[[entity]]` references where text mentions exist without links.
- **`wiki-query`** always reads the wiki (never answers from memory). Cites with both `[[Page]]` and `((uid))`, and offers to file the answer back as a `#wiki-analysis` page that re-uses the same uids.
- **`wiki-lint`** uses Datalog queries (via `roam_datomic_query`) for orphans, stale claims, and missing-page detection. Writes a `[[Lint <date>]]` page and offers fixes that queue as `#wiki-change-request` TODOs.
- **`wiki-update`** displays current/proposed/reason/source diffs per change, queues each as a `{{[[TODO]]}} … #wiki-change-request` block under the affected page, sweeps downstream pages for related claims, and logs unconditionally.

### Mutation policy (read this once)

`roam-mcp` exposes only read and create tools — no update, no delete. This plugin treats wiki content as **append-only**:

- New blocks are appended.
- Edits are *queued* as `{{[[TODO]]}} Revise: … #wiki-change-request` blocks under the affected page. You apply them manually in the Roam UI, then mark the TODO done.
- `roam_search_by_status("TODO")` filtered by `#wiki-change-request` lists every pending change.

If a future MCP server exposes write transactions, `wiki-update` should be revised to apply changes directly.

### Tag conventions

| Tag | Used for |
| --- | --- |
| `#ai` | Auto-injected by roam-mcp on every write — benign, but use the `#wiki-*` tags below for filtering. |
| `#wiki-meta` | Schema, index, overview, lint reports |
| `#wiki-source` | Source pages (papers, articles, etc.) |
| `#wiki-entity` / `#wiki-concept` | Entity and concept pages |
| `#wiki-analysis` | Saved query answers |
| `#wiki-page` | Generic wiki page (codebase modules, flows, decisions) |
| `#wiki-log` | Daily-note operation log entries (sub-tagged `#wiki-ingest`, `#wiki-query`, `#wiki-lint`, `#wiki-update`) |
| `#wiki-change-request` | Queued edits for the user to apply manually |

## Use Cases

Works for any domain where you're accumulating knowledge over time:

- **Research** — papers, articles, reports on a topic
- **Codebase documentation** — modules, APIs, architecture decisions, data flows (see `skills/wiki-init/codebase.md`)
- **Reading notes** — books, papers, podcasts
- **Competitive analysis** — tracking companies, products, developments
- **Personal knowledge** — goals, health, self-improvement

## Migration from v1 (filesystem / Obsidian)

v1 of this plugin (commit [`c5f2ce5`](https://github.com/kfchou/wiki-skills/tree/c5f2ce5)) targeted filesystem markdown with YAML frontmatter and Obsidian-style `[[slug]]` links. v2 is a clean break — there is no automated migration. If you have an existing v1 wiki, install the plugin at the v1 SHA to keep the old behavior, or re-ingest your sources into Roam with the v2 plugin.

## Inspired By

[Andrej Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) (April 2026)

> "The wiki keeps getting richer with every source you add and every question you ask."

Roam-side conventions follow [`roam-mcp`](https://github.com/happytk/roam-mcp) by happytk.

## License

MIT
