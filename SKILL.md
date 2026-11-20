---
name: planning-multi-agent-strategy
description: Use when about to dispatch three or more agents for one task, start a Workflow, fan out subagents, or split a large task across models or external workers (GLM, Qwen, Codex, Kimi, DeepSeek) - before the first dispatch, and whenever choosing a model, effort level or executor for delegated work.
---

# Planning Multi-Agent Strategy

The user approves the strategy once, before anything starts, so they never
have to interrupt a run to correct the model, the effort or the breakdown.

For routing and for when to dispatch, this skill takes precedence over the
model-orchestration delegation defaults. That plugin still supplies the
worker machinery (`~/.claude-workers/<name>/run`, the `worker-dispatch` agent).

## Does the gate apply?

Count the dispatches the WHOLE task will need (Claude subagents + external
workers), not the dispatches in this turn. A row marked x5 is five. Re-sends
after a failed audit do not count.

- **3 or more, or any Workflow:** follow steps 1-6. The gate in step 4 applies.
- **1 or 2:** pick executor and effort from the routing table, say each choice
  in one line (naming the provider if a worker gets the data), and dispatch.
  No pause. Still audit any worker result yourself.

## Steps

1. **Break the task down.** List subtasks, what depends on what, what runs in
   parallel, which files each one owns, and where results get audited. A file
   that several subtasks depend on gets exactly one owning row.
2. **Check which executors exist.** `grep -L PASTE_YOUR_API_KEY_HERE ~/.claude-workers/*/settings.json` lists
   plugin workers that have a key (a worker without one is not usable, and
   none are usable unless `command -v claude` finds the CLI);
   `command -v <cli>` for each entry under Other CLIs. Only these are usable.
3. **Fill in the strategy table** (format below). Every cell is required.
4. **Send the table and end your turn.** Until the user replies to the table:
   no Agent call, no Workflow, no worker command, and no file changes toward
   the task (no branch, no installs, no shared helpers). Reading is fine.
   If the user edits rows, re-send the changed rows and end your turn again.
5. **Dispatch exactly as approved.** Every Claude subagent, whether an Agent
   call or a Workflow `agent()`, passes the row's `model` AND `effort`.
   Workers get the row's command. No rows added, merged or re-routed.
   Where an orchestrator row sits between dispatch rows, split the Workflow
   into one script per stage and do the orchestrator row between them.
6. **Audit and escalate.** Audit each result against its criteria.
   - First failed audit: send the same executor back with specific fixes.
     Findings from a later row (a review, a design check) sent back to the
     row that made the change are this same kind of re-send.
   - Second failed audit of the same subtask: you may escalate it ONE step
     up the ladder, once, without asking. Only the failing copy of an xN row
     moves. The new executor repairs the existing output. Tell the user in
     one line when it happens and list it in the final report.
   - Ladder: worker -> `sonnet` -> `opus` -> orchestrator; `haiku` -> `sonnet`.
     A row moving from a worker to a Claude model takes the Effort in its
     routing row. A row already on a Claude model may instead go one effort
     step up (`low` -> `medium` -> `high`) on the same model.
   - Anything else needs the user's yes first: a second escalation of the
     same subtask, new work discovered mid-run, dropping or reverting a row,
     changing any other row. Propose it as a single table row and keep the
     already-approved rows running while you wait.

The final report states what ran against the approved table and every
escalation made.

## Strategy table (the message you send in step 4)

```
| # | Subtask | Executor | Model | Effort | Why (routing row) |
|---|---------|----------|-------|--------|-------------------|
| 1 | Map auth module, 12 files   | subagent     | haiku    | low    | mechanical, read-only search |
| 2 | Design new session flow     | orchestrator | -        | -      | design                       |
| 3 | Implement handlers (x3)     | worker codex | (codex)  | -      | build, well-specced          |
| 4 | Check against Figma         | subagent     | sonnet   | medium | tools, needs connector       |
| 5 | Audit each result from 3    | orchestrator | -        | -      | audit                        |
| 6 | Review whole change         | subagent     | opus     | high   | review                       |

Runs: 1 -> 2 -> 3 (three in parallel) -> 4, 5 -> 6
Leaves this machine: handler code and spec to OpenAI (codex)
Workers skipped as not configured: none
Approve, edit rows, or change tiers?
```

One row per subtask; parallel copies of the same subtask share a row with a
count, and each worker row gets its own audit row. For workers, Model is the
worker's model, or its name in brackets if unknown, and Effort is `-`. If a
worker named in the routing row was not found in step 2, name it on the
"Workers skipped" line with what was used instead.

## Routing table

Use the first row whose scope matches. Where a cell lists several workers,
use the first one found in step 2; if none is found, use the fallback.
"orchestrator" means do it yourself. Effort applies whenever the executor is
a Claude model, including a fallback or an escalation.

| Row | Scope of subtask | First choice | Fallback (no worker found) | Effort |
|---|---|---|---|---|
| integrate | Security-sensitive, irreversible, or final integration of the pieces | orchestrator | n/a | n/a |
| design | Architecture, design, task breakdown, writing specs for other rows | orchestrator | n/a | n/a |
| review | Independent review of the whole change | `opus` | n/a | high |
| audit | Checking one delegated result against its criteria | orchestrator | n/a | n/a |
| tools | Needs in-session tools (connectors, browser) | `sonnet` | n/a | medium |
| ambiguous | Implementation or debugging with no clear spec | `opus` | n/a | high |
| analysis | Read-only analysis that needs judgment: code audits, findings lists, root-cause | `sonnet` | n/a | medium |
| build | Well-specced implementation, tests, refactors, tooling setup | `codex`, `glm`, `qwen` | `sonnet` | medium |
| read | Token-heavy reading with a factual answer: large docs, logs, research sweeps | `glm`, `qwen` | `sonnet` | low |
| write-up | Notes, summaries and docs written from finished results | `sonnet` | n/a | low |
| mechanical | Search, rename, formatting, bulk edits | `haiku` | n/a | low |

`read` is for answers stated in the source; if the answer has to be inferred,
it is `analysis`. If no row matches, use the closest and write "closest: <row>" in the Why
cell. A spec written in a `design` row makes the work that follows it
well-specced (`build`).

When a wrong result would be costly to detect later: on a Claude row, raise
effort one step, up to `high`; on a worker row, use the fallback Claude model
instead of the worker. Say so in the Why cell. Use `xhigh` or `max` only when the user
asks for it in their reply to the table.

Every worker row needs an `audit` row. Any task with 3+ dispatches that
changes code needs a `review` row.

## Dispatch commands

| Executor | Command | Limits |
|---|---|---|
| Claude subagent | Agent tool with `model` and `effort` | - |
| Plugin worker (`glm`, `qwen`, any dir under `~/.claude-workers/`) | From the orchestrator, via Bash: `~/.claude-workers/<name>/run -p "<spec>"`, adding `--permission-mode acceptEdits` only when the subtask edits files. Inside a Workflow script, use the `model-orchestration:worker-dispatch` agent at `haiku`, `low` | No in-session tools, no effort setting, at most 8 running at once |
| Other CLIs: `codex` | Edits files: `codex exec -s workspace-write -C <repo> "<spec>"`. Read-only: `codex exec -s read-only -C <repo> "<spec>"`. Add `-o <file>` to capture the final message | Same limits. Never `--dangerously-bypass-approvals-and-sandbox`. Flags checked against codex-cli 0.170.0 |

Add a row here for each further standalone CLI. Worker specs are
self-contained: paths, acceptance criteria, files owned, "return a summary
with file paths".

## What counts as approval

Only the user's reply to the strategy table.

| Thought | Reality |
|---|---|
| "They said go fast / there's a deadline" | The table is one message. An interrupted run costs more time. |
| "They said 'I trust you' / 'just run with it'" | Said before they saw the table. It is not approval of the table. |
| "They're away, the pause will stall the work" | They chose this gate knowing that. Send the table and end the turn. |
| "They already asked for a workflow" | That picks the tool, not the routing. |
| "I'll describe the plan in prose and start" | Prose plus dispatch gives them nothing to edit. Table, then stop. |
| "A branch and shared helpers are only setup" | They are file changes toward the task. After approval. |
| "Sonnet at medium for everything is safe" | Each row gets its own routing row. Uniform tiers mean the table was skipped. |
| "The worker might not be worth the audit overhead" | The routing table already decided. Follow it or raise it in the table. |
