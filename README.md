# planning-multi-agent-strategy

A Claude Code skill that makes the orchestrator show you its plan before a
large multi-agent task starts: which subtasks it will dispatch, to which model
or external worker, and at what effort. You approve or edit that table once,
and the run then follows it.

It exists because the alternative is interrupting a run halfway to say "that
subtask didn't need the biggest model" or "don't split it that way".

## What it does

- **Gate.** For a task that needs 3 or more dispatches, or any Workflow, Claude
  sends a strategy table and ends its turn. Nothing is dispatched and no files
  change until you reply. For 1 or 2 dispatches it picks from the routing table
  and proceeds without pausing.
- **Routing table.** Eleven named rows (`design`, `review`, `build`, `read`,
  `mechanical`, ...) map the scope of a subtask to an executor and an effort
  level, with a Claude fallback when no external worker is available.
- **External workers.** Rows can route to headless workers on other providers
  (GLM, Qwen, and others set up under `~/.claude-workers/`) and to standalone
  CLIs such as Codex. The table says what data leaves your machine and to whom.
- **Bounded escalation.** After a second failed audit of a subtask, Claude may
  move it one step up the ladder, once, and must tell you. Anything more needs
  your yes.

Example of the message you get before anything starts:

```
| # | Subtask | Executor | Model | Effort | Why (routing row) |
|---|---------|----------|-------|--------|-------------------|
| 1 | Map auth module, 12 files   | subagent     | haiku    | low    | mechanical, read-only search |
| 2 | Design new session flow     | orchestrator | -        | -      | design                       |
| 3 | Implement handlers (x3)     | worker codex | (codex)  | -      | build, well-specced          |
| 4 | Audit each result from 3    | orchestrator | -        | -      | audit                        |
| 5 | Review whole change         | subagent     | opus     | high   | review                       |

Runs: 1 -> 2 -> 3 (three in parallel) -> 4 -> 5
Leaves this machine: handler code and spec to OpenAI (codex)
Workers skipped as not configured: none
Approve, edit rows, or change tiers?
```

## Install

```bash
git clone https://github.com/lucoile/planning-multi-agent-strategy ~/.claude/skills/planning-multi-agent-strategy
```

Then add a pointer to `~/.claude/CLAUDE.md` so the skill is loaded reliably,
not only when its description happens to match:

```markdown
# Multi-agent work

Before dispatching 3 or more agents or external workers for one task, or starting any Workflow, load the `planning-multi-agent-strategy` skill and follow it: send the strategy table and wait for my reply before the first dispatch.
For 1 or 2 dispatches, pick model and effort from that skill's routing table without pausing.
```

## External workers (optional)

The skill works with Claude subagents alone. Worker rows fall back to a Claude
model when no worker is found.

- **Plugin workers** (`glm`, `qwen`, ...) are headless Claude Code instances
  pointed at another provider's Anthropic-compatible endpoint, one directory
  each under `~/.claude-workers/<name>/` with a `run` wrapper. The skill expects
  the layout created by the `model-orchestration` plugin, and takes precedence
  over that plugin's delegation defaults for routing. It needs the `claude` CLI
  on your PATH.
- **Codex** is used through `codex exec` if `command -v codex` finds it.

## Make it yours

The routing table is one person's preferences. Edit `SKILL.md` directly:

- Reorder or replace the workers in the `build` and `read` rows.
- Change the gate threshold (3 dispatches) in "Does the gate apply?" and in
  your `CLAUDE.md` pointer.
- Add a row under "Dispatch commands" for any other standalone CLI.

## How it was tested

Written test-first: three baseline runs without the skill all dispatched
immediately (4, 5 and 22 agents) with a prose plan and near-uniform model
choices. With the skill, the same scenarios sent the table and stopped, and a
2-dispatch task proceeded without a pause. The wording of the escalation rule
was revised after those runs and has not been re-tested.

## License

MIT
