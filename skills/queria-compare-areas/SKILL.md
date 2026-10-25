---
name: queria-compare-areas
description: Compare places in Japan with Queria data to answer a location decision — where to open a shop (出店・立地・商圏), which neighbourhood or town to live in or move to (住まい選び・移住), which station area or municipality suits a purpose (エリア比較・自治体比較). Use when the user is choosing where to put something or someone — a shop, an office, a home — and asks どこがいい / どのあたり / どのエリア / どっちがいい, even if they do not mention data or Queria. Not for sightseeing, travel or restaurant recommendations. Builds candidates from the data, measures each one the same way (population by age, daytime population, stations, competitors and complements, land prices) and states the limits of the data. General querying is the queria skill; writing the result up as a page is queria-write-page.
---

# Comparing areas with Queria

A question like 「つくば市でクレープ屋を出すならどのあたり？」 is a judgement, but the judgement
rests on facts Queria holds down to the 町丁目: who lives there and how old they are, how
many people are there in the daytime, how many use the nearest station, and what shops are
already there. Answer it from those numbers, not from web articles or memory. Use web search
only for what Queria cannot hold (rents, very recent openings, opening hours).

## Tools

Through the Queria MCP server:

- `query` runs one SELECT on the server and returns the rows to you. Use it to read numbers
  and decide the next SQL — this loop is where the analysis happens. Not every account has
  it; if it is not in your tool list, use `show_sql` and ask the user what they see, or the
  CLI.
- `show_sql` runs SQL in the user's browser and shows a table, chart or map to them. You do
  not see the result. Use it for the final comparison the user should look at, not to explore.
- `trade_area_sql` builds the SQL for a circular trade area around a point (population and
  households prorated by 町丁目, land prices, shops in Tokyo).
- `search`, `get_columns` to find and check tables. Check columns before writing SQL; the
  names below are starting points, not a schema.

Tables below are written as they go in SQL: dataset, schema and table, as in
`nlftp.railway.station_passenger` (dataset `nlftp`, schema `railway`, table
`station_passenger`). `get_columns` takes the dataset and the table on their own —
`get_columns(dataset="nlftp", table="station_passenger")`, not `railway.station_passenger`.
The schema is part of the name; do not replace it with `main`.

With a shell, `uvx queria sql "<query>"` does what `query` does.

## The method

1. Say what the decision depends on. Before any SQL, write two or three lines: who the
   customers or residents are, how they arrive (walk from a station, drive, live nearby), and
   which measures follow from that. A crepe shop needs foot traffic and young people or
   families; a family moving needs children, schools and land prices. If the user's purpose is
   unclear, state your assumption and go on — ask at the end, not first.

2. Build candidates from the data, in this order, and list them before measuring. Do not
   start from a list you remember. A station or mesh within 800m of a candidate already
   taken is the same place; skip it and take the next one.
   1. Stations inside the area: `nlftp.railway.station_passenger` (passengers per day by
      year), picked by location and summed by `station_group_code` (see Pitfalls). Rank
      them by the latest year each one has a count — some have none in 2024 — and take
      the top four. Use the centroid of the group's `geometry` as the point.
   2. Where shops cluster: count `overturemaps.places.place` by `mesh3` (1km mesh) for the
      places inside the area and add the two meshes with the most places. Use the centroid
      of the mesh in `e_stat.boundary.mesh_1km` as the point (`mesh_code` is VARCHAR, so
      cast `mesh3`).
   3. Places the user named.

   That gives five to eight candidates. If the area has fewer than four stations, add
   meshes until there are five. When customers come by car, take only the top two stations
   and add meshes until there are five. Give each a name and a point (lng, lat).

3. Measure every candidate the same way — the same radius and the same year. One table,
   one row per candidate. The radius is 800m (about a 10-minute walk) when customers or
   residents arrive on foot, and 3km when they come by car; use another only when the user
   asks for it, and say which one you used. `trade_area_sql` takes up to 3km.
   - Residents: `trade_area_sql` gives total population and households only. For the age
     mix, take its `small_areas` query (`key_code` and `share`, the part of each 町丁目 inside
     the circle) and join `e_stat.census.census_small_area_age` on `area = key_code`,
     multiplying `value` by `share`. Without the share, a large 字 that only touches the
     circle counts in full
   - Daytime population: `population_estimates.daytime.small_area` or
     `.daytime.mesh_1km` (estimates; see their descriptions)
   - Competitors and complements: `overturemaps.places.place` by `taxonomy_primary`
     (`dessert_shop`, `cafe`, `coffee_shop`, `bakery`…). `basic_category` is coarser —
     dessert shops sit under `casual_eatery` there. Chains are also in `alltheplaces.places.place`
   - Land prices: from `trade_area_sql`, or search 地価
   - Anything the purpose needs beyond these: `search` with `keys` (`small_area`, `mesh`,
     `city`) finds tables you can join on the same areas

4. Read the table and go one level deeper where it matters. If two candidates are close,
   look at what separates them (age mix, daytime vs night, competitors per 1,000 people). If
   a number looks wrong, check it before using it.

5. Show the comparison and give a verdict. Put the final table in front of the user with
   `show_sql` when the client can show it, then rank the candidates with one reason each.
   Name the trade-off of the top choice (e.g. most traffic but most competitors).

6. State the limits of the data in a few lines: the census is 2020, daytime population is
   an estimate built on 2016 employment figures, Overture misses small and new shops,
   coverage of a dataset is partial. Say which way each limit pushes the numbers.

7. Offer the next step. Compare the top two by walking time instead of a circle (a `gis`
   block in a page, Pro plan only), add land prices or rents, or write it up as a page
   (queria-write-page).

## Pitfalls

- 国勢調査の小地域 tables (`census_small_area_*`) mix levels in `area`: 5-digit municipalities,
  9-digit 町丁・字等 and their 11-digit 丁目. Summing a table on its own, filter by `area_level`
  first. Joined to `e_stat.boundary.small_area`, do not filter: the polygons are already one
  layer of 9 and 11 digits without overlap, and keeping only `small_area` drops the 丁目-split
  areas (about 64% of the population). Sex and age are both in `cat01` / `age_class`, and
  `cat01` holds the 総数, 5-year bands and the 15歳未満 / 15〜64 / 65歳以上 groups side by side;
  pick codes that do not overlap. The column named `sex` is the 秘匿・合算 flag, not 男女.
  秘匿 rows hold 0 — those people are counted in a neighbouring 合算 area, so a 0 there does
  not mean nobody lives there.
- Area codes differ by table: 5-digit standard area codes (e_stat) vs 6-digit 全国地方公共団体コード
  (`local_government_code`, lg_code). Convert through `lg_code` before joining. In 政令市,
  Overture's `local_government_code` is the city (横浜市 141003) while e_stat has only the wards
  (14101…), so the codes never match; count Overture places by distance or polygon instead.
- Station passengers: there is one row per line, and each operator carries its count on one
  of its lines (新宿 has four counts: JR, 京王, 小田急, 東京メトロ); the other rows are NULL.
  Sum by `station_group_code`; taking one non-NULL row drops the other operators. Never
  pick stations by name alone — names repeat across Japan (万博記念公園 is in Ibaraki and in
  Osaka). Narrow by location first.
- Do not present a count of zero competitors as "no competition"; it may be a gap in coverage.
