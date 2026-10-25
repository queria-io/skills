---
name: queria-write-page
description: Write up findings from Queria data as a page on queria.io — Markdown with SQL, Plotly charts and trade-area maps that run in the reader's browser — and create, update or delete it with queria page or the MCP page tools. Covers the front matter, declaring datasets, sql/plot/gis code blocks, visibility, and what queria.io refuses. Use when the user wants a report, article, dashboard-like write-up or ページ/記事/レポート from Queria data. Querying the data itself is the queria skill.
---

# Writing a page on Queria

A page is a Markdown document with SQL and chart code blocks, served at
`https://queria.io/pages/<handle>/<slug>`. The SQL runs in the reader's browser against the
datasets the page declares, so every number on the page can be checked by re-running it.
It is the same format as Queria's showcase articles (`https://queria.io/pages/queria/<slug>`).

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
| ` ```gis` | a map of your query results, or a trade-area map around a point (see below) |

`plot` takes a Plotly spec as JSON (`data` and `layout`), not JavaScript. Every `$<id>` must
be a `sql id=` in the page. Each SQL block is one `SELECT`.

### Trade-area maps

When the page is about places (a station, a shopping street, facilities, candidate sites),
add a map around a representative point. It shows the population, households and age mix
inside the range, plus land prices and facilities, and the reader can click to move the
point and recount, open it in the trade-area tool, or open the SQL behind the numbers.

```gis
type: around
lng: 139.393896
lat: 35.679414
radius: 1000
caption: 日野駅から半径1km
```

- The body is YAML with `type` (`around`), `lng` / `lat` (numbers, not strings), the range
  (below), and optional `show` and `caption`. Nothing goes on the fence line.
- The range is `radius` in whole metres, 100–3000.
- The point must be in Japan: `lng` 122–154, `lat` 20–46. Anything outside is refused.
- `caption` must be a string. Quote it when YAML would read it as something else or it
  contains `: ` (`caption: "2024"`, `caption: "駅前: 半径1km"`).
- To measure by travel time instead, write one of `walk` (5, 10, 15 or 20 minutes), `bike`
  (5, 10 or 15) or `car` (10, 15, 20 or 30). Travel time needs a Pro or organisation owner:
  it is checked against the page owner's plan when the page is saved, and anyone can read the
  saved result. `radius` can be written alongside it (1000 when omitted); readers who cannot
  count travel time themselves get that circle when they move the point, while Pro readers
  get the travel-time range again.
- `show` picks which sections to show: `age`, `households`, `housing`, `land_price`,
  `facilities` (a list, or one string). Total population, households and the list of 町丁目 are
  always shown. Choosing sections also needs a Pro or organisation owner.
- The map's own tables do not go in `datasets`; declare only what your `sql` blocks use.
- Take the coordinates from the data with SQL rather than guessing them, and write them to
  about six decimal places.
- The map's numbers are saved with the page like a chart's result, and recounted in the
  reader's browser when they move the point. Don't copy them into the text; if the text needs
  a number, show it with a `sql` block too, or point the reader at the map instead.
- The map computes in the reader's browser, so put at most three on a page. Put the
  overview first as a `plot`, then zoom into places with maps.

### Maps of query results

To put rows from a `sql id=` block on a map, use `type: map` rather than writing a
`choroplethmap` by hand. Place each layer by an area code column or by longitude and
latitude columns; the boundaries, colours, legend and the view that fits the data are built
for you.

```gis
type: map
caption: 市区町村別の人口
layers:
  - from: pop           # a sql id=
    area: city          # prefecture / city / small_area
    code: lg_code       # area code column: 2, 5 (or 6), or 9/11 digits
    color: population   # numeric column to colour by (optional)
    label: city_name    # shown on hover (optional)
    name: 人口           # legend entry (optional)
  - from: stations
    lng: lon
    lat: lat
```

- Use either `area` + `code` or `lng` + `lat` per layer. Later layers are drawn on top.
- The boundary datasets do not go in `datasets`.
- `lng` / `lat` / `zoom` on the block are optional; leave them out to fit the data.
- The map's data is stored like a chart's, so it shows the saved result. The page's preview
  image comes from the first `plot`, so still lead with a chart. `sql id=` names must not
  start with `__`.
- To count around a point on the same map, add `around:` with `lng`, `lat` and a range
  (`radius`, or `walk` / `bike` / `car`; the trade-area fields above). The map is then framed on the point, so don't set
  `lng` / `lat` / `zoom` on the map itself. `type: around` is a shorthand for this.

Keep the result sizes small — results are saved with the page and sent to every reader, and the
reader's browser runs the queries itself when the saved result does not match.
Aggregate in SQL rather than sending thousands of rows to a chart.

## Limits

- No images yet (including `hero`).
- Up to 20 datasets per page, 200KB of Markdown, tables up to 100 columns.
- Up to 500 pages per account.
- The slug is lowercase letters, digits and hyphens, and cannot be changed after creation.

When queria.io refuses a page, the error says which rule it broke. Fix the Markdown and
run `queria page edit` (or `create` if the first create failed) again.
