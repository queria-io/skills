---
name: queria-publish-dataset
description: Publish a dataset of your own to Queria (data.queria.io) — register it, declare it in dataset.yml and *.table.yml, build it with any tool you like, and push. Covers what queria validate refuses and why, licence ids, and the two settings that silently produce a broken dataset. Use when you have data of your own (自分のデータ) and want it in Queria's catalog, or when queria validate / fdl commit / fdl push is failing. Not for querying data that is already published (that is the queria skill), and it does not decide what your descriptions should say (queria-describe-dataset does).
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
# write dataset.yml and *.table.yml
uvx --from fdl-cli fdl checkout https://data.queria.io/datasets/<handle>/my-dataset/ .fdl/work.sqlite
# run the build, writing into the working catalog (attach it with the statement checkout prints)
FDL_WORKING=.fdl/work.sqlite uvx queria validate   # check the declarations against the data
uvx --from fdl-cli fdl commit .fdl/work.sqlite -m "Build" --author <name>
uvx --from fdl-cli fdl push .fdl/work.sqlite
```

`checkout` copies the published version into a working catalog; `commit` freezes it with the
declarations into a new version; `push` uploads it and Queria publishes it. A push is refused
when someone else pushed after your checkout: check out again and rebuild.

Do not ask the user which build tool to standardise on. Use what the repository already
has; if there is nothing, the smallest thing that writes a table is enough.

Write the tables people query as tables, not views (in dbt, `+materialized: table` for the
models you declare). A view is computed from its sources every time it is read, and
`fdl push` computes each view in full to profile it, so views make queries and pushes slow.
Views are fine for intermediate steps you do not declare.

## The declarations

Two kinds, in the directory `fdl commit` runs from. Directory names are free — what a file
declares comes from its contents (`schema` and `name`), not from where it sits. What only
Queria reads (themes, a column's `semantic`, the title's reading) goes under `annotations.queria`.

| File | Content |
| --- | --- |
| `dataset.yml` (repository root, required) | the dataset itself |
| `**/*.table.yml` (anywhere) | one table, or several |

```yaml
# dataset.yml
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
# models/calendar/day.table.yml
schema: calendar
name: day
title: Japanese calendar
primary_key: date
fields:
  - name: date
    title: Date
```

**Never write column types.** Types, column order and nullability are read from the data,
so they cannot drift from it. Writing them is rejected.

**Declare `primary_key`** with the column, or list of columns, that identifies one row.
`queria validate` reads the data and rejects repeated or NULL key values, so a failure there
means either the data has duplicates or the declared grain is wrong.

Tables sharing a column layout go in one file, with YAML anchors carrying the shared part:

```yaml
_fields: &fields
  - { name: local_government_code, title: Local government code, annotations: { queria: { semantic: { role: entity, name: city } } } }
  - { name: value, title: Value, annotations: { queria: { semantic: { role: measure, agg: sum } } } }
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
| `fdl commit` refuses inlined rows | DuckLake inlined rows into the catalog | Set `DATA_INLINING_ROW_LIMIT` to 0, as checkout's `ATTACH` does |
| `fdl push` is refused | Someone pushed after your checkout | `fdl checkout` again and rebuild |

The `.dbtignore` one is easy to miss. Check for them before declaring
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

- Do not hand-edit a version's `dataset.json`. `fdl commit` makes it from the declarations and the data.
- Do not make a dataset public without being asked to.
- `queria delete` is immediate. Confirm the name with the user before running it.

Full reference: https://docs.queria.io/publish

Done. Next: `queria-describe-dataset`, to write the descriptions that make the dataset
usable by someone who did not build it.
