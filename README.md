# wiki-skills (Roam Research)

A Claude Code plugin implementing [Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — a persistent, compounding knowledge base maintained by your LLM — backed by a [Roam Research](https://roamresearch.com/) graph instead of filesystem markdown.

Instead of RAG (re-deriving answers from raw documents every time), this system builds and maintains a wiki: a structured, interlinked set of Roam pages and blocks that gets richer with every source you add and every question you ask. Source text is uploaded paragraph-per-block so every excerpt earns a stable `((uid))` that any other page can cite directly.

## Prerequisites

> ⚠️ **roam-mcp registration is required.** Every skill in v2 calls the `roam_*` tools (`roam_create_page`, `roam_create_block`, `roam_fetch_page_by_title`, `roam_search_for_tag`, `roam_search_by_text`, `roam_search_by_status`, `roam_datomic_query`, `roam_find_pages_modified_today`). If those tools aren't present in your Claude Code MCP context, every skill will stop at its first tool call. There is no filesystem fallback in v2 — if you don't plan to use a Roam graph, install [v1 instead](#alternative-mcp-servers--v1-fallback) and skip the rest of this section.

This plugin talks to Roam through the [`roam-mcp`](https://github.com/happytk/roam-mcp) MCP server, which runs as a **Cloudflare Worker** (HTTP transport, JSON-RPC 2.0). Setup is two-sided: credentials live on the Worker; Claude Code only needs the Worker URL.

### 1. Roam credentials

| Variable | Where it goes | Notes |
| --- | --- | --- |
| `ROAM_API_TOKEN` | Cloudflare Worker secret (set with `wrangler secret put`) | Must start with `roam-graph-token-`. **Not** `roam-graph-local-token-`. Get it from Roam: Settings → Graph → API tokens. |
| `ROAM_GRAPH_NAME` | `wrangler.toml` `[vars]` block on the Worker | Exact case/hyphens as it appears in `roamresearch.com/#/app/<graph-name>`. |

Neither variable is set on the Claude Code side — they exist only on the Worker.

### 2. Deploy the Worker

```bash
git clone https://github.com/happytk/roam-mcp.git
cd roam-mcp
# edit wrangler.toml: set ROAM_GRAPH_NAME under [vars]
npx wrangler deploy
npx wrangler secret put ROAM_API_TOKEN     # paste the token when prompted
```

Verify the Worker is healthy:

```bash
curl https://roam-mcp.<your-account>.workers.dev/check
# → {"ok": true, "graph": "<your-graph>", "sampleCount": 1}
```

Prerequisites for this step: Node.js 18+, a Cloudflare account (free tier works).

### 3. Register the Worker as an MCP server in Claude Code

Pick **one** of:

- **Project-shared** (`<repo>/.mcp.json`, committed so collaborators inherit it):
  ```json
  {
    "mcpServers": {
      "roam": {
        "type": "http",
        "url": "https://roam-mcp.<your-account>.workers.dev/mcp"
      }
    }
  }
  ```
- **User-level** (applies to all your projects):
  ```bash
  claude mcp add --scope user --transport http roam https://roam-mcp.<your-account>.workers.dev/mcp
  ```

Replace `<your-account>` with your Cloudflare subdomain.

### 4. Confirm the connection

In a Claude Code session, ask Claude to call `roam_find_pages_modified_today`. If it returns a list (possibly empty), you're ready to run `wiki-init`. If it errors, re-check the `/check` endpoint and the URL in your MCP config.

### Per-project Roam graph (header override)

`roam-mcp` reads three HTTP headers per request:

| Header | Effect | Default if absent |
| --- | --- | --- |
| `X-Roam-Graph` | Override `ROAM_GRAPH_NAME` from `wrangler.toml` | the Worker's baked-in graph |
| `X-Roam-Token` | Override `ROAM_API_TOKEN` from the Worker secret | the Worker's baked-in token |
| `X-Roam-Ai-Tag` | Toggle the auto `#ai` tag on root blocks of `roam_create_page`, `roam_create_block`, `roam_add_todo`. Accepts `true`/`false`, `1`/`0`, `on`/`off`, `yes`/`no` (case-insensitive) | `true` (auto-tag enabled) |

The first two let you register **one** roam-mcp endpoint at user scope and target a different Roam graph per project — no redeploy, no second MCP registration. The third lets you turn off the `#ai` noise per project (or for one specific MCP entry) without forking roam-mcp; see also [Tag conventions](#tag-conventions). In each project's `.mcp.json`:

```json
{
  "mcpServers": {
    "roam": {
      "type": "http",
      "url": "https://roam-mcp.<your-account>.workers.dev/mcp",
      "headers": {
        "X-Roam-Graph": "<project-graph>",
        "X-Roam-Token": "roam-graph-token-xxxxxxxxxxxx",
        "X-Roam-Ai-Tag": "false"
      }
    }
  }
}
```

You can also override at user scope by editing `~/.claude.json` directly — add the same `headers` block to the `roam` (or `roam-local`) MCP entry. Without these headers, the Worker falls back to the values baked in at deploy time.

### Local development mode (optional)

Two ways to run roam-mcp on your own machine instead of (or alongside) a deployed Worker. Once `localhost:8787/mcp` is reachable, the per-project header override above works identically against it — one `roam-local` MCP entry in `~/.claude.json` can serve every project.

**A. Always-on via launchd (recommended on macOS)**

Set up once, then `localhost:8787/mcp` is always reachable across logins and crashes.

```bash
git clone https://github.com/happytk/roam-mcp.git ~/code/roam-mcp
cd ~/code/roam-mcp
# wrangler.toml: set ROAM_GRAPH_NAME under [vars]
echo "ROAM_API_TOKEN=roam-graph-token-..." > .dev.vars
which npx     # remember the path; you'll paste it into the plist below
```

Save the following as `~/Library/LaunchAgents/dev.roam-mcp.local.plist` (replace `YOU` with your username, the `npx` path with what `which npx` returned, and the `WorkingDirectory` if you cloned somewhere else):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>dev.roam-mcp.local</string>
  <key>WorkingDirectory</key>
  <string>/Users/YOU/code/roam-mcp</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/npx</string>
    <string>wrangler</string>
    <string>dev</string>
    <string>--port</string>
    <string>8787</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key>
  <string>/tmp/roam-mcp.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/roam-mcp.err.log</string>
</dict>
</plist>
```

(Apple Silicon Homebrew puts `npx` at `/opt/homebrew/bin/npx`; Intel Homebrew at `/usr/local/bin/npx`; nvm users will have a path under `~/.nvm/versions/node/...`.)

Load and verify:

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/dev.roam-mcp.local.plist
launchctl print gui/$UID/dev.roam-mcp.local | grep -E 'state|pid'
curl http://localhost:8787/check
```

Reload after editing the plist or `wrangler.toml`:

```bash
launchctl bootout gui/$UID/dev.roam-mcp.local
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/dev.roam-mcp.local.plist
```

Logs land at `/tmp/roam-mcp.log` and `/tmp/roam-mcp.err.log`.

**B. Manual run (one-shot)**

```bash
cd ~/code/roam-mcp
npx wrangler dev
```

Same endpoint, but only while the terminal stays open. Useful for ad-hoc testing or when iterating on `wrangler.toml`.

### Alternative MCP servers / v1 fallback

If you don't want to deploy a Cloudflare Worker, you have two paths:

- **Local stdio MCP server** — any MCP server that exposes the same `roam_*` tool surface (e.g. the npm-distributed [`roam-research-mcp`](https://www.npmjs.com/package/roam-research-mcp) by 2b3pro) can be registered with Claude Code via stdio transport. Tool names and parameters may differ slightly between implementations; verify by calling each `roam_*` tool from a Claude Code session before running the skills, and rename or wrap as needed. The skills assume the exact tool names listed at the top of this section.
- **No Roam at all** — install v1 of this plugin, which writes filesystem markdown and has no MCP dependency. Pin the marketplace install to commit [`c5f2ce5`](https://github.com/kfchou/wiki-skills/tree/c5f2ce5). v1 and v2 are different plugins under the same name; do not run them against the same wiki.

## Installation

```bash
/plugin marketplace add kfchou/wiki-skills
/plugin install wiki-skills@kfchou/wiki-skills
```

## Skills

| Skill | Description |
|---|---|
| `wiki-init` | Bootstrap a new Roam-backed wiki for any domain — creates `[[Wiki Schema]]`, `[[Wiki Index]]`, `[[Wiki Overview]]`. |
| `wiki-ingest` | Add a source (paper, URL, file, transcript, …). Three modes: ingest now, queue for later (`#wiki-ingest-queue`), or process the queue. Uploads extracted text paragraph-per-block under `Raw Text::`, creates entity/concept pages, runs the backlink audit. |
| `wiki-query` | Ask a question — answers cite via `[[Page]]` and `((uid))` block refs into `Raw Text::` excerpts; offers to queue gap-driven follow-ups for ingest. |
| `wiki-lint` | Health audit via Datalog: orphans, stale claims, broken refs, missing pages, contradictions, and ingest-queue progress. |
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

### What you can ingest

| Source kind | Examples | Raw Text chunking |
|---|---|---|
| `paper` | arXiv PDF, journal article, ADR, RFC | one block per paragraph (or section → paragraph children) |
| `article` | blog post, news, doc page | one block per paragraph |
| `transcript` | podcast, video, interview, meeting | **one block per speaker turn** |
| `code` | source file, config, snippet | **one block per logical unit** (function/class/decl) with `path:line-range` prefix |
| `book-excerpt` | book chapter, manual section | one block per paragraph (excerpts only — copyright-aware) |
| `thread` | Twitter/X, HN, mailing list | **one block per post or reply**, with reply chains as parent/child |
| `dataset` | CSV/JSON sample, OpenAPI spec | structural summary + sample rows (full file → `raw/`) |
| `notes` | meeting notes, email, Slack export | one block per paragraph or message |
| `other` | anything else | LLM judgment, with a `Chunking note::` block recording the choice |

Out of scope: ASR (audio/video without an existing transcript), OCR (image-only sources), and full uploads of large datasets or copyrighted full texts. For each, cite the source and excerpt the relevant parts.

### Ingest queue

You can capture sources to ingest later — anywhere in Roam — by dropping a TODO block:

```
{{[[TODO]]}} Ingest: <title or url> #wiki-ingest-queue
  URL:: https://...
  Source kind:: paper
  Reason:: <why this is worth ingesting>
  Queued:: April 25th, 2026
```

The minimum is `{{[[TODO]]}} <url> #wiki-ingest-queue`. Other skills also seed the queue:

- `wiki-ingest` surfaces cited references and skipped sections after each run, and offers to queue them
- `wiki-query` offers to queue gap-driven follow-ups whenever an answer reveals missing source coverage

Run `wiki-ingest` with no source argument to **process the queue** — the skill pulls pending items, lets you pick which to ingest, runs the full ingest flow on each, and threads `Ingested as:: [[<source page>]]` back under the queue TODO. `roam_search_by_status("TODO")` filtered by `#wiki-ingest-queue` shows the pending backlog at any time; `wiki-lint` reports queue progress on each run.

### Page metadata

Roam has no YAML frontmatter. Each page starts with **flat top-level attribute blocks** of the form `Key:: value`:

```
Type:: #wiki-source
Source:: https://arxiv.org/abs/1706.03762
Source kind:: paper
Tags:: #ml #transformer
```

…followed by section blocks (`Summary`, `Key Takeaways`, `Entities & Concepts`, `Raw Text`, …) as siblings.

Last-edit time is **not** written as a `Updated::` block. Roam tracks `:edit/time` automatically on every block (visible in the right-sidebar block menu and queryable via `roam_datomic_query`); writing manual `Updated::` blocks just produces accumulating noise because there is no mutate API.

### Cross-references

- `[[Page Title]]` — Roam-native page reference (no slug munging)
- `((uid))` — block reference for citing a specific paragraph or quote
- `{{embed: ((uid))}}` — embed the original block inline

`wiki-query` and `wiki-update` use `((uid))` aggressively so every claim traces back to source text in `Raw Text::`.

### Outliner discipline

Every skill writes Roam blocks **one idea per block**. Multi-value attributes are parent attribute blocks with one child block per value:

```
Sources::
  [[Source A]]
  [[Source B]]
  [[Source C]]
```

…not `Sources:: [[a]], [[b]], [[c]]` packed into a single block string. The same rule governs `Pages updated::`, `Backlinks added on::`, `Tags::` (when more than 2-3), `Categories::`, and any other multi-value attribute. Structured payloads (`Reason: …; Source: …`) likewise become a parent block with `Reason::` and `Source::` children. This is what lets Roam's outliner, attribute table, and ref index actually do their job.

### Typical Workflow

```
wiki-init          → bootstrap a new wiki
wiki-ingest        → add a source (Mode A) / queue for later (Mode B) / process the queue (Mode C)
wiki-query         → ask questions; cite ((uid)); offer to queue gap follow-ups
wiki-lint          → periodic health check (every 5-10 ingests); reports ingest-queue progress
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
| `#ai` | Auto-injected by roam-mcp on every write by default. Disable per MCP entry with the `X-Roam-Ai-Tag: false` header (see [Per-project Roam graph](#per-project-roam-graph-header-override)). When left on, use the `#wiki-*` tags below for filtering. |
| `#wiki-meta` | Schema, index, overview, lint reports |
| `#wiki-source` | Source pages (papers, articles, etc.) |
| `#wiki-entity` / `#wiki-concept` | Entity and concept pages |
| `#wiki-analysis` | Saved query answers |
| `#wiki-page` | Generic wiki page (codebase modules, flows, decisions) |
| `#wiki-log` | Daily-note operation log entries (sub-tagged `#wiki-ingest`, `#wiki-query`, `#wiki-lint`, `#wiki-update`) |
| `#wiki-change-request` | Queued edits for the user to apply manually (from `wiki-update` or `wiki-lint`) |
| `#wiki-ingest-queue` | Sources captured for later ingest. Drop `{{[[TODO]]}} <url> #wiki-ingest-queue` anywhere; `wiki-ingest` (no args) processes the queue |

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
