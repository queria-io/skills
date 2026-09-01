# Queria SQL Recipes

Run with `uvx queria sql "<query>"`.
Reference tables as `dataset.schema.table`. Referencing another dataset attaches it
automatically, so you can join across datasets directly. Prefer `mart_`-prefixed tables
for analysis-ready data.

Data values (place names, item names, categories) are in Japanese — filter and search
with Japanese strings as shown below.

## DuckDB dialect

Queries run on DuckDB. Most PostgreSQL habits carry over, but not all — `to_char`
does not exist here, for example; use `strftime(d, '%Y-%m')`.

Forms DuckDB has and PostgreSQL does not. Run them one at a time — `queria sql` takes
a single statement per call.

```sql
SELECT prefecture, count(*) AS c FROM lg_code.main.mart_lg_code GROUP BY ALL;

SELECT * EXCLUDE (geometry) FROM nlftp.boundary.prefecture;

SELECT arg_max(area_name, value) AS most_populous   -- one column's value at another column's max
FROM e_stat.ssds.a_pref_population
WHERE item_name = 'A1101_総人口' AND area_name <> '全国';

SELECT 1 AS a UNION BY NAME SELECT 2 AS b;   -- union on column names rather than position
```

`QUALIFY` filters window results without wrapping the query in a subquery; the e-Stat
recipes below use it to drop breakdown rows.

`PIVOT` turns a long table wide without spelling out a column per value, which suits
the e-Stat tables below — they carry one row per (indicator, area, year). Total
population by prefecture, one column per year:

```sql
PIVOT (SELECT area_name, year, value FROM e_stat.ssds.a_pref_population
       WHERE item_name = 'A1101_総人口' AND area_name <> '全国' AND year >= 2021)
ON year USING max(value)
ORDER BY area_name
```

## Discovery and schema inspection

```bash
uvx queria list                     # list datasets
uvx queria search 不動産             # search datasets/tables/columns ("real estate")
uvx queria info reinfolib           # metadata (license, source, schema)
uvx queria schema e_stat            # list tables
uvx queria columns reinfolib        # columns (types, descriptions)
uvx queria summarize zipcode.main.mart_zipcode   # column stats (full scan)
```

Inspect columns of any table directly:
```sql
SELECT column_name, column_type FROM (DESCRIBE e_stat.ssds.a_pref_population)
```

## Postal codes x municipality codes

```sql
SELECT g.prefecture, g.city, COUNT(*) AS zip_count
FROM zipcode.main.mart_zipcode z
JOIN lg_code.main.mart_lg_code g ON z.lg_code = g.lg_code
GROUP BY 1, 2 ORDER BY zip_count DESC
```

## Corporations: gBizINFO x NTA corporate numbers (join on corporate_number)

Top companies by capital stock:
```sql
SELECT h.name, h.prefecture_name, c.capital_stock, c.employee_number
FROM gbizinfo.main.mart_gbizinfo_company c
JOIN houjin_bangou.main.mart_houjin_bangou h
  ON c.corporate_number = h.corporate_number
WHERE c.capital_stock IS NOT NULL
ORDER BY c.capital_stock DESC LIMIT 20
```

Subsidy recipients (enrich location from houjin_bangou):
```sql
SELECT h.prefecture_name, COUNT(*) AS subsidies
FROM gbizinfo.main.mart_gbizinfo_subsidy s
JOIN houjin_bangou.main.mart_houjin_bangou h
  ON s.corporate_number = h.corporate_number
GROUP BY 1 ORDER BY subsidies DESC
```

## Statistics: e-Stat SSDS (System of Social and Demographic Statistics)

SSDS tables are long-format (`item_name`, `area_name`, `year`, `value`). Filter
indicators by `item_name`. The same (area, year) may contain multiple `cat01` values
or time points, so also filter by `cat01` when needed.

Find available indicator names (総人口 = total population):
```sql
SELECT DISTINCT item_name FROM e_stat.ssds.a_pref_population WHERE item_name LIKE '%総人口%'
```

Total population by prefecture (latest year, excluding 全国 = nationwide):
```sql
SELECT area_name, value
FROM e_stat.ssds.a_pref_population
WHERE item_name = 'A1101_総人口'
  AND year = (SELECT MAX(year) FROM e_stat.ssds.a_pref_population
              WHERE item_name = 'A1101_総人口')
  AND area_name <> '全国'
QUALIFY ROW_NUMBER() OVER (PARTITION BY area_name ORDER BY value DESC) = 1
ORDER BY value DESC
```

Search across statistical tables (what tables exist; 人口 = population):
```sql
SELECT statistics_name, table_title, collect_area
FROM e_stat.main.stats_catalog
WHERE lower(statistics_name || ' ' || table_title) LIKE '%人口%' LIMIT 20
```

## Statistics x municipality codes (municipality-level joins)

SSDS `_municipal_` tables have `area` (5-digit municipality code), which joins to
`lg_code_5` in lg_code. The latest year differs per indicator, so scope the year
subquery to the same `item_name`:
```sql
SELECT g.prefecture, g.city, p.value AS population
FROM e_stat.ssds.a_municipal_population p
JOIN lg_code.main.mart_lg_code g ON p.area = g.lg_code_5
WHERE p.item_name = 'A1101_総人口'
  AND p.year = (SELECT MAX(year) FROM e_stat.ssds.a_municipal_population
                WHERE item_name = 'A1101_総人口')
QUALIFY ROW_NUMBER() OVER (PARTITION BY p.area ORDER BY p.value DESC) = 1  -- dedupe breakdown rows
ORDER BY population DESC LIMIT 20
```
(Match code digit counts on both sides with `columns` before joining. e_stat `area` is
5 digits; lg_code has the 6-digit `lg_code` and the 5-digit `lg_code_5`.)

## Geography: national land numerical information (GIS)

Boundary polygons have a `geometry` column and the spatial extension is preloaded, so
`ST_*` functions work. Coordinates are longitude/latitude in degrees (`ST_X` is the
longitude), which means plain `ST_Area` and `ST_Distance` return degrees — usable for
relative comparison, meaningless as an area or a distance.

Real-world units come from the `_Spheroid` functions, and those read latitude first.
`queria sql` accepts read-only statements only, so `SET geometry_always_xy = true` is
not available; flip the axes per call with `ST_FlipCoordinates` instead. Skipping the
flip yields `nan` rather than an error, and `nan` sorts to the top under
`ORDER BY ... DESC`.

Prefecture areas in km² (北海道 comes out at 83,417.9, slightly under the 国土地理院
面積調):
```sql
SELECT prefecture_name, round(ST_Area_Spheroid(ST_FlipCoordinates(geometry)) / 1e6, 1) AS km2
FROM nlftp.boundary.prefecture ORDER BY km2 DESC LIMIT 10
```

Distance between two stations in metres:
```sql
WITH t AS (SELECT geometry AS g FROM nlftp.railway.station
           WHERE station_name = '東京' AND operator = '東日本旅客鉄道'
           ORDER BY station_code LIMIT 1),
     s AS (SELECT geometry AS g FROM nlftp.railway.station
           WHERE station_name = '新宿' AND operator = '東日本旅客鉄道'
           ORDER BY station_code LIMIT 1)
SELECT ST_Distance_Spheroid(ST_FlipCoordinates(t.g)::POINT_2D,
                            ST_FlipCoordinates(s.g)::POINT_2D) AS metres
FROM t, s
```
(A station name matches one row per line served, so pick a row deterministically —
without the `ORDER BY` the distance changes between runs. Check column names with
`uvx queria columns nlftp`.)

## Real estate

```sql
SELECT prefecture, COUNT(*) AS deals, AVG(trade_price) AS avg_price
FROM reinfolib.main.mart_trade_prices
GROUP BY 1 ORDER BY deals DESC
```
(Check actual columns with `uvx queria columns reinfolib`.)

## Handing off to other skills (visualization / analysis / dashboards)

This skill does not visualize. Write results out and pass them along:
```bash
uvx queria sql "
  SELECT area_name, value FROM e_stat.ssds.a_pref_population
  WHERE item_name='A1101_総人口' AND year=2024 AND area_name<>'全国'
" --out /tmp/pref_population.parquet
```
Pass the written CSV/Parquet to data:create-viz (charts) / data:analyze (analysis) /
data:build-dashboard or Tableau/PowerBI MCP (dashboards).
