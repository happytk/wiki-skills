---
name: wiki-lint
description: Use when auditing a Roam-backed wiki for health issues — orphan pages, stale claims, broken references, missing pages, contradictions, and coverage gaps. Run after every 5-10 ingests. Uses Datalog queries against the Roam graph plus LLM reading for semantic checks.
---

# Wiki Lint

Audit the Roam wiki. Produce a categorized report as a new `[[Lint <date>]]` page. Offer concrete fixes (which queue as `#wiki-change-request` TODOs since roam-mcp can't mutate). Always log to today's daily note.

## Pre-condition

Locate the wiki:
1. `roam_fetch_page_by_title("Wiki Schema")`
2. Fall back to `roam_search_by_text("Wiki Schema")`
3. If neither finds it, tell the user to run `wiki-init` first

Read the schema for the **`Language::`** (default Korean), category list, and conventions.

## Language policy

The lint report page (`[[Lint <date>]]`), every `Fix::` recommendation, contradiction descriptions, coverage-gap suggestions, and the daily-note log line are written in the wiki's `Language::`. Page titles and `((uid))` citations from offending blocks are quoted as-is (they're tokens, not prose).

## Process

### 1. Build the page inventory

Use Datalog (via `roam_datomic_query`) so you don't have to fetch pages one at a time:

**All wiki-managed pages** (we tag them `#wiki-source`, `#wiki-entity`, `#wiki-page`, `#wiki-analysis`, `#wiki-meta`):

```clojure
[:find ?title ?uid ?edit
 :where
 [?p :node/title ?title]
 [?p :block/uid ?uid]
 [?p :edit/time ?edit]
 [?p :block/children ?c]
 [?c :block/refs ?tag]
 [?tag :node/title ?tagname]
 [(contains? #{"wiki-source" "wiki-entity" "wiki-page" "wiki-analysis" "wiki-meta"} ?tagname)]]
```

This gives `(title, uid, edit-time)` triples for all wiki pages. Cache the set of titles for orphan detection.

### 2. Run all checks

> **Datalog correctness caveat:** the queries below are best-effort against the Roam pull schema (`:node/title`, `:block/string`, `:block/uid`, `:block/children`, `:block/refs`, `:edit/time`). On the first lint run, sanity-check each query's output against a few known pages before trusting the report.

**🔴 Errors (must fix)**

- **Pages referenced but never created** — Roam treats unresolved `[[Foo]]` as a virtual page with no content. Find blocks that ref a page entity whose own `:block/children` set is empty:

  ```clojure
  [:find (distinct ?reffed-title)
   :where
   [?b :block/refs ?p]
   [?p :node/title ?reffed-title]
   [(missing? $ ?p :block/children)]]
  ```

  These are the "broken links" of the Roam world. Recommend create-or-remove.

- **Missing `Type::` attribute** — for each wiki page, check that a `Type:: #wiki-*` block exists as a direct child. A page missing it is malformed:

  ```clojure
  [:find ?title
   :where
   [?p :node/title ?title]
   (not-join [?p]
     [?p :block/children ?c]
     [?c :block/string ?s]
     [(clojure.string/starts-with? ?s "Type:: #wiki-")])
   ;; restrict to wiki-managed pages — join via the same Type:: query elsewhere
  ]
  ```

  Note: `Updated::` is **not** a required attribute. Roam tracks last-edit time automatically via `:edit/time` on every block — the stale-claim check below uses that directly. Manual `Updated::` blocks were a v2.0 design that produced accumulating noise; the current spec drops them.

**🟡 Warnings (should fix)**

- **Legacy flat layout (advisory)** — content pages (`#wiki-source`, `#wiki-entity`, `#wiki-concept`, `#wiki-page`, `#wiki-analysis`) where meta attributes (`Source::`, `Sources::`, `Source kind::`, `Tags::`, `Aliases::`, `Category::`) sit as direct children of the page rather than nested under a `Meta` parent. The wiki still functions correctly — Roam's attribute index is depth-tolerant — so this is **advisory only**. Surface the page so the user can opt to re-group it (manually or via wiki-update) when convenient. Do **not** auto-rewrite. `Type::` itself stays at depth 0 by design and is exempt.

  ```clojure
  [:find ?title
   :where
   [?p :node/title ?title]
   [?p :block/children ?type-block]
   [?type-block :block/string ?ts]
   [(re-find #"^Type:: #wiki-(source|entity|concept|page|analysis)" ?ts)]
   [?p :block/children ?attr]
   [?attr :block/string ?as]
   [(re-find #"^(Source|Sources|Source kind|Tags|Aliases|Category)::" ?as)]
   (not-join [?p]
     [?p :block/children ?meta]
     [?meta :block/string "Meta"])]
  ```

  Pure-navigation `#wiki-meta` pages (`Wiki Index`, `Wiki Overview`) are exempt — they're dashboards and stay flat by convention.

- **Orphan pages** — wiki pages with zero inbound `:block/refs` from anywhere except `[[Wiki Index]]` and the daily-note log. Approximate with:

  ```clojure
  [:find ?title
   :where
   [?p :node/title ?title]
   [?p :block/children ?c]
   [?c :block/refs ?meta]
   [?meta :node/title "wiki-page"]   ;; or wiki-source/wiki-entity
   (not-join [?p]
     [?b :block/refs ?p]
     [?b :block/page ?bp]
     [?bp :node/title ?bp-title]
     [(not= ?bp-title "Wiki Index")]
     [(not (clojure.string/starts-with? ?bp-title "April"))])]   ;; rough exclusion of daily notes
  ```

  Tune the daily-note exclusion to your locale's date format. This query is best-effort; verify with one or two known orphans before bulk action.

- **Contradictions** — LLM check, not Datalog. For each wiki page modified in the last lint window, fetch and read; flag claims that conflict with claims in other pages on the same entity (different dates, counts, names, relationships).

- **Stale claims** — Datalog: pages with `:edit/time` older than 90 days that contain text matching `current|latest|recent|state-of-the-art` or any year literal two-or-more years old:

  ```clojure
  [:find ?title ?edit
   :where
   [?p :node/title ?title]
   [?p :edit/time ?edit]
   [(< ?edit <90-days-ago-ms>)]
   [?p :block/children ?c]
   [?c :block/string ?s]
   [(re-find #"(?i)current|latest|recent|state-of-the-art|202[0-3]" ?s)]]
  ```

**🔵 Info (consider addressing)**

- **Missing concept pages** — `[[Foo]]` references that appear 3+ times across the wiki but `[[Foo]]` has no content of its own. Reuse the broken-links query and add a count threshold.
- **Coverage gaps** — open questions listed in `[[Wiki Overview]]` (`Open Questions` section) that could be answered by a web search or a new ingest. LLM judgment.
- **Missing cross-references** — pages that both discuss `[[Entity]]` but don't link to each other. Compute by joining `roam_search_for_tag(<entity>)` results in code.
- **Ingest queue status** — `roam_search_by_status("TODO")` filtered by `#wiki-ingest-queue` for pending count + oldest items; `roam_search_by_status("DONE")` filtered by `#wiki-ingest-queue` for processed count since last lint. Flag any TODO items whose `Queued::` date is more than 60 days ago — likely abandoned and worth pruning.

### 3. Write the lint report

Today's date in ordinal format → page title `Lint April 25th, 2026`.

`roam_create_page("Lint <ordinal-date>")`, then build the report as a block-tree. Always write — do not ask permission. **One fact per block** — every offending page, missing attribute, claim, and conflicting page is its own block under a parent. No comma-joined lists inside one block string.

```
Type:: #wiki-meta #lint

Meta
  Run date:: <today, ordinal>

Notes
  Summary
    🔴 Errors:: <N>
    🟡 Warnings:: <N>
    🔵 Info:: <N>

  🔴 Pages Referenced But Not Created
    [[Missing Foo]]
      Referenced from::
        [[Source Page A]]
        [[Source Page B]]
      Fix:: create [[Missing Foo]] or replace the references

  🔴 Missing Required Attributes
    [[Page]]
      Missing::
        Type::

  🟡 Legacy Flat Layout (advisory)
    [[Page]]
      Detected meta attributes at depth 0::
        Source::
        Tags::
      Fix:: re-group under a Meta parent when convenient. The wiki still functions; this is cosmetic. wiki-update can move the blocks via roam_move_block when mutation is enabled.

  🟡 Orphan Pages
    [[Slug]]
      Status:: no inbound references outside [[Wiki Index]]
      Fix:: add link from a related page, or delete if no longer relevant

  🟡 Contradictions
    Conflict on <topic>
      [[Page A]]
        Claim:: "<claim>"
      [[Page B]]
        Claim:: "<conflicting claim>"
      Recommendation:: <which to trust, or "investigate further">

  🟡 Stale Claims
    [[Page]]
      Last edited:: <date from :edit/time>
      Trigger:: contains "latest"
      Fix:: re-verify the claim; any corrective edit will refresh :edit/time, or queue a wiki-update

  🔵 Missing Concept Pages
    [[Foo]]
      Reference count:: N
      Fix:: run wiki-ingest or create a stub via wiki-update

  🔵 Coverage Gaps
    Open question from [[Wiki Overview]]
      Question:: "<question>"
      Suggestion:: search for <X> or ingest <source type>

  🔵 Missing Cross-References
    [[Entity]]
      Discussed by::
        [[Page A]]
        [[Page B]]
      Issue:: pages do not link to each other
      Fix:: queue [[Entity]] reference under the relevant section of each page

  🔵 Ingest Queue Status
    Pending:: <N>     ((roam_search_by_status("TODO") + #wiki-ingest-queue))
    Processed since last lint:: <M>     ((roam_search_by_status("DONE") + #wiki-ingest-queue, filtered by Queued::/Ingested on:: date))
    Oldest pending::
      ((<queue-todo-uid>))   Queued <date>, Suggested by <source>
      ((<queue-todo-uid>))   ...
    Stale queue items (queued > 60 days ago, still TODO)::
      ((<queue-todo-uid>))
      ((<queue-todo-uid>))
```

The Ingest Queue Status section is informational — surfaces backlog and progress. No fix offer; the user runs `wiki-ingest` (Mode C) to process pending items, or removes stale ones manually in Roam.

Append to `[[Wiki Index]]` under a `Maintenance` category (create the category if it doesn't exist):

```
[[Lint <ordinal-date>]] — <N errors, N warnings, N info> _(lint <date>)_
```

### 4. Offer concrete fixes

Mutation availability decides the fix path. **Probe first** (same as wiki-update — try a no-op `roam_update_block` or check the tool list for `roam_*_block` mutation tools). Then for each fixable category:

**With mutation enabled (`X-Roam-Mutate: true`):** apply the fix in place after per-fix confirmation. Show the exact `roam_*` call before executing.

| Lint finding | Fix call |
| --- | --- |
| Missing cross-reference between `[[A]]` and `[[B]]` | `roam_create_block(content="[[B]]", page="A", parent_uid=<section uid>)` (additive — no mutation needed) |
| Duplicate legacy `Updated::` blocks (carry-over from v2.0 wikis) | Keep one block, `roam_delete_block` the rest. Confirm count before deleting. |
| `undefined` blocks (pre-guardrail residue) | `roam_delete_block` per offending uid. These cannot have meaningful descendants — safe to cascade. |
| Stale claim with a clear correction (e.g. "GPT-4" → "GPT-5" sourced) | `roam_update_block(uid, new_content)`. Always cite source via the wiki-update flow — consider routing to wiki-update for richer per-change confirmation. |
| Orphan page with no inbound refs | Three choices: (a) add `Status:: orphan` attribute via `roam_create_block` (keep, mark for review), (b) `roam_rename_page` to merge into a related page (then user fills in via wiki-ingest), or (c) `roam_delete_page(title=<orphan>)` to remove entirely (destructive — confirm twice; never use on pages with any inbound refs). |
| Block in wrong section | `roam_move_block(uid, parent_uid=<correct section uid>, order=<n>)`. |
| Pages referenced but not created | `roam_create_page(<title>)` + a stub Type:: attribute. The user can populate later via wiki-ingest. |
| Stub page that should be merged into another | `roam_rename_page(title=<stub>, new_title=<canonical>)` consolidates into the canonical page (Roam merges blocks). Confirm and surface inbound-ref count from `roam_search_for_tag` first. |
| Legacy flat layout (advisory) | `roam_create_block(content="Meta", page=<title>, order=1)` then `roam_move_block(uid=<attr-uid>, parent_uid=<meta-uid>)` for each meta attribute, and the same with a `Notes` parent for content sections. **Always opt-in per page** — never bulk-rewrite. Skip this row entirely if the user prefers to leave existing pages alone; it's purely cosmetic. |

**Without mutation (legacy queue mode):** queue each fix as a `{{[[TODO]]}} … #wiki-change-request` block under the affected page (the v2.0 behavior). Tell the user clearly that mutation was unavailable and recommend enabling `X-Roam-Mutate: true`.

Examples (legacy mode):

```
{{[[TODO]]}} Add [[Page B]] reference under <section name> #wiki-change-request
  Source:: lint report [[Lint <ordinal-date>]]
```

For orphan pages in legacy mode, propose `Status:: orphan` (attribute block, not a TODO — it's passive metadata).

### 5. Append to today's daily note

Always — do not ask permission. Counts and applied-fix references are siblings; never comma-joined.

```
[[Wiki Schema]] lint #wiki-log #wiki-lint
  Report:: [[Lint <ordinal-date>]]
  🔴 Errors:: <N>
  🟡 Warnings:: <N>
  🔵 Info:: <N>
  Fixes applied::
    [[Page A]] — added missing cross-reference
    [[Page B]] — deleted 3 duplicate Updated:: blocks
    [[Page C]] — moved <N> stranded blocks to correct section
    (or one block: "none")
  Fixes queued (legacy mode)::
    (only when mutation was unavailable; one block per queued TODO)
    (or one block: "none")
  Ingest queue::
    Pending:: <N>
    Processed since last lint:: <M>
```

## Common Mistakes

- **Trusting Datalog without sanity-checking** — verify each query against 1-2 known cases on the first lint run, especially the orphan query (the "exclude daily-note pages" predicate is locale-dependent).
- **Skipping the mutation probe** — without checking, you may queue TODOs even though `roam_update_block` etc. would work. Always probe at the start of a run.
- **`roam_delete_block` without confirming descendants** — deletes cascade. For any block with children, show the child count and confirm twice before calling.
- **Skipping the daily-note log** — it's the only audit trail.
- **Comma-joining offenders inside one block** — `[[a]], [[b]] missing Type::` defeats the outliner. One offender per block, with structured children for the failure details.
