---
name: queria
description: Explore and query Queria's public Japanese open data (data.queria.io) with read-only DuckDB SQL, through the Queria MCP connector (search_catalog, list_columns, query, show_chart) or the queria CLI. Covers population down to 町丁目 and mesh, the numbers around a station or address (駅周辺・半径・商圏の人口や世帯), postal codes (郵便番号), corporate numbers (法人番号), e-Stat government statistics (政府統計), national land numerical information (国土数値情報), shops and facilities, land prices, calendars, municipality codes, and more, with cross-dataset joins. Use when asked to find or use data via Queria, look up numbers for a place in Japan, explore Japanese open data (日本のオープンデータ), or join datasets across sources. Also covers reporting missing datasets, data defects, and documentation errors back to Queria.
---

# Queria Open Data Exploration

Queria publishes Japanese open data as DuckLake catalogs (data.queria.io). This skill
finds datasets, inspects schemas, retrieves data with SQL and joins across datasets.
There is no static list of datasets — always discover them from the catalog.

Most data values (place names, statistics categories, company names) are in Japanese.
Search with Japanese keywords when possible; table and column names are in English.

## Pick the route

Use the first one that works.

1. **A shell** (Claude Code, Codex and other coding agents): run the queria CLI with
   `uvx queria …` (or `pip install queria`). It needs no sign-in and does not use the MCP
   allowance. If the command cannot run or cannot reach data.queria.io (a sandbox without
   network, as in some chat apps), go to 2. If a subcommand is missing (`docs`, for
   example), the uvx cache is stale: run `uvx queria@latest …`. Anonymous access is
   rate-limited; when the CLI says the limit is reached, set up a token
   (https://docs.queria.io/connection/authentication) or go to 2.
2. **Queria MCP tools** (`search_catalog`, `query`, …) in your tool list: use them. This is
   the route in claude.ai, the Claude apps and Cowork.
3. **Neither**: tell the user to add the connector `https://mcp.queria.io`
   (https://docs.queria.io/ai-chat) and stop. Do not answer from memory or web articles
   when the question is about the numbers Queria holds.

One exception to the order: the trade area functions below exist only in the MCP server.
For numbers around a point, use `query` when the MCP tools are there, even with a shell.
`query` is not offered to every account (some organisation tokens lack it); without it,
call the trade area functions in the SQL of `show_chart`.

| Step | MCP tool | CLI |
|---|---|---|
| Find datasets, tables, columns | `search_catalog` (`keyword`, `keys`, `area`, `theme`, `year`), `list_datasets` | `uvx queria search <keyword>`, `uvx queria list` |
| Licence and source | `get_dataset` | `uvx queria info <dataset>` |
| Tables and columns | `list_tables`, `list_columns(dataset, table)` | `uvx queria schema <dataset>`, `uvx queria columns <dataset> [table]` |
| Run SQL, read the rows | `query(sql, datasets)` | `uvx queria sql "<query>"` |
| Show the result to the user | `show_chart` (table, chart or map in the conversation) | `--out result.csv` and hand the file on |
| Docs | `list_docs`, `read_doc` | `uvx queria docs list`, `uvx queria docs show <name>` |
| Report a gap | `submit_feedback` | `uvx queria feedback send` |

`list_columns` takes the dataset and the table separately:
`list_columns(dataset="nlftp", table="station_passenger")`, not `railway.station_passenger`.

## Numbers around a point: use the trade area functions

For anything inside a radius of a point — 「つくば駅から 1km 以内の人口」, households near an
address, shops or land prices around a station — call these functions inside the SQL of
`query` or `show_chart`. **Do not compute overlaps of meshes or small areas with a circle
yourself**: it takes minutes and the numbers will not match Queria's.

| Function `(lng, lat, radius_m)` | Returns |
|---|---|
| `trade_area_population` | one row: `population`, `households`, `small_areas` |
| `trade_area_small_areas` | each 町丁目 the circle touches: `key_code`, `city_name`, `area_name`, `share`, `population`, `households` (already multiplied by `share`) |
| `trade_area_facilities` | shops and facilities: `name`, `industry_name`, `longitude`, `latitude`, `address`… |
| `trade_area_land_prices` | land price points: `current_price` (yen/m²), `year`, `use_category`… |

The radius is 100–3000 m. The tables they read are added for you; list only the datasets
your own SQL reads in `datasets`. Take the point from the data in the same SQL, not from
memory. A station name repeats across Japan and across lines, so narrow it by operator or
area:

```sql
WITH s AS (
  SELECT station_name, operator,
         ST_X(ST_Centroid(geometry)) AS lng, ST_Y(ST_Centroid(geometry)) AS lat
  FROM nlftp.railway.station
  WHERE station_name = 'つくば'
)
SELECT s.*, p.*
FROM s, LATERAL trade_area_population(s.lng, s.lat, 1000) p
```

(`query(sql, datasets=["nlftp"])`.) For the age mix inside the circle, join
`trade_area_small_areas` to `e_stat.census.census_small_area_age` on `area = key_code` and
multiply by `share` (the queria-compare-areas skill has the details).

These functions exist only in the MCP server. With only the CLI, say that the exact radius
numbers need the connector or https://queria.io/maps, rather than rebuilding the circle
overlaps by hand. For a location comparison, queria-compare-areas describes a rougher
count by small-area point that it labels as approximate.

## Exploration workflow

1. Discover: `search_catalog(keyword="人口 小地域")` or `uvx queria search 人口`. A place name
   alone finds that city's small datasets; nationwide tables hold places as area codes,
   so search by topic and narrow by code in SQL
2. Check provenance: licence and source URL (`get_dataset` / `info`)
3. Understand the schema: tables, then the columns of the table you will use
4. Look at values: one SQL with `LIMIT 20` that checks everything you are unsure of
5. Retrieve and check the numbers
6. Answer, then show the result the user asked for

With the CLI, show the result as a table in your answer, or write it with `--out` for
another tool. With MCP, `query` is counted (the free plan allows 50 a week), so put related checks into
one SQL and learn tables from `list_tables` / `list_columns` rather than from `query`. If
it says the allowance is used up, stop exploring and answer from what you have. Write the
answer in full (verdict, reasons, limits of the data) and call `show_chart` once at the end
for the table, chart or map; do not run the same final SQL with `query` first. For numbers
by area, give `show_chart` a `map` (`read_doc('charts')`).

### Writing SQL

The engine is DuckDB, so write DuckDB SQL rather than PostgreSQL-shaped SQL.
Dialect reference: https://duckdb.org/docs/stable/sql/introduction

Reference tables as `dataset.schema.table` (e.g. `zipcode.main.mart_zipcode`). The schema
is part of the name; do not replace it with `main`. Referencing several datasets attaches
them all, so you can join across them (with `query`, list each one in `datasets`).
Analysis-ready tables use the `mart_` prefix (`raw_` is raw data, `stg_` is intermediate).

```sql
SELECT g.prefecture, g.city, COUNT(*) AS zip_count
FROM zipcode.main.mart_zipcode z
JOIN lg_code.main.mart_lg_code g ON z.lg_code = g.lg_code
GROUP BY 1, 2 ORDER BY zip_count DESC
```

### Common join keys

Always verify digit counts and column names with `list_columns` / `columns`. In
`list_columns`, a column description ending in `[key: city]` marks a key; `search_catalog(keys=["city"])` lists the
tables you can join on it.

- `lg_code`: national local government code (全国地方公共団体コード). `lg_code.main.mart_lg_code`
  is the join hub, holding the 6-digit `lg_code` and the 5-digit `lg_code_5`. The 5-digit
  `area` column in e-Stat municipality-level tables joins to `lg_code_5`
- `corporate_number`: corporate number (法人番号). Joins corporate datasets to each other
- `key_code`: joins boundary polygons (e.g. census small areas) to statistics

See `references/sql-recipes.md` for common queries.

## Reporting gaps and problems

When exploration turns up something the maintainers should know — a dataset that does
not exist, a column that looks wrong or stale, a documentation page that misled you —
report it with `submit_feedback` or `uvx queria feedback send`. Do not silently work
around a gap and move on; that information is only in your hands right now.

- **Show the user the exact text and get their explicit agreement before sending.**
  Feedback is sent in their name.
- **Never include query results, credentials, or personal data** in the message.

Send one consolidated report near the end of the session, and do not nag — if the user is
not interested, drop it. See `references/feedback.md`.

## Related skills

- Choosing where to open a shop or where to live: queria-compare-areas
- Writing the result up as a page on queria.io: queria-write-page
- Without MCP, write results with `--out result.parquet` and hand them to a charting or
  analysis skill

## Constraints

- Read-only. Public datasets, plus private ones the signed-in account can read (MCP).
- Unfiltered SELECTs and `SUMMARIZE` scan all remote data. Probe with `LIMIT` first.
- Never ATTACH DuckLake directly from the DuckDB CLI (version mismatch corrupts the
  catalog). Always go through the MCP server or the queria CLI.
