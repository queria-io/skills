# Splitting a dataset's caveats: a worked example

Queria's e-Stat dataset is where the rule in `SKILL.md` came from. It holds Japanese
government statistics across 6 schemas and about 30 tables. Before the pass, everything
that mattered lived in one field.

```
dataset.yml
  description              2 lines
  ai_context.instructions  18 lines   ← 6 schemas, 30 tables, all in here
*.table.yml
  description              a line each, mostly restating the title
  fields[].description     absent
```

Nothing was wrong, exactly. It was just all at the layer where the fewest readers look and
the most statements are only sometimes true.

## Sentence by sentence

Each line below was in that 18-line block. The rule — *the smallest thing it is true of* —
moves it somewhere specific.

| The sentence | True of | Moved to |
| --- | --- | --- |
| "Values are in one `value` column, so filter by indicator before summing" | the `value` and `cat01` columns of the 22 SSDS tables | those columns' `description` |
| "Units differ per indicator, so summing across them is meaningless" | the `unit` column | that column's `description` |
| "`area` includes a national total row" | one column of the prefecture tables | that column's `description` |
| "Suppressed rows carry 0, which does not mean zero" | the `value` column of the 4 census tables | that column's `description` |
| "Boundary polygons exclude water-surface survey areas" | one table | that table's `description` |
| "Census `area` and boundary `key_code` are the same code system" | two schemas | stays at the dataset's `instructions` |
| "Municipality codes are 5 digits, not the 6-digit local government code" | two schemas | stays at the dataset's `instructions` |

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

## What did not move

The dataset's `description` stayed short. It answers "what is this" for someone scanning a
list of datasets, and it is not where caveats belong.

`examples` stayed as three questions, matching headings in the cookbook manuscripts:

```yaml
examples:
  - 市区町村別の人口の推移を出したい
  - 小地域の年齢構成を地図に出したい
  - 消費者物価指数の推移を品目別に見たい
```

## The keyword overlap

`e-Stat` appeared in both `keywords` and `ai_context.synonyms`. It only ever means this
dataset, so it belongs in `synonyms` alone:

```yaml
keywords: [統計, 人口, 社会]
ai_context:
  synonyms: [e-Stat, 政府統計, 社会・人口統計体系, SSDS, 国勢調査, 小地域, 消費者物価指数]
```

## The tell

If the dataset's `instructions` is much longer than its `description`, the split has not
been done. Long `instructions` are not a sign of a well-documented dataset; they are a
sign that nobody decided where anything goes.
