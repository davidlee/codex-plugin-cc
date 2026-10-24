# Codex plugin for Claude Code (fork)

Use Codex from inside Claude Code: let Claude ask Codex for adversarial
reviews and second opinions, delegate work to it, and run Codex code reviews.

This is a fork of [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc).
Versions are upstream's plus a `-dl.N` suffix (for example `1.0.6-dl.6`).

## Why this fork exists

Claude should be able to run Codex in a project exactly as you would run it
yourself there: with your Codex config, the project's MCP servers,
`AGENTS.md` and hooks. Upstream overrides parts of your config, and that
breaks MCP tools:

- It forces `approvalPolicy: "never"` on every thread. Codex then refuses MCP
  tool calls that would otherwise go through your approval setup.
- It forces a sandbox (`read-only`, or `workspace-write` with `--write`).
  Where your config deliberately sets `sandbox_mode = "danger-full-access"`
  with `approval_policy = "never"` (for example inside a container or jail
  that is itself the sandbox), the forced sandbox makes Codex refuse every
  MCP tool call.

The fork also replaces upstream's `/codex:rescue` command and its subagent
with one skill, `codex:ask`. Claude uses it directly for adversarial review,
including rounds on [doctrine](#adversarial-review-on-a-doctrine-review-ledger)
review ledgers, and for delegation, and it continues a Codex thread across
turns.

### Differences from upstream

| | upstream | this fork |
|---|---|---|
| Approval policy | forced to `never` | your Codex config |
| Task sandbox | forced to `read-only` (`workspace-write` with `--write`) | your Codex config; `--write` / `--read-only` override it |
| Delegation | `/codex:rescue` command and `codex-rescue` subagent | `codex:ask` skill |
| Model choice | built-in model aliases | `--model` is passed to Codex verbatim; the skill picks a tier that your own instructions map to a model |
| `--effort` | checked against a fixed list | passed to Codex verbatim |

Removed: `/codex:rescue`, the `codex-rescue` agent, and the three internal
skills that supported them (runtime contract, prompting guide, result
handling).

## Requirements

- A ChatGPT subscription (including Free) or an OpenAI API key. Usage counts
  against your [Codex usage limits](https://developers.openai.com/codex/pricing).
- Node.js 18.18 or later.
- The Codex CLI (`npm install -g @openai/codex`), signed in.

## Install

The fork keeps upstream's marketplace and plugin names (`openai-codex`,
`codex@openai-codex`). If you have upstream's marketplace installed, remove it
first (`/plugin marketplace remove openai-codex`).

```bash
/plugin marketplace add davidlee/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

`/codex:setup` reports whether Codex is installed and signed in. If Codex is
missing and npm is available, it offers to install it. If Codex is installed
but not signed in, run `!codex login`.

To update, run `claude plugin marketplace update openai-codex`, then
`claude plugin update codex@openai-codex`, then restart Claude Code.

## `codex:ask`

A skill Claude invokes itself when you ask it to consult Codex or delegate to
it. Each Codex turn is one call to the plugin's companion `task` command;
Claude briefs Codex, waits, and relays the answer, keeping Codex's claims
apart from what it has verified itself.

```text
Ask Codex for an adversarial review of this branch against main.
Have Codex review the design in docs/cache.md, then resume it once I've answered.
Delegate the flaky-test investigation to Codex.
```

### Modes

- **Adversarial review.** Codex challenges the approach, its assumptions and
  how it fails under real conditions, not only line-level defects. Every
  finding cites evidence. Codex changes nothing except a review ledger.
- **General delegation.** Claude states the goal, constraints, files in scope,
  what "done" means and how to report. If Codex edits files, Claude reviews
  the diff before building on it.

`--resume` continues the most recent finished Codex thread from the current
Claude session; a resumed thread keeps its model. `--fresh` (the default)
starts a new one.

### Adversarial review on a doctrine review ledger

When the project uses doctrine (it has a `.doctrine/` directory), reviews run
on an RV review ledger, with Codex as the raiser and Claude as the responder:

```text
Claude: open the ledger (doctrine review new --raiser codex …)
  └─▶ Codex: raise findings, conclude the pass (doctrine MCP tools or CLI)
        └─▶ Claude: read the ledger itself; dispose each finding
              └─▶ Codex (--resume): verify, contest or reopen
```

Codex writes the ledger with its usual sandbox, so this needs a config in
which Codex can write to the project (`workspace-write` or
`danger-full-access`).

### Model tiers

The skill never names models; they change every few months. It picks a tier
and resolves it from your own instructions, typically your `CLAUDE.md`:

| mode | tier | effort |
|---|---|---|
| Adversarial review | strong | `medium`; `high` for substantial changes |
| Adversarial review, very complex (concurrency, security, data integrity) | strongest | `high` |
| General delegation | default (Codex config) | config default |
| General delegation, especially complex | strong | config default or `high` |

Map the tiers in your `CLAUDE.md`, for example:

```markdown
## GPT

Model tiers for codex:ask:
- default: omit `--model` (Codex config default)
- strong: <model>
- strongest: <model>
```

`codex debug models` lists the models your account serves and the effort
levels each supports. If no model is mapped to a tier, Claude omits `--model`
and says which tier it wanted.

### Sandbox

By default `codex:ask` sends no sandbox, so Codex uses `sandbox_mode` from
your config. `--write` forces `workspace-write`; `--read-only` forces
`read-only`. With `approval_policy = "never"`, either flag turns Codex's own
sandbox on, and Codex then refuses all MCP tool calls. The skill therefore
leaves both flags off unless a run must not be able to write.

## Commands

### `/codex:review`

Runs a normal Codex code review of your uncommitted changes, or of your
branch against a base with `--base <ref>`. It matches `/review` inside Codex.
It is not steerable; use `/codex:adversarial-review` or `codex:ask` for
focused challenges. It supports `--wait` and `--background`, and it is
read-only.

```bash
/codex:review
/codex:review --base main --background
```

### `/codex:adversarial-review`

A steerable review that questions the chosen design, tradeoffs and failure
modes. It takes the same target options as `/codex:review`, plus focus text.
It is read-only.

```bash
/codex:adversarial-review --base main challenge the caching and retry design
```

### `/codex:status`, `/codex:result`, `/codex:cancel`

Track Codex jobs for the current repository. `/codex:status [job-id]` shows
running and recent jobs; for a task it includes the sandbox it ran with
(`config`, `workspace-write` or `read-only`). `/codex:result [job-id]` prints
a finished job's output and its Codex session ID. `/codex:cancel [job-id]`
stops a background job.

### `/codex:transfer`

Creates a persistent Codex thread from the current Claude Code session and
prints a `codex resume <session-id>` command, so you can continue the same
conversation in the Codex app or TUI. `--source <path>` overrides the
transcript path that the plugin's `SessionStart` hook supplies; the source
must be under `~/.claude/projects`. It requires a Codex version that exposes
session import.

### `/codex:setup`

Checks that Codex is installed and signed in, and manages the optional review
gate:

```bash
/codex:setup --enable-review-gate
/codex:setup --disable-review-gate
```

With the gate on, a `Stop` hook runs a read-only Codex review of Claude's last
turn and blocks the stop if Codex finds issues.

> [!WARNING]
> The review gate can create a long-running Claude/Codex loop and drain usage
> limits quickly. Enable it only while you are watching the session.

## How it works

```text
Claude ──codex:ask──▶ codex-companion.mjs task
        ──unix socket──▶ app-server broker (one per workspace)
        ──stdio JSON-RPC──▶ codex app-server ──▶ OpenAI
```

The plugin wraps the [Codex app server](https://developers.openai.com/codex/app-server)
and uses the `codex` binary on your `PATH`, with its authentication and
[configuration](https://developers.openai.com/codex/config-basic):

- user config in `~/.codex/config.toml`;
- project config in `.codex/config.toml`, which loads only when the
  [project is trusted](https://developers.openai.com/codex/config-advanced#project-config-files-codexconfigtoml).
  MCP servers defined there load only then too.

Task threads persist, so you can reopen any of them in Codex with
`codex resume <session-id>`, using the ID from `/codex:status` or
`/codex:result`.
