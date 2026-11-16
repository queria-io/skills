---
name: queria-publish-dataset
description: Publish a dataset of your own to Queria (data.queria.io) — register it, describe it in queria.yml and a Markdown file per table, build it with any tool you like, pack and push. Covers what queria metadata validate refuses and why, licence ids, and the two settings that silently produce a broken dataset. Use when you have data of your own (自分のデータ) and want it in Queria's catalog, or when queria metadata validate / pack, fdl commit or fdl push is failing. Not for querying data that is already published (that is the queria skill), and it does not decide what your descriptions should say (queria-describe-dataset does).
---

# Publishing a dataset on Queria

Queria hosts datasets in the FDL format (https://fdl.queria.io): a DuckLake catalog frozen
into one version per update. Publishing means two things: producing the data, and declaring
what it is. **How you produce the data is up to you** — dbt, a Python
script, or writing Parquet directly. This skill covers the declaring and the pushing.

There is no "official dataset" tier. The datasets already in the catalog belong to one
account and sit alongside yours on equal terms.

Where `queria-describe-dataset` decides **what each description should say**, this skill
gets the dataset into the catalog and keeps it there.

## Before you start

- A shell with the queria CLI (PyPI: `queria`) and fdl (PyPI: `fdl-cli`). With uv no install
  is needed: `uvx queria`, `uvx --from fdl-cli fdl`. The MCP connector cannot create or push datasets; in a chat without a shell, explain the
  steps below and leave running them to the user.
- `uvx queria login` — publishing needs an account; reading does not. fdl uses the same token.
- A build that produces data. Anything that writes into the attached catalog works.

## The loop

```bash
uvx queria create my-dataset        # register it (private unless you pass --public)
# write queria.yml and a Markdown file per table
uvx --from fdl-cli fdl clone https://data.queria.io/datasets/<handle>/my-dataset/ .fdl/work.sqlite
# run the build, writing into the working catalog (attach it with the statement clone prints)
FDL_WORKING=.fdl/work.sqlite uvx queria metadata pack   # check, then write .queria/pack/
uvx --from fdl-cli fdl commit .fdl/work.sqlite --dataset .queria/pack/dataset.json -m "Build" --author <name>
uvx --from fdl-cli fdl push .fdl/work.sqlite
```

`clone` copies the published version into a working catalog; `pack` checks the description
against Queria's rules and the data and writes the folder a version is made from
(dataset.json, README.md, the guides and documents); `commit` freezes the catalog with that
folder into a new version; `push` uploads it and Queria publishes it. A push is refused
when someone else pushed after your clone: clone again and rebuild.

Do not ask the user which build tool to standardise on. Use what the repository already
has; if there is nothing, the smallest thing that writes a table is enough.

We recommend, but do not require, building in three steps: raw (the data as fetched, the
only step that reads the fetched files), stg (one raw table typed, renamed and normalized) and
mart (joins, reshaping and the grain; the tables you declare live here). Keep fetch scripts
to fetching and converting file formats, and do the content transformations in stg and mart.
See https://docs.queria.io/en/publish/conventions#transformation-steps.

Write the tables people query as tables, not views (in dbt, `+materialized: table` for the
models you declare). A view is computed from its sources every time it is read, and
`fdl push` computes each view in full to profile it, so views make queries and pushes slow.
Views are fine for intermediate steps you do not declare.

## The description

Two kinds of file. Each names its format in `$schema`, and only files that do are read.

| File | Content |
| --- | --- |
| `queria.yml` (repository root) | the dataset itself |
| a Markdown file per table (any name, anywhere) | the table's description in the front matter, its guide below |

```yaml
# queria.yml
$schema: https://queria.io/schemas/dataset/v1.json
name: calendar
title: Japanese calendar
description: A date spine from 1955 to 2027 with holidays, weekdays and Japanese eras.
language: ja
licenses: [CC-BY-4.0]
contributors:
  - title: Cabinet Office
    roles: [rightsHolder]
```

```markdown
---
$schema: https://queria.io/schemas/table/v1.json
table: calendar.day
title: Japanese calendar
primary_key: [date]
fields:
  - name: date
    title: Date
    semantic: {role: entity, name: date}
---

Holidays follow the Cabinet Office's list.
```

`table:` says which table the file describes, so put it where it is easiest to keep in step,
such as next to the dbt model. Keys only Queria reads (themes, cover, a column's `semantic`,
`published`) sit at the same level as the rest. The root `README.md` is the dataset's
overview. Other documents are listed in `queria.yml` under `docs:`.

**Never write column types.** Types, column order and nullability are read from the data,
so they cannot drift from it. Writing them is rejected.

**Declare `primary_key`** with the list of columns that identifies one row.
`queria metadata validate` reads the data and rejects repeated or NULL key values, so a
failure there means either the data has duplicates or the declared grain is wrong.

Repositories still on `dataset.yml` and `*.table.yml` keep working with the same commands.
`queria metadata unpack <folder>` writes a version's description (what `fdl clone` gives, or
another pack) back into these files, so changes made on Queria come back into the code.

## What to infer, and what to ask

Keep going rather than stopping on thin input, except where a wrong guess is expensive.

| Field | What to do |
| --- | --- |
| `name` | Infer from the directory name, lowercase it, and echo it for confirmation |
| `language` | The language the *declarations* are written in, not the data. Default `ja` |
| `title` / `description` | Draft from what the data actually contains, then show it |
| `licenses` | **Always ask.** Never guess — an id outside the registry stops `validate` |
| visibility | Private unless the user explicitly says public. Do not offer to flip it |

## Errors

`queria metadata validate` (and `pack`) prints `level: source: message [code]`. Read the code, not the prose.

| What you see | Cause | Fix |
| --- | --- | --- |
| `licenses-missing` (warning) | No `licenses` | Fine, but the user reserves every right. Confirm they meant to |
| a license id is rejected | Free text instead of an id | Ask which licence; use the registry id |
| `attribution-missing` | Licence needs attribution, `contributors` absent | Add `contributors` with `roles: [rightsHolder]` |
| `schema-violation` | A key the format does not define, often a typo | Fix the key; the message names where |
| `fdl-rejected`, "described but not in the catalog" | `table:` names a table the build did not make | Fix the name, or build the table |
| "$schema … update queria" | The file is written for a newer format | `uvx queria@latest` |
| `fdl commit` refuses inlined rows | DuckLake inlined rows into the catalog | Set `DATA_INLINING_ROW_LIMIT` to 0, as clone's `ATTACH` does |
| `fdl push` is refused | Someone pushed after your clone | `fdl clone` again and rebuild |

## Verify as a reader

A push that succeeded is not the same as a dataset someone can use. Read it back through
the public path before reporting done:

```bash
uvx queria info my-dataset
uvx queria schema my-dataset
uvx queria sql "SELECT * FROM my-dataset.main.some_table LIMIT 5"
```

## Constraints

- Do not hand-edit `dataset.json`. `queria metadata pack` makes it from queria.yml and the table files, and `fdl commit` adds what the data says.
- Do not make a dataset public without being asked to.
- `queria delete` is immediate. Confirm the name with the user before running it.

Full reference: https://docs.queria.io/publish

Done. Next: `queria-describe-dataset`, to write the descriptions that make the dataset
usable by someone who did not build it.
