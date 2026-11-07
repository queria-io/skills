# Queria Skills

[English](README.md)

Queria が公開する日本のオープンデータ（[data.queria.io](https://queria.io)）を探索するための [Agent Skills](https://agentskills.io/) コレクション。データセットの発見・スキーマ把握・read-only SQL・横断結合を、対応エージェントから自然言語で行える。

郵便番号、国税庁法人番号、gBizINFO、e-Stat 政府統計、不動産価格、国土数値情報（GIS）、自治体コード、暦などを、別々のデータセットを跨いで JOIN しながら探索できる。利用できるデータセットの一覧は常にカタログから動的に取得する。

## インストール

Claude（claude.ai、デスクトップ・モバイルのアプリ、Cowork）では Queria の MCP コネクタ経由で、Agent Skills 標準に対応したエージェント（Claude Code、OpenAI Codex、Cursor、OpenCode など）ではコネクタか queria CLI 経由で使える。

### Claude（claude.ai・デスクトップ・Cowork）

「Customize」>「Plugins」>「Add」>「Add marketplace」で `queria-io/skills` を入力し、`queria` プラグインを追加する。続けてプラグインの「Connectors」タブで Queria（`https://mcp.queria.io`）を接続すると、Queria のログイン画面が開く。スキルはコネクタのツールを使うので、質問する前に接続しておく。

コネクタだけでも Claude は Queria のデータを読める。プラグインを入れると、テーブルの探し方、駅周辺の集計、場所の比較、ページの書き方のスキルが加わる。

### Queria CLI

エージェントを問わず、[queria CLI](https://docs.queria.io/) からインストールする:

```
uvx queria skills install
```

書き込み先はいまのプロジェクト。Codex・Cursor・OpenCode などが読む `.agents/skills/` に書き、Claude Code が設定されている環境では `.claude/skills/` にも書く。このマシンの全プロジェクトで使うなら `--global` を付ける。オプションは [CLI リファレンス](https://docs.queria.io/reference/cli#queria-skills)を参照。

### Claude Code

[プラグインマーケットプレイス](https://code.claude.com/docs/en/discover-plugins)からインストールする:

```
/plugin marketplace add queria-io/skills
/plugin install queria@queria-io
```

### OpenAI Codex

Codex CLI の[プラグイン](https://developers.openai.com/codex/plugins)としてインストールする:

```
codex plugin marketplace add queria-io/skills
codex plugin add queria@queria-io
```

### npx skills

[`npx skills`](https://skills.sh/) CLI で、その他の Agent Skills 対応エージェントにインストールする:

```
npx skills add queria-io/skills
```

### Clone / Copy

このリポジトリを clone して、スキルフォルダをエージェントのスキルディレクトリにコピーしてもよい:

| Agent | Skill Directory | Docs |
| --- | --- | --- |
| Claude Code | `~/.claude/skills/` | [docs](https://code.claude.com/docs/en/skills) |
| Cursor | `~/.cursor/skills/` | [docs](https://cursor.com/docs/context/skills) |
| OpenCode | `~/.config/opencode/skills/` | [docs](https://opencode.ai/docs/skills/) |
| OpenAI Codex | `~/.agents/skills/` | [docs](https://developers.openai.com/codex/skills/) |
| Pi | `~/.pi/agent/skills/` | [docs](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent#skills) |

## Skills

| Skill | Useful for |
| --- | --- |
| queria | 日本のオープンデータの発見・スキーマ把握・read-only SQL・横断結合・CSV/Parquet 書き出し・不足や不備の報告 |
| queria-publish-dataset | 自分のデータセットの公開: 登録・宣言・ビルド・push |
| queria-describe-dataset | データセット・テーブル・列の短い説明と README への書き分け、schema.org と GIF の語彙に沿った列名の付け方 |
| queria-write-page | SQL とグラフでレポートを書き、`create_report` か `queria report create` で保存する |
| queria-compare-areas | 出店・住まい選びなど場所の判断を、人口・昼間人口・駅・競合・地価で候補地を比べて答える |

スキルの本体は [skills/queria/SKILL.md](skills/queria/SKILL.md)、定番クエリは [references/sql-recipes.md](skills/queria/references/sql-recipes.md)、問題の報告手順は [references/feedback.md](skills/queria/references/feedback.md) を参照。

探索中に「このデータがない」「この列がおかしい」「ドキュメントが違う」と気づいたら、`submit_feedback` か `uvx queria feedback send` で Queria に報告できる。送信前に必ず本文の全文が表示され、同意なしに送られることはない。エージェントが代わりに送るには、事前の明示的な許可が必要。

コネクタ経由では、結果を `show_chart` で表・グラフ・地図にして見せる。CLI では結果を CSV/Parquet に書き出して別のツールに渡す。

## 必要なもの

次のどちらか。

- [MCP コネクタ](https://docs.queria.io/mcp) `https://mcp.queria.io`（プラグインに `.mcp.json` として入っている）。Queria の無料アカウントが要る。無料プランの query は週 50 回まで。地点の周りの集計（商圏の関数）はこちらでだけ使える
- [queria CLI](https://docs.queria.io/)（PyPI: `queria`）。シェルを使えるエージェント向け。uv があれば `uvx queria` でインストール不要、なければ `pip install queria`（Python 3.10+）。認証は不要（匿名アクセスには[レートリミット](https://docs.queria.io/connection/authentication)がある）

claude.ai のコード実行でも、「外部ネットワークへの通信を許可」をオンにして、許可リストに `pypi.org`・`files.pythonhosted.org`・`extensions.duckdb.org`・`queria.io`・`data.queria.io` を入れれば（またはすべてのドメイン）CLI が使える。

両方あるときは、スキルは CLI を使う。地点の周りの集計だけはコネクタを使う。queria-compare-areas はどの段でも地点の周りを数えるので、コネクタがあれば最初から最後までコネクタを使う。

## 送られるデータ

スキルは指示の文書だけで、スキル自体は何も送らない。エージェントが書いた SQL と、利用者が承認したフィードバックが、コネクタ（mcp.queria.io）か CLI（queria.io）を通して Queria に送られる。コネクタ経由では結果の行がエージェントに返り、そのエージェントの提供元にも渡る。詳しくは [MCP サーバー](https://docs.queria.io/mcp)を参照。

## 仕組み

Queria は各データセットを DuckLake カタログとして公開している。queria CLI が Web 版と同じく DuckLake を read-only で ATTACH してクエリする。データセット一覧と全スキーマは `catalog` データセットの `mart_*` テーブルに統合されており、`list` / `search` / `info` はここから発見する。カタログ format と CLI の互換性は CLI 側が管理し、更新が必要な場合はエラーメッセージで案内される。

## License

MIT
