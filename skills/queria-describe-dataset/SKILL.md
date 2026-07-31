---
name: queria-describe-dataset
description: Decide which layer of a Queria dataset each thing you know belongs to — a column's description, a table's description or ai_context, or the dataset's ai_context — and write keywords, synonyms and examples that do not overlap. Use when writing or fixing dataset.yml / *.table.yml descriptions, ai_context, keywords or synonyms (説明・メタデータの書き方), or when a dataset's caveats have piled up in one place. Not for getting a dataset built or published (queria-publish-dataset), and not for exploring data that is already published (queria).
---

# Describing a Queria dataset

A Queria dataset has several places to write down what you know: a column's
`description`, a table's `description` and `ai_context`, the dataset's `description` and
`ai_context`, and the cookbook manuscripts in the repository's `docs/*.md`.

Deciding case by case means everything ends up in one of them. Queria's own e-Stat dataset
had a 2-line `description` and an 18-line `ai_context.instructions` — caveats covering
6 schemas and 30 tables, piled into the coarsest layer there is. That is sorting by
length, not by meaning.

Where `queria-publish-dataset` gets the dataset into the catalog, this skill decides
**which layer each sentence belongs to**.

## Write it where it is true

Pick the smallest thing the sentence is true of. That decides it — there is nothing else
to weigh.

| The sentence is true of | Where it goes |
| --- | --- |
| One column | that column's `description` |
| One table | that table's `description` |
| Whether to reach for this table at all | that table's `ai_context.instructions` |
| More than one table | the dataset's `ai_context.instructions` |
| What people call this dataset | `keywords` / `synonyms` (below) |
| Something you can run as SQL | a cookbook manuscript in `docs/*.md` |

**Once it is written small, do not repeat it upward.** What stays at the top is only what
you cannot say without comparing several things. A column note copied into the dataset's
`instructions` means one of the two copies goes stale.

## What is worth writing

1. Grain — what one row represents
2. Definition, source, how it was derived
3. How it differs from something similar
4. What breaks an aggregation or a join
5. What is outside the coverage

4 and 5 are what people get wrong by not reading, and they are usually the bulk of it.
Push them down to the smallest layer where they hold.

- "This column mixes totals with their breakdown" → that column's `description`
- "This table's grain differs from the boundary data" → that table's `ai_context`
- "These two schemas join at different grains" → the dataset's `instructions`

## keywords vs synonyms

**Words that could apply to other datasets are `keywords`. Names that only ever mean this
one are `synonyms`.**

```yaml
keywords: [statistics, population, society]     # could apply to others
ai_context:
  synonyms: [e-Stat, SSDS, 政府統計]              # only ever means this one
```

Never put the same word in both. `synonyms` feeds the search index, so someone who does
not know the official name still finds it.

Keep `keywords` to a few that classify the dataset. Words specific to one table go in that
table's own `keywords`, not the dataset's.

## examples and the cookbook

`examples` holds **questions this dataset can answer**, one per line, in the language of
its users. No SQL.

```yaml
examples:
  - 小地域の年齢構成を地図に出したい
  - 市区町村別の人口の推移を出したい
```

The SQL that answers them goes in the repository's `docs/*.md`. Matching the wording turns
the two into a path.

## Keep ai_context at three fields

`instructions` / `synonyms` / `examples` match the fields recommended by Apache Ossie. It
is tempting to break caveats or out-of-scope uses into new keys — write them as prose
inside `instructions` instead. Ossie says of its current version, "Schema is mutable; do
not depend on this version in production", and the name `ai_context` is still under
discussion upstream. Inventing keys means fixing it twice.

`instructions` reads well in this order: what it is good for → what is easy to get wrong
(nulls, duplicates, digits, units) → the coverage and what falls outside it.

## Doing a pass over a real dataset

1. Read the data before writing about it. `uvx queria sql` against the published dataset,
   or query the local build. Do not describe what you assume is there.
2. Write each finding at the smallest layer where it holds.
3. Re-read what was already at the dataset level and push down anything that turned out to
   be about one table or one column.
4. Check `keywords` against `synonyms` for shared words.

See `references/splitting-a-dataset.md` for a worked before/after on a real dataset.

## Before you stop

- Could this sentence be said about something smaller than where you put it?
- Is the same thing written in two places?
- Do `keywords` and `synonyms` share a word?
- Did any SQL end up in `examples`?
- Does `uvx queria validate` still pass?

Full reference: https://docs.queria.io/publish/writing-descriptions — if this skill and
that page disagree, the page is right.

Done. Next: publish the change with `queria-publish-dataset`.
