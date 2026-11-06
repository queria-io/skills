# Shortening a table description: a worked example

Queria's e-Stat dataset (`e_stat`) is where the rules in `SKILL.md` came from. Its 66 table
descriptions had a median of about 240 characters and some over 2,000: grain, source,
coverage, validation numbers and source URLs, all in one field. Lists showed all of it on
one line.

## Before

`building_starts.municipality_use`, 13 lines:

```
建築着工統計調査（建築物着工統計）の市区町村別・用途別（大分類）の着工建築物の数・床面積・工事費予定額（年次）。
建築基準法の建築工事届にもとづく全数の集計で、床面積の合計が10平方メートルを超える建築物が対象。
用途は18区分（居住専用住宅・製造業用・宿泊業，飲食サービス業用など）と、その計。
収録は2011年〜2024年の14年、505,153行。地域は市区町村（特別区を含む。1年に1,716〜1,735）と
政令指定都市の区（170〜175）。
市区町村の行を足すと全国の値に一致する（建築物の数・床面積とも全年・全用途で実測）。区は政令指定都市の
内訳なので、area_level で絞らずに合計すると政令指定都市の分を二重に数える。
着工が1棟も無かった年の地域は行が無い。0 の行は持たない。
工事費予定額は2019年までしか無い（2020年以降の市区町村別の表に項目が無い）。
2019年までも秘匿（原典の「＊」）が用途の内訳の23%にあり、その値は NULL。
地域コードは各年の時点のもので、合併で消えた市町村と浜松市の行政区再編前の区（2023年まで）は
現行の code.municipality に無い。2024年の地域コードはすべて code.municipality に載る。
出典: 国土交通省 建築着工統計調査 建築物着工統計。
```

## Sentence by sentence

| The sentence | Moved to |
| --- | --- |
| What the table holds, by municipality and use, annual | the summary sentence |
| Ward rows are a breakdown of designated cities; summing without `area_level` double-counts | caveat bullet 1 |
| Estimated cost only up to 2019; suppressed values are NULL | caveat bullet 2 |
| Area codes are as of each year; merged codes are missing from `code.municipality` | caveat bullet 3 |
| Survey scope (building notices over 10 m²), 18 use categories, row counts | the table's section in `docs/building-starts.md` |
| "Municipality rows sum to the national value (measured for all years)" | a validation number: the docs page, not the description |
| Source and URLs | the README's 「出典と利用条件」 and `sources` in `dataset.yml` |

## After

```yaml
description: |-
  建築着工統計調査の市区町村別・用途別の着工建築物の数・床面積・工事費予定額を、2011年から年次で持つテーブルです。

  - 市区町村と政令指定都市の区の行があるので、area_level で絞らずに合計すると二重に数えます
  - 工事費予定額は2019年までで、秘匿の値は NULL です
  - 地域コードは各年の時点のもので、合併で消えたコードは code.municipality にありません
```

The table list now shows only the first line. A reader who picks the table reads the three
caveats on the table page, and the README or the docs page for the rest.

## The README

With 66 tables, per-table sections would make the README too long. `e_stat`'s README is a
table of contents instead: 出典と利用条件, a list of schemas linking to `docs/<schema>.md`,
one SQL example under 使い方, three caveats that span schemas under 注意, and 更新. The
one-time import procedure for `local_tables.yml` moved to `MAINTAINING.md`.

## The tell

If a table description needs more than three bullets, either the details belong in the
README, or the table should be split so the caveat is no longer needed.
