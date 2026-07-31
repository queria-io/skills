---
name: queria-describe-dataset
description: Decide which layer of a Queria dataset each thing you know belongs to — the column, the table, or the dataset — and write keywords and synonyms that do not overlap. Use when writing or fixing descriptions in dataset.yml / *.table.yml, or keywords and synonyms (説明・メタデータの書き方), or when a dataset's caveats have all piled up in one place. Not for getting a dataset built or published (queria-publish-dataset), and not for exploring data that is already published (queria).
---

# Describing a Queria dataset

A Queria dataset has three layers to write down what you know — the column, the table
and the dataset — plus the cookbook manuscripts in the repository's `docs/*.md`.

Deciding case by case means everything ends up in one of them. Queria's own e-Stat
dataset had caveats covering 6 schemas and 30 tables piled into the coarsest layer there
is. That is sorting by length, not by meaning.

Where `queria-publish-dataset` gets the dataset into the catalog, this skill decides
**which layer each sentence belongs to**.

## Write it where it is true

Pick the smallest thing the sentence is true of. That decides it — there is nothing else
to weigh.

| The sentence is true of | Where it goes |
| --- | --- |
| One column | that column's `description` |
| One table | that table's `description` |
| More than one table | the dataset's `description` |
| What people call this dataset | `keywords` / `ai_context.synonyms` (below) |
| Something you can run as SQL | a cookbook manuscript in `docs/*.md` |

**Once it is written small, do not repeat it upward.** What stays at the top is only what
you cannot say without comparing several things. A column note copied into the dataset's
description means one of the two copies goes stale.

## What is worth writing

1. Grain — what one row represents
2. Definition, source, how it was derived
3. How it differs from something similar
4. What breaks an aggregation or a join
5. What is outside the coverage

4 and 5 are what people get wrong by not reading, and they are usually the bulk of it.
Push them down to the smallest layer where they hold.

- "This column mixes totals with their breakdown" → that column's `description`
- "This table's grain differs from the boundary data" → that table's `description`
- "These two schemas join at different grains" → the dataset's `description`

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

## SQL belongs in the cookbook

Do not put runnable SQL in a description. It goes in the repository's `docs/*.md`
manuscripts, which are picked up as the dataset's cookbook. A description is what keeps
someone from getting it wrong; a manuscript is what they can run.

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
- Did any SQL end up in a description?
- Does `uvx queria validate` still pass?

Full reference: https://docs.queria.io/publish/writing-descriptions — if this skill and
that page disagree, the page is right.

Done. Next: publish the change with `queria-publish-dataset`.
