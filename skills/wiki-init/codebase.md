# Codebase Wiki Guide (Roam)

Use this when `wiki-init` is configured for a codebase domain. Covers page-title conventions, ingestion order, exploration strategy, and decision capture, all targeting a Roam Research graph via roam-mcp.

---

## Recommended Categories

```
Modules | APIs | Decisions | Flows
```

- **Modules** — major packages, services, or libraries (one page per unit of deployment or ownership)
- **APIs** — public interfaces, REST endpoints, RPC contracts, SDK surfaces
- **Decisions** — architectural and technical decisions, both documented and reconstructed
- **Flows** — end-to-end paths through the system (e.g. request lifecycle, data pipeline, auth flow)

---

## What to Ingest First

Order matters — later pages can cross-reference earlier ones via `[[Page Title]]`:

1. Top-level `README.md` and any `docs/` directory
2. Existing ADRs or decision docs (`docs/adr/`, `DECISIONS.md`, etc.)
3. CI/CD config (`Makefile`, `Justfile`, `.github/workflows/`) — reveals build model and deployment topology
4. Dependency manifests (`package.json`, `pyproject.toml`, `go.mod`) — reveals external surface area and key choices
5. Entry point files (`main.py`, `cmd/server/main.go`, `src/index.ts`) — reveals system shape

If formal documentation is sparse or missing, skip ahead to exploration (below) and treat the codebase itself as the source.

---

## Exploration Strategy

Work outward from the edges of the system:

1. **Entry points** — where does execution start? what are the top-level handlers?
2. **Module boundaries** — what are the coarse-grained units? who owns what?
3. **API surface** — what does the system expose? what does it consume?
4. **Key flows** — trace 2-3 representative user journeys end-to-end
5. **Decision log** — capture every non-obvious choice encountered while exploring (see below)

Resist documenting everything at once. Five deep pages beat fifty shallow ones.

---

## Capturing Decisions

`Decisions` pages are the most valuable output of codebase exploration, especially when formal documentation is sparse.

**Two types:**

- **Sourced** — a formal ADR or decision doc exists. Ingest it as a source; the wiki page synthesizes and cross-references affected modules/flows. The synthesis page should `((uid))`-cite the original ADR text uploaded under `Raw Text::` on the source page.
- **Reconstructed** — no formal doc exists. The decision is inferred from code structure, git history, dependency choices, or naming conventions. The wiki page is the first and only written record. Cite specific files/lines inline (e.g. `src/db/pool.ts:42`) and capture supporting commit SHAs.

Reconstructed decisions are often more valuable than sourced ones — they surface the implicit choices that were never written down.

**Watch for undocumented decisions while exploring:**
- Why is this library used instead of a more common alternative?
- Why is this boundary drawn here?
- Why does this flow bypass the usual path?
- Why is this module structured differently from its neighbors?

When you notice one, create a `Decisions` page immediately rather than noting it for later.

---

## Page Title Conventions

Roam pages use natural titles. For a codebase wiki, **use prefixes** because module names frequently collide with concept names (e.g. "Auth" the module vs. "Auth" the abstract concept).

| Prefix          | Example                            | Use for                           |
|-----------------|------------------------------------|-----------------------------------|
| `Mod: …`        | `Mod: Auth Service`                | Module/service pages              |
| `API: …`        | `API: User Endpoints`              | API surface pages                 |
| `Decision: …`   | `Decision: Postgres over MySQL`    | Decisions (sourced or reconstructed) |
| `Flow: …`       | `Flow: Checkout Request`           | End-to-end flows                  |

For non-codebase wikis, drop prefixes — Roam's `[[Attention Is All You Need]]` is the idiomatic form.

---

## Page Layout (Roam attribute blocks, not YAML)

Every codebase wiki page has flat top-level attribute blocks at depth 0:

```
Type:: #wiki-page
Category:: Module | API | Decision | Flow
Sources:: [[Source Page Title 1]], [[Source Page Title 2]]
Updated:: April 25th, 2026
Tags:: #auth #postgres
```

…followed by section blocks (`Summary`, `Boundaries`, `Decisions Affecting This`, `Cross-References`, etc.) as siblings.

When a claim ties to a specific source paragraph, cite via `((uid))` into that source's `Raw Text::` block.

---

## Handling Drift

Codebases change. Keep the wiki current by:

- Running `wiki-lint` after significant refactors to find stale claims (Datalog query on `:edit/time`)
- Using `wiki-update` when a module is renamed, split, or deleted (queues `#wiki-change-request` TODO blocks for you to apply manually)
- Noting the approximate date or git ref a claim was verified by adding a `Verified:: ~2026-01 (sha abc1234)` attribute block
- Flagging pages that need revisiting with a `Stale:: true` attribute block

---

## README vs. Wiki

This distinction must be actively maintained on every ingest and update.

**README** answers: *what is this, how do I run it, how do I contribute?*
**Wiki** answers: *why is it this way, how does it fit together, what tradeoffs were accepted?*

| Belongs in README | Belongs in wiki |
|---|---|
| Setup and installation | Architectural rationale |
| How to run tests | Module relationships and boundaries |
| Contributing guidelines | Decision history (sourced and reconstructed) |
| Environment variables | End-to-end flows |
| Quick-start examples | Cross-cutting concerns |

**The test:** If a PR reviewer would check "does the README reflect this change?", it belongs in the README. If explaining it requires cross-referencing multiple parts of the system, it belongs in the wiki.

**When ingesting a README, do two things:**

1. **Extract for the wiki** — structural signals: what modules exist, how the system is shaped, what the README reveals about design intent. Do not reproduce operational content. Link to the README instead: `See the project README for setup instructions.`

2. **Evaluate the README itself** — assess whether it covers what it should. Flag gaps and suggest edits:
   - Missing sections (no setup instructions, no contributing guide, no description of what the project does)
   - Outdated content (references to removed modules, stale commands)
   - Content that belongs in the wiki, not the README (lengthy architecture explanations, decision rationale)
   - Present suggested edits to the user and offer to apply them via the editing tools (not via roam-mcp — README edits land on disk)

**Subdirectory READMEs** (`src/auth/README.md`, etc.) follow the same pattern — synthesize design intent into the wiki, evaluate and suggest improvements to the file itself.

This rule is written into `[[Wiki Schema]]` at init time so it is enforced on every subsequent operation.

---

## Cross-Referencing Code

Anchor claims to specific locations using inline code paths:

```
The request router is defined in `src/server/router.ts` and delegates to handlers in `src/handlers/`.
```

Do not embed full code snippets in wiki pages. The wiki describes *what* and *why*; the codebase is the source of truth for *how*. If you must capture a snippet for analysis, upload it to the source page's `Raw Text::` section as one block per logical unit, then `((uid))`-cite from analysis pages.
