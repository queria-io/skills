---
name: queria-write-page
description: Write up findings from Queria data as a page on queria.io — Markdown with SQL and Plotly charts that run in the reader's browser — and create, update or delete it with queria page or the MCP page tools. Covers the front matter, declaring datasets, sql/plot code blocks, visibility, and what queria.io refuses. Use when the user wants a report, article, dashboard-like write-up or ページ/記事/レポート from Queria data. Querying the data itself is the queria skill.
---

# Writing a page on Queria

A page is a Markdown document with SQL and chart code blocks, served at
`https://queria.io/pages/<handle>/<slug>`. The SQL runs in the reader's browser against the
datasets the page declares, so every number on the page can be checked by re-running it.
It is the same format as Queria's showcase articles.

Use the `queria` skill to explore the data and get the SQL right first. This skill is
about turning that into a page.

## Before you start

- `uvx queria login` — creating pages needs an account and a token that can write.
- Or, through the MCP server: `create_page`, `update_page`, `get_page` (and `show_doc("pages")`).

## The loop

```bash
uvx queria page create report.md              # new page, private, URL printed
uvx queria page view alice/report -o report.md  # fetch the current Markdown
uvx queria page edit alice/report report.md   # replace the Markdown
uvx queria page edit alice/report --public    # publish (only when the user asks)
uvx queria page delete alice/report --yes
```

The slug defaults to the file name (`report/index.md` → `report`). `--owner <handle>`
creates the page in an organisation the user belongs to. New pages are **private**.
**Never make a page public unless the user asked for it** — public pages are visible to
anyone and indexed by search engines. `--members` shows it to the owning account's members.

After creating or editing, give the user the URL so they can check it in the browser.

## The document

```markdown
---
title: 都道府県別の人口
description: 都道府県ごとの人口を並べます
publishedAt: 2026-10-23
author:
  name: 山田 太郎
tags: [人口]
datasets:
  - queria/e_stat as e_stat
---

## 人口の多い都道府県

```sql id=pref_pop
SELECT pref_name, population FROM e_stat.<schema>.<table> ORDER BY population DESC LIMIT 10
```

```plot caption="人口（上位10都道府県）"
{"data": [{"type": "bar", "x": "$pref_pop.pref_name", "y": "$pref_pop.population"}]}
```
```

Front matter keys `title`, `description`, `publishedAt` (`YYYY-MM-DD`), `author.name`,
`tags` and `datasets` are required. Unknown keys are rejected — check spelling. The page's
author is shown as the account that owns it, whatever `author` says.

Headings start at `##` (`title` is the page heading). Raw HTML is dropped.

### Declaring datasets

`datasets` lists every dataset the SQL touches, as `<handle>/<name> as <alias>`, and the
SQL uses the alias: `e_stat.<schema>.<table>`. Always write the handle — short names can
become ambiguous later and the page would break in the reader's browser.

queria.io refuses the page when:

- the SQL references a database that is not in `datasets`
- a dataset is in `datasets` but no SQL uses it
- two datasets share an alias
- a public page uses a dataset that is not public

### Code blocks

| Block | What it does |
| --- | --- |
| ` ```sql id=<name>` | a named query (a view), not shown; charts and later SQL refer to it |
| ` ```sql id=<name> display` | the same, and also shows the result as a table |
| ` ```sql caption="..."` | runs the query and shows the result as a table |
| ` ```sql run=false` | shows SQL without running it |
| ` ```plot caption="..."` | a Plotly chart; the JSON spec refers to data as `$<id>.<column>` |

`plot` takes a Plotly spec as JSON (`data` and `layout`), not JavaScript. Every `$<id>` must
be a `sql id=` in the page. Each SQL block is one `SELECT`.

Keep the result sizes small — the reader's browser runs every query when the page opens.
Aggregate in SQL rather than sending thousands of rows to a chart.

## Limits

- No images yet (including `hero`).
- Up to 20 datasets per page, 200KB of Markdown, tables up to 100 columns.
- Up to 500 pages per account.
- The slug is lowercase letters, digits and hyphens, and cannot be changed after creation.

When queria.io refuses a page, the error says which rule it broke. Fix the Markdown and
run `queria page edit` (or `create` if the first create failed) again.
