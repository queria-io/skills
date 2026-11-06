---
name: queria-describe-dataset
description: Write short dataset, table and column descriptions for a Queria dataset and put the details in README.md and docs/*.md — decide what goes where and how long each is, fix pitfalls in the data's shape before describing them, write keywords, and name columns after schema.org or, for Japanese public data, the government vocabulary (GIF). Use when writing or fixing dataset.yml / *.table.yml descriptions, keywords, a dataset README (説明・README・メタデータの書き方), when naming columns or their Japanese titles (列名・英語名), or when descriptions have grown too long to read. Not for getting a dataset built or published (queria-publish-dataset), and not for exploring data that is already published (queria).
---

# Describing a Queria dataset

A Queria dataset has these places to write down what you know: the `description` of the
dataset, of each table and of each column, the repository's `README.md`, and `docs/*.md`
pages linked from the README. Descriptions appear as-is in lists and search results, so
they stay short. Everything else goes in the README.

Where `queria-publish-dataset` gets the dataset into the catalog, this skill decides **what
goes where, and how long it is**.

## Where it goes and how long

| Where | What | Length |
| --- | --- | --- |
| Dataset `description` | What the data is, and its source | 1–2 sentences plus 1 on the source (about 100 Japanese characters) |
| Table `description` | What one row represents, what is easy to get wrong | A 1-sentence summary (about 80 characters), a blank line, up to 3 caveat bullets |
| Column `description` | Definition, unit, code system | About 50 characters |
| `README.md` | Source and license, table list, example SQL, caveats, per-table details | No limit |
| `docs/*.md` | Explanations that do not fit in the README, linked from it | No limit |
| `MAINTAINING.md` | Build and processing steps for maintainers, not linked from the README | No limit |

Table lists and search results show only the **first paragraph** of a table description.
Write the summary so a reader can decide from it alone whether to use the table.

```yaml
description: |-
  建築着工統計調査の市区町村別・用途別の着工建築物の数・床面積・工事費予定額を、2011年から年次で持つテーブルです。

  - 市区町村と政令指定都市の区の行があるので、area_level で絞らずに合計すると二重に数えます
  - 工事費予定額は2019年までで、秘匿の値は NULL です
  - 地域コードは各年の時点のもので、合併で消えたコードは code.municipality にありません
```

Write descriptions and the README in plain です・ます Japanese for Japanese datasets.
Use 「データセット」「テーブル」「列」「バージョン」.

## Write it where it is true

Pick the smallest thing the sentence is true of.

| The sentence is true of | Where it goes |
| --- | --- |
| One column | that column's `description` |
| One table | that table's `description` (or its README section, if it does not fit in 3 bullets) |
| More than one table | the README's 「注意」 |
| What people call this dataset | `keywords` |
| Something you can run as SQL | the README's 「使い方」 or `docs/*.md` |

**Once it is written small, do not repeat it upward.** Two copies drift apart when only
one gets fixed.

## Fix pitfalls in the data first

A caveat is a sign the table could be shaped better. Before writing one, check whether the
shape can prevent the mistake:

- Rows of different grain in one table → split the table
- Only a 5-digit municipality code → also provide the 6-digit local government code
- Values for different years side by side → tell them apart by column name

Keep in the description only what comes from the source and cannot be changed:
suppression, coverage, the source's own inconsistencies.

## What stays out of descriptions

- Validation numbers ("matches the national total", "0 mismatches") → dbt tests or the README
- Full license text → `licenses` and the README's 「出典と利用条件」
- Processing and build steps → `MAINTAINING.md`
- Runnable SQL → the README's 「使い方」 or `docs/*.md`

## README layout

The README appears on the dataset page below the table list. Use this order:

1. `# <dataset title>` and 1–2 sentences on what the data is
2. `## 出典と利用条件` — links to the sources and the terms of use
3. `## テーブル` — table name, contents and row count
4. `## 使い方` — SQL that runs as-is against the published dataset
5. `## 注意` — caveats that span several tables
6. `## <schema>.<table>` — columns, grain and caveats per table
7. `## 更新` — `uv sync` and `bash scripts/build.sh`

`docs/*.md` pages linked from the README appear in the dataset page's 「ドキュメント」 tab.
When a dataset has many tables, move the per-table sections into `docs/*.md` and make the
README a table of contents that links them.

The README and `docs/` are published with the dataset by `fdl commit` (fdl-cli 0.19.0 or
later).

## Naming columns

Column names follow a vocabulary chosen by the kind of data:

- General data (maps, places, data from outside Japan) → schema.org terms in snake_case
- Japanese public-sector data (municipal and national statistics, registers) → GIF English
  names in snake_case; when a GIF term only adds a qualifier to a schema.org term
  (`latitudeOfRepresentativePoint`), use the schema.org one (`latitude`)
- `title` → the GIF Japanese name either way, the IMI term when GIF has none, the source's own
  heading when neither does

`queria-io/vocabulary` holds the vocabularies that tie names together. Fetch them from
`https://raw.githubusercontent.com/queria-io/vocabulary/main/data_elements.csv` and
`.../entities.csv`. Each row of `data_elements.csv` is one data element (what a column
represents): `name` (the name for Japanese public data), `title`, `schema_org`, `gif_term`,
`imi_term`, the `entity` it identifies, and `aliases` (other names in use for it).
`entities.csv` lists the names allowed in `semantic: { role: entity, name: ... }`.

1. The element is in `data_elements.csv` → use `name` for Japanese public data, or `schema_org`
   in snake_case for general data (falling back to `name`), plus its `title` and `entity`
2. Not there → find the term in your vocabulary and use it in snake_case
   (`nameKana` → `name_kana`)
3. In no vocabulary → name it yourself

Do not rename existing columns just to match. Abbreviations such as `lat` or `pref_code`
are the exception: add the new column beside the old one, mark the old one deprecated in
its `description`, and drop it later — never rename in place.

Column names differ; `semantic` entity names do not. Cross-dataset joins follow the entity
name, so the name of the column does not matter to them.

## keywords

`keywords` feeds the search index. Put in the words that classify the dataset **and the
other names people call it by**, so someone who does not know the official name can still
find it.

```yaml
keywords: [統計, 人口, 社会, e-Stat, 政府統計, 社会・人口統計体系, SSDS]
```

Words specific to one table go in that table's own `keywords`, not the dataset's.

## Doing a pass over a real dataset

1. Read the data before writing about it. Query the published dataset, or the local build.
   Do not describe what you assume is there.
2. Write each table's summary sentence, then its caveats. Anything that does not fit in 3
   bullets goes in the table's README section.
3. Move validation numbers, license text and build steps out of the descriptions.
4. Write the README in the layout above. Run every SQL example in 「使い方」.
5. Check that no sentence from the old descriptions was lost: each one is now in a
   description, the README, `docs/*.md` or `MAINTAINING.md`.

See `references/splitting-a-dataset.md` for a worked before/after on a real dataset.

## Before you stop

- Is the first paragraph of each table description one sentence that tells a reader whether to use the table?
- Could this sentence be said about something smaller than where you put it?
- Is the same thing written in two places?
- Could the shape of a table or column prevent the mistake instead of a caveat?
- Did SQL, validation numbers or build steps end up in a description?
- Does every SQL example in the README run?
- Is any new column an abbreviation listed in `aliases` in `data_elements.csv`?
- Does `uvx queria validate` still pass?

Full reference: https://docs.queria.io/publish/writing-descriptions (what goes where) and
https://docs.queria.io/publish/conventions (names, types, grain, keys) — if this skill and
those pages disagree, the pages are right.

Done. Next: publish the change with `queria-publish-dataset`.
