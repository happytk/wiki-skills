---
name: wiki-query
description: Use when asking a question against a Roam-backed wiki built with wiki-init and wiki-ingest. Always read Roam pages first; never answer from general knowledge. Cites with [[Page]] and ((uid)) block references.
---

# Wiki Query

Ask a question. Read the wiki in Roam. Synthesize with `[[Page]]` and `((uid))` citations into the original `Raw Text::` excerpts. Offer to file the answer back as a new Roam page.

## Pre-condition

Locate the wiki:
1. `roam_fetch_page_by_title("Wiki Schema")`
2. Fall back to `roam_search_by_text("Wiki Schema")` if missing
3. If neither finds it, tell the user to run `wiki-init` first

Read the schema for the index categories and conventions.

## Process

### 1. Read `[[Wiki Index]]` first

`roam_fetch_page_by_title("Wiki Index")`. Scan the full tree to identify which pages are likely relevant. Do NOT answer from general knowledge — the wiki is the source of truth, even if you think you know the answer.

### 2. Read relevant pages

For each candidate page:

1. `roam_fetch_page_by_title(<title>)` — pulls 3 levels deep, which covers attribute blocks + section parents + first-level children.
2. If the page has a `Raw Text::` section that was truncated by the depth limit, run a Datalog pull via `roam_datomic_query` to get all `Raw Text::` children (each block's `:block/uid` and `:block/string`):

   ```clojure
   [:find (pull ?c [:block/string :block/uid :block/order])
    :where
    [?p :node/title "<title>"]
    [?p :block/children ?s]
    [?s :block/string "Raw Text"]
    [?s :block/children ?c]]
   ```

3. Follow one hop of `[[Page]]` links if they look directly relevant. Don't go deeper than two hops without justification — it explodes context.

4. For broader topical recall, consider `roam_search_for_tag(<concept>)` to find every block that references a concept page.

### 3. Synthesize the answer

Write a response that:

- Is grounded in the Roam pages and blocks you read
- Cites every claim using **`[[Page Title]]`** for topical references and **`((uid))`** for specific quotes or precise claims drawn from a particular `Raw Text::` block
- Inlines `{{embed: ((uid))}}` when the original phrasing carries weight (a quote, a number, a precise definition)
- Notes agreements and disagreements between pages — flag them as `[[Page A]] says X but [[Page B]] says Y`
- Flags gaps: "the wiki has no page on X" or "[[Page]] doesn't cover Y yet"
- Suggests follow-up sources to ingest or questions to investigate

Format for the question type:
- Factual → prose with citations
- Comparison → table
- How-it-works → numbered steps
- What-do-we-know-about-X → structured summary with open questions

### 4. Always offer to save

After answering, ask:

> "Worth filing back as `[[<suggested page title>]]`?"

If **yes:**

1. `roam_fetch_page_by_title(<title>)` — confirm it doesn't already exist with content.
2. `roam_create_page(<title>)`.
3. Build the page with attribute blocks at depth 0 and section blocks below:

   ```
   Type:: #wiki-analysis
   Question:: <verbatim user question>
   Sources:: [[<page1>]], [[<page2>]], …
   Updated:: <today, ordinal>

   Answer
     <first paragraph block, with ((uid)) and [[Page]] citations>
     <second paragraph block, …>
     {{embed: ((uid-of-key-quote))}}

   Open Follow-ups
     <follow-up question>
     <suggested source to ingest>
   ```

   The `Answer` blocks should re-use the same `((uid))`s the synthesis cited so the analysis stays auditable to source.

4. Append to `[[Wiki Index]]` under the `Analyses` (or equivalent) category:

   ```
   [[<title>]] — <one-line summary> _(query <today>)_
   ```

5. Append to today's daily note:

   ```
   [[Wiki Schema]] query | <one-line question summary> #wiki-log #wiki-query
     Filed as:: [[<title>]]
     Citations:: <count of [[Page]] refs>, <count of ((uid)) refs>
   ```

If **no:**

Append to today's daily note:

```
[[Wiki Schema]] query | <one-line question summary> #wiki-log #wiki-query
  Filed:: not filed
```

## Citation Rules

- **`[[Page Title]]`** — use for topical attribution ("[[Transformer Architecture]] introduces self-attention")
- **`((uid))`** — use for any specific claim, definition, statistic, or quote that traces to one paragraph of source text. Always pair with the parent `[[Page]]` somewhere in the response so a reader can navigate even if Roam strips block-ref hover ("…as stated in `[[Attention Is All You Need]]` ((Abc12dEfG))").
- **`{{embed: ((uid))}}`** — use sparingly when the original phrasing must be preserved (a precise definition, an exact number, a contested quote).

**Block-uid stability caveat:** `((uid))` survives edits, but the surrounding context can drift if the source page is restructured. Pair every long-lived citation with the parent `[[Page]]` for context recovery.

## Common Mistakes

- **Answering from memory** — always read the wiki first. The wiki may contradict what you think you know, and that contradiction is valuable signal.
- **Skipping the save offer** — good query answers compound the wiki's value. Always offer.
- **`[[Page]]`-only citations** — a query response without `((uid))` citations is weak when source text exists in `Raw Text::`. Reach for the original.
- **Forgetting to log** — every query operation must land in today's daily note tagged `#wiki-log #wiki-query`, even when the answer is not filed.
- **Going deeper than two ref hops** — explodes context for marginal gain. Stop and ask the user if a deeper trace is needed.
