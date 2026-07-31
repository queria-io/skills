---
name: queria-publish-dataset
description: Publish a dataset of your own to Queria (data.queria.io) — register it, declare it in dataset.yml and *.table.yml, build it with any tool you like, and push. Covers what queria validate refuses and why, licence ids, and the two settings that silently produce a broken dataset. Use when you have data of your own (自分のデータ) and want it in Queria's catalog, or when queria validate / queria push is failing. Not for querying data that is already published (that is the queria skill), and it does not decide what your descriptions should say (queria-describe-dataset does).
---

# Publishing a dataset on Queria

Queria hosts datasets as DuckLake catalogs. Publishing means two things: producing the
data, and declaring what it is. **How you produce the data is up to you** — dbt, a Python
script, or writing Parquet directly. This skill covers the declaring and the pushing.

There is no "official dataset" tier. The datasets already in the catalog belong to one
account and sit alongside yours on equal terms.

Where `queria-describe-dataset` decides **what each description should say**, this skill
gets the dataset into the catalog and keeps it there.

## Before you start

- The queria CLI (PyPI: `queria`). With uv no install is needed: `uvx queria`.
- `uvx queria login` — publishing needs an account; reading does not.
- A build that produces data. Anything that writes into the attached catalog works.

## The loop

```bash
uvx queria create my-dataset        # register it (private unless you pass --public)
# write dataset.yml and *.table.yml
uvx queria init                     # first time only: create the catalog you build into
uvx queria sync -- python main.py   # pull, run your build, push
uvx queria validate                 # check the declarations against the data
```

`sync` is `pull` → your build → `push`. Anything after `--` runs as the build.

Do not ask the user which build tool to standardise on. Use what the repository already
has; if there is nothing, the smallest thing that writes a table is enough.

## The declarations

Two kinds. Directory names are free — what a file declares comes from its contents
(`schema` and `name`), not from where it sits.

| File | Content |
| --- | --- |
| `dataset.yml` (repository root, required) | the dataset itself |
| `**/*.table.yml` (anywhere) | one table, or several |

```yaml
# dataset.yml
spec_version: "0.1"
name: calendar
title: Japanese calendar
description: A date spine from 1955 to 2027 with holidays, weekdays and Japanese eras.
language: ja
licenses: [CC-BY-4.0]
contributors:
  - title: Cabinet Office
    roles: [rightsHolder]
```

```yaml
# models/main/mart_calendar.table.yml
schema: main
name: mart_calendar
title: Japanese calendar
published: true
fields:
  - name: date
    title: Date
    semantic: { role: entity, name: date }
```

**Never write column types.** Types, column order and nullability are read from the data,
so they cannot drift from it. Writing them is rejected.

Tables sharing a column layout go in one file, with YAML anchors carrying the shared part:

```yaml
_fields: &fields
  - { name: area, title: Area code }
  - { name: value, title: Value, semantic: { role: measure, agg: sum } }
tables:
  - { schema: ssds, name: a_population, title: Population, fields: *fields }
  - { schema: ssds, name: b_land,       title: Land,       fields: *fields }
```

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

`validate` prints `level: source: message [code]`. Read the code, not the prose.

| What you see | Cause | Fix |
| --- | --- | --- |
| `licenses-missing` (warning) | No `licenses` | Fine, but the user reserves every right. Confirm they meant to |
| a license id is rejected | Free text instead of an id | Ask which licence; use the registry id |
| `attribution-missing` | Licence needs attribution, `contributors` absent | Add `contributors` with `roles: [rightsHolder]` |
| `table-not-declared` | Data has a table no `*.table.yml` covers | Declare it, or stop building it |
| dbt fails reading a `.yml` | `*.table.yml` under dbt's `model-paths` | Add `*.table.yml` to `.dbtignore` |
| No parquet is written at all | DuckLake inlined every row into the catalog | Set `DATA_INLINING_ROW_LIMIT` to 0 in the build |

The last two produce a dataset that looks fine and is not. Check for them before declaring
success.

## Verify as a reader

A push that succeeded is not the same as a dataset someone can use. Read it back through
the public path before reporting done:

```bash
uvx queria info my-dataset
uvx queria schema my-dataset
uvx queria sql "SELECT * FROM my-dataset.main.some_table LIMIT 5"
```

## Constraints

- Do not hand-edit `dist/dataset.json`. It is compiled from the declarations and the data.
- Do not make a dataset public without being asked to.
- `queria delete` is immediate. Confirm the name with the user before running it.

Full reference: https://docs.queria.io/publish

Done. Next: `queria-describe-dataset`, to write the descriptions that make the dataset
usable by someone who did not build it.
