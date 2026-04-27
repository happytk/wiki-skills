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

Read the schema for the **`Language::`** (default Korean), index categories, and conventions.

## Language policy

Answer in the wiki's `Language::` regardless of the source pages' original language. If the user phrases the question in another language, mirror the user's question language for the conversational reply but write the **filed analysis page** (if the user accepts the save offer) in the wiki language. Original-language verbatim is preserved only via `((uid))` citations and `{{embed: ((uid))}}` blocks pointing into source `Raw Text::`.

Concretely: if the wiki has English `Raw Text::` paragraphs and Language:: is Korean, the answer reads as Korean prose with English `((uid))` excerpts inline — the user gets Korean synthesis with one-click access to the English original.

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

- **Is in the wiki's `Language::`** (Korean by default) for all prose. Source-original text appears only inside `((uid))` excerpts or `{{embed: ((uid))}}` blocks
- Is grounded in the Roam pages and blocks you read
- Cites every claim using **`[[Page Title]]`** for topical references and **`((uid))`** for specific quotes or precise claims drawn from a particular `Raw Text::` block
- Inlines `{{embed: ((uid))}}` when the original phrasing carries weight (a quote, a number, a precise definition) — the embed renders source-language verbatim alongside your wiki-language synthesis
- Notes agreements and disagreements between pages
- Flags gaps explicitly (in the wiki language): "the wiki has no page on X" / "[[Page]] doesn't cover Y yet"
- For each flagged gap, names a concrete next step (a paper, URL, file, or question) — this is what step 5 will offer to queue

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
3. Build the page with attribute blocks at depth 0 and section blocks below. Multi-value attributes use parent + children; one idea per block.

   ```
   Type:: #wiki-analysis
   Question:: <verbatim user question>
   Sources::
     [[<page1>]]
     [[<page2>]]
     ...

   Answer
     <paragraph 1 — one block, with ((uid)) and [[Page]] citations>
     <paragraph 2 — one block>
     {{embed: ((uid-of-key-quote))}}     (its own block)

   Open Follow-ups
     <follow-up question 1 — one block>
     <follow-up question 2 — one block>
     <suggested source to ingest — one block>
   ```

   The `Answer` blocks should re-use the same `((uid))`s the synthesis cited so the analysis stays auditable to source.

4. Append to `[[Wiki Index]]` under the `Analyses` (or equivalent) category:

   ```
   [[<title>]] — <one-line summary> _(query <today>)_
   ```

5. Append to today's daily note. Distinct facts are sibling child blocks — never packed into a single comma-joined string.

   ```
   [[Wiki Schema]] query | <one-line question summary> #wiki-log #wiki-query
     Filed as:: [[<title>]]
     Page citations:: <count of [[Page]] refs>
     Block citations:: <count of ((uid)) refs>
     Pages cited::
       [[<page1>]]
       [[<page2>]]
   ```

If **no:**

Append to today's daily note:

```
[[Wiki Schema]] query | <one-line question summary> #wiki-log #wiki-query
  Filed:: not filed
```

### 5. Offer to queue gap-driven follow-ups

If the answer flagged any wiki gaps in step 3, present each flagged gap as a concrete follow-up and ask: **"Queue any of these to `#wiki-ingest-queue` for later ingest?"**

For each accepted gap, append a queue TODO. Drop it under today's daily note by default, or under the most relevant existing source/concept page if context makes that obvious:

```
{{[[TODO]]}} Ingest: <gap → concrete next step> #wiki-ingest-queue
  URL:: <url, if a specific source was identified>
  Source kind:: paper | article | ...   (best guess)
  Reason:: surfaced by query "<original question>"
  Suggested by:: wiki-query <today, ordinal>
  Queued:: <today, ordinal>
```

If the gap is a question that needs investigation rather than a specific source to ingest (e.g., "is X compatible with Y?"), append it to `[[Wiki Overview]]`'s `Open Questions` section as a plain block — those don't belong in the ingest queue.

After queuing, append a child block to the daily-note query log:

```
  Queue items added::
    ((<queue-todo-uid-1>))
    ((<queue-todo-uid-2>))
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
- **Comma-joining cited pages** — `Sources:: [[a]], [[b]], [[c]]` defeats the outliner. Use parent + child blocks.
- **Naming a gap without naming the next step** — "the wiki has no page on X" is incomplete. State what would close the gap (a specific URL, paper, file) so the queue offer in step 5 is actionable.
- **Queueing investigation questions instead of source-to-ingest items** — pure questions belong in `[[Wiki Overview]]` Open Questions. `#wiki-ingest-queue` is for concrete sources you'd hand to `wiki-ingest`.
