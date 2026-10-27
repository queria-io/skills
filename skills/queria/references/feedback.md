# Reporting gaps and problems to Queria

Queria's catalog grows from what people actually need. A gap you hit during
exploration is signal nobody else has, and it is lost unless it gets reported.

**The one thing you must never do is send text the user has not seen and agreed to.**
It goes to Queria in their name, from their machine.

## What kind of report is it

| `--kind` | Use for |
|---|---|
| `dataset_request` | The data does not exist in the catalog at all |
| `data_issue` | The data exists but is wrong, stale, or has unexpected gaps |
| `docs_issue` | docs.queria.io is wrong, missing something, or misleading |
| `cli_issue` | The `queria` CLI or MCP server itself misbehaves |
| `other` | Anything else, including plain thanks (default) |

Add `--dataset <name>` whenever the report is about a specific dataset — it is what
makes requests countable, so reports carrying it get acted on first.

## From a shell (CLI)

```bash
uvx queria feedback send "市区町村別の待機児童数が欲しい" --kind dataset_request
```

Without `--yes` this prints the exact payload and asks for confirmation. That is the
right flow when the user is at the terminal.

When you are driving the CLI yourself, show the user the message text, get their
agreement, and only then pass `--yes`:

```bash
uvx queria feedback send "e_stat の人口テーブルに2025年が入っていない" \
  --kind data_issue --dataset e_stat --yes
```

`--yes` skips the prompt, not the disclosure — the payload is still printed. Do not
reach for it as a way to avoid asking. Without a terminal and without `--yes` the
command refuses to send, by design.

## From MCP (`submit_feedback`)

The tool enforces the review step for you. Do not try to route around it.

1. Call `submit_feedback` with `kind`, `message` and (if relevant) `dataset`.
2. The first call always returns `status: "needs_approval"`. Show the `preview`
   **verbatim** and ask whether to send it. Only if they agree, call `submit_feedback`
   again with the same `kind`, `message` and `dataset` plus the `confirm_token`. If they
   decline, do nothing — no retry, no rephrasing.
3. `status: "sent"` means it is stored. Say so once, briefly.
4. When it cannot send (usually a credential in the text or a too-long message), the
   tool returns an error with the reason. Fix it and, if the user still agrees, try again.

`confirm_token` is a signature over the draft and expires after an hour. Changing the
text means starting over from step 1, not guessing another value.

The tool is there only when the user has connected an account.

## Writing a report worth acting on

Good — specific, reproducible, no data pasted in:

> `e_stat.main.mart_population` は2024年までしか入っていない。2025年の推計人口
> （総務省が2025年4月公表）を取り込んでほしい。`area` 別の時系列で使いたい。

> `zipcode.main.mart_zipcode` の `lg_code` が5桁の行と6桁の行が混ざっていて、
> `lg_code.main.mart_lg_code` と直接 JOIN できない。`SELECT DISTINCT length(lg_code)`
> で再現する。

Bad:

> データがおかしい

> （クエリ結果を1000行貼り付けたもの）

> ユーザーの田中さん（tanaka@example.com）が困っています

Keep it to a few sentences: what you expected, what you found, and how to see it.

## What gets sent

The message, the kind, the dataset name, an anonymous per-machine id, the tool
version, whether it came from the CLI or MCP, and the name of the agent runtime.
If the user has an API token configured it is sent so the report can be attached to
their account — the server resolves that, the client never handles an account id.

There is no separate contact field. If the user wants a reply and is not logged in,
they have to say so in the message; ask before putting any contact detail in there.

The CLI refuses to send a message containing something that looks like an API token
or a credential. Treat that refusal as a real finding, not an obstacle to work
around: something leaked into the text that should not be there.
