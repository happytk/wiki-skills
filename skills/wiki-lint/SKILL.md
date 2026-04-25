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

Read the schema for category list and conventions.

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

- **Missing required attributes** — for each wiki page, check that `Type::` and `Updated::` blocks exist as direct children. A page missing them is malformed:

  ```clojure
  [:find ?title
   :where
   [?p :node/title ?title]
   [?p :block/children ?c]
   [?c :block/string ?s]
   [(clojure.string/starts-with? ?s "Type:: #wiki-")]
   ;; …then a not-clause for pages without an Updated:: child
  ]
  ```

  (Run a separate query for `Updated::` and set-difference in code.)

**🟡 Warnings (should fix)**

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

### 3. Write the lint report

Today's date in ordinal format → page title `Lint April 25th, 2026`.

`roam_create_page("Lint <ordinal-date>")`, then build the report as a block-tree. Always write — do not ask permission. **One fact per block** — every offending page, missing attribute, claim, and conflicting page is its own block under a parent. No comma-joined lists inside one block string.

```
Type:: #wiki-meta #lint
Updated:: <today, ordinal>

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
      Updated::

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
    Last updated:: <date>
    Trigger:: contains "latest"
    Fix:: re-verify and append an `Updated:: <today>` attribute block, or queue a wiki-update

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
```

Append to `[[Wiki Index]]` under a `Maintenance` category (create the category if it doesn't exist):

```
[[Lint <ordinal-date>]] — <N errors, N warnings, N info> _(lint <date>)_
```

### 4. Offer concrete fixes

For each fixable category, offer to queue a `#wiki-change-request` TODO under the affected page (since direct mutation isn't possible). Show the exact block content before queuing.

For example, for a missing cross-reference between `[[Page A]]` and `[[Page B]]`:

```
{{[[TODO]]}} Add [[Page B]] reference under <section name> #wiki-change-request
  Source:: lint report [[Lint <ordinal-date>]]
```

`roam_create_block(content=<above>, page="Page A")`. Apply only after per-fix confirmation.

For orphan pages, propose tagging:

```
Status:: orphan
```

(an attribute block, not a TODO — it's metadata, not an action).

### 5. Append to today's daily note

Always — do not ask permission. Counts and queued-fix references are siblings; never comma-joined.

```
[[Wiki Schema]] lint #wiki-log #wiki-lint
  Report:: [[Lint <ordinal-date>]]
  🔴 Errors:: <N>
  🟡 Warnings:: <N>
  🔵 Info:: <N>
  Fixes queued::
    [[Page A]] — added missing cross-reference
    [[Page B]] — flagged Status:: orphan
    (or one block: "none")
```

## Common Mistakes

- **Trusting Datalog without sanity-checking** — verify each query against 1-2 known cases on the first lint run, especially the orphan query (the "exclude daily-note pages" predicate is locale-dependent).
- **Mutating directly** — you can't. Every fix is a queued TODO or an appended attribute block.
- **Skipping the daily-note log** — it's the only audit trail.
- **Comma-joining offenders inside one block** — `[[a]], [[b]] missing Type::, Updated::` defeats the outliner. One offender per block, with structured children for the failure details.
