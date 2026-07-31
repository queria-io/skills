# Splitting a dataset's caveats: a worked example

Queria's e-Stat dataset is where the rule in `SKILL.md` came from. It holds Japanese
government statistics across 6 schemas and about 30 tables. Before the pass, everything
worth knowing sat at the top.

```
dataset.yml     one prose block carrying caveats for all 6 schemas
*.table.yml     one line each, mostly restating the title
                no descriptions on the fields at all
```

Nothing was wrong, exactly. It was all at the layer where the fewest readers look and
where the most statements are only sometimes true.

## Sentence by sentence

Each line below was in that block. The rule — *the smallest thing it is true of* —
moves it somewhere specific.

| The sentence | True of | Moved to |
| --- | --- | --- |
| "Values are in one `value` column, so filter by indicator before summing" | the `value` and `cat01` columns of the 22 SSDS tables | those columns' `description` |
| "Units differ per indicator, so summing across them is meaningless" | the `unit` column | that column's `description` |
| "`area` includes a national total row" | one column of the prefecture tables | that column's `description` |
| "Suppressed rows carry 0, which does not mean zero" | the `value` column of the 4 census tables | that column's `description` |
| "Boundary polygons exclude water-surface survey areas" | one table | that table's `description` |
| "Census `area` and boundary `key_code` are the same code system" | two schemas | stays at the dataset's `description` |
| "Municipality codes are 5 digits, not the 6-digit local government code" | two schemas | stays at the dataset's `description` |

The last two are the only ones you cannot say by looking at a single table. They are what
the dataset layer is for.

## What the column descriptions became

Not a restatement of the name. The thing that makes someone's query wrong if they do not
read it:

```yaml
- name: cat01
  description: >-
    Indicator code. One letter (the table's category, A–K) plus 4–6 digits; longer codes
    are breakdowns of shorter ones, and both sit in this column. Summing without filtering
    on cat01 double-counts.
- name: value
  description: >-
    The statistic. See unit for what it is measured in. Non-numeric source values ("-",
    "X") land as NULL.
```

## What stayed

Only the last two rows above. Both are about how two schemas relate, so there is nowhere
smaller to put them. Everything else moved down, and the dataset's description went back
to answering "what is this" for someone scanning a list.

## The keywords

The dataset classified itself in four words, which is not how anyone searches for it. The
names people actually type went in too:

```yaml
keywords: [統計, 政府統計, e-Stat, 社会・人口統計体系, SSDS, 国勢調査, 小地域, 消費者物価指数, 人口, 社会]
```

## The tell

If the dataset's description is much longer than the descriptions on its tables and
columns, the split has not been done. A long dataset description is not a sign of a
well-documented dataset; it is a sign that nobody decided where anything goes.
