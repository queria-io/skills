# Queria Skills

[日本語](README.ja.md)

A collection of [Agent Skills](https://agentskills.io/) for exploring Queria's public Japanese open data ([data.queria.io](https://queria.io)). Discover datasets, inspect schemas, run read-only SQL, and join across datasets — all in natural language from any supported agent.

Explore postal codes, NTA corporate numbers, gBizINFO, e-Stat government statistics, real estate prices, national land numerical information (GIS), municipality codes, calendars, and more, joining across independent datasets. The list of available datasets is always discovered dynamically from the catalog.

## Installation

Works in Claude (claude.ai, the desktop and mobile apps, Cowork) through the Queria MCP connector, and in agents that support the Agent Skills standard (Claude Code, OpenAI Codex, Cursor, OpenCode, and others) through the connector or the queria CLI.

### Claude (claude.ai, desktop, Cowork)

In **Customize > Plugins > Add > Add marketplace**, enter `queria-io/skills` and add the `queria` plugin. Then open the plugin's **Connectors** tab and connect Queria (`https://queria.io/mcp`), which opens Queria's sign-in. The skills use the connector's tools, so connect it before you ask.

The connector alone already lets Claude read Queria's data. The plugin adds the skills: how to find tables, count around a station, compare places and write a page.

### Queria CLI

Install with the [queria CLI](https://docs.queria.io/en), whichever agent you use:

```
uvx queria skills install
```

The files go into the project you are in: `.agents/skills/`, which Codex, Cursor, OpenCode and others read, plus `.claude/skills/` wherever Claude Code is set up. Add `--global` to install them for every project on this machine. See the [CLI reference](https://docs.queria.io/en/reference/cli#queria-skills) for the options.

### Claude Code

Install from the [plugin marketplace](https://code.claude.com/docs/en/discover-plugins):

```
/plugin marketplace add queria-io/skills
/plugin install queria@queria-io
```

### OpenAI Codex

Install as a Codex CLI [plugin](https://developers.openai.com/codex/plugins):

```
codex plugin marketplace add queria-io/skills
codex plugin add queria@queria-io
```

### npx skills

Install into other Agent Skills-compatible agents with the [`npx skills`](https://skills.sh/) CLI:

```
npx skills add queria-io/skills
```

### Clone / Copy

You can also clone this repository and copy the skill folder into your agent's skills directory:

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
| queria | Discovering Japanese open data, inspecting schemas, read-only SQL, cross-dataset joins, CSV/Parquet export, reporting gaps back to Queria |
| queria-publish-dataset | Publishing a dataset of your own: registering it, declaring it, building, pushing |
| queria-describe-dataset | Writing short dataset, table and column descriptions with the details in the README, and naming columns after schema.org and the GIF vocabulary |
| queria-write-page | Writing a report from SQL and charts, and saving it with `create_report` or `queria report create` |
| queria-compare-areas | Comparing places in Japan for a location decision (where to open a shop, where to live) from population, daytime population, stations, competitors and land prices |

The skill itself lives in [skills/queria/SKILL.md](skills/queria/SKILL.md); common queries are in [references/sql-recipes.md](skills/queria/references/sql-recipes.md), and how to report problems is in [references/feedback.md](skills/queria/references/feedback.md).

When exploration turns up a missing dataset, a data defect, or a documentation error, the skill can report it with `submit_feedback` or `uvx queria feedback send`. Nothing is ever sent without you seeing the exact text and agreeing to it, and agents need your explicit permission before they can send anything at all.

Through the connector, results are shown as a table, chart or map with `show_chart`. With the CLI, results are exported to CSV/Parquet for other tools.

If you publish data rather than consume it, the two publishing skills cover the whole path from `queria create` to a dataset someone else can read. The reference is at [docs.queria.io/en/publish](https://docs.queria.io/en/publish).

## Requirements

Either of these:

- The [MCP connector](https://docs.queria.io/en/mcp) `https://queria.io/mcp` (bundled in the plugin as `.mcp.json`). Needs a free Queria account; the free plan runs up to 50 queries a week. Counting around a point (trade area functions) works only here
- The [queria CLI](https://docs.queria.io/en) (PyPI: `queria`), for agents with a shell. With uv, `uvx queria` needs no install; otherwise `pip install queria` (Python 3.10+). No authentication required (anonymous access is [rate-limited](https://docs.queria.io/en/authentication))

In claude.ai's code execution, the CLI works when "Allow network egress" is on and the allowlist includes `pypi.org`, `files.pythonhosted.org`, `extensions.duckdb.org`, `queria.io` and `data.queria.io` (or all domains).

When both are there, the skills use the CLI, except for counting around a point, which needs the connector. queria-compare-areas counts around points at every step, so it uses the connector throughout when it is there.

## Data

The skills contain only instructions; they send nothing themselves. The SQL the agent writes, and the feedback you approve, go to Queria (queria.io) through the connector or the CLI. Through the connector, result rows go back to the agent and so to its provider. See [MCP server](https://docs.queria.io/en/mcp).

## How it works

Queria publishes each dataset as a DuckLake catalog. The queria CLI ATTACHes DuckLake read-only and queries it, just like the web app. The dataset list and all schemas are consolidated into `mart_*` tables in the `catalog` dataset, which `list` / `search` / `info` use for discovery. Compatibility between the catalog format and the CLI is managed by the CLI, which reports required upgrades in its error messages.

## License

MIT
