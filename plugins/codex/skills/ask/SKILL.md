---
name: ask
description: Send a prompt to Codex (OpenAI GPT models) and get its answer back, optionally continuing the same Codex thread. Use for adversarial review, second opinions, design challenges, and review-ledger work that Codex should record itself. Runs Codex in the current project with the user's Codex config, MCP servers, AGENTS.md, and hooks.
---

# Ask Codex

One Bash call per turn. Codex works in the project directly; you brief it,
wait, and relay its answer.

```bash
node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --json \
  [--write] [--model <model>] [--effort <level>] [--resume | --fresh] \
  [--cwd <project-root>] [--prompt-file <path> | "<prompt>"]
```

## Flags

| flag | meaning |
|---|---|
| `--write` | Sandbox `workspace-write` instead of `read-only`. Needed whenever Codex must change files, including recording findings on a review ledger. |
| `--model` | Codex model. Omit to use the Codex config default. Pass one when the user or their instructions name one. |
| `--effort` | `none`, `minimal`, `low`, `medium`, `high`, `xhigh`. Omit for the config default. |
| `--resume` | Continue the most recent finished Codex thread from this Claude session. The thread keeps its model. |
| `--fresh` | Start a new thread (the default when neither flag is given). |
| `--cwd` | Project root. Defaults to the shell's working directory. |
| `--prompt-file` | Read the prompt from a file. Use it for long briefs (write the brief to the scratchpad first). |

## Running it

- Codex runs can take minutes. Use Bash with a 600000 ms timeout, or
  `run_in_background: true` for long reviews, and wait for the notification.
- Output is JSON: `status` (0 on success), `threadId`, `rawOutput` (Codex's
  answer), `touchedFiles`, `reasoningSummary`. `touchedFiles` lists only
  Codex's own patch edits; files changed by commands it ran (for example a
  ledger CLI) do not appear. Check those with `git status` or the tool itself.
- Codex sees the project's MCP servers and `.codex/config.toml` only when the
  project is trusted in `~/.codex/config.toml`. If expected tools are missing,
  say so rather than working around it.

## Writing the brief

Codex has none of this conversation. State:

- the subject (paths, diff range, design doc, ledger id);
- the stance: adversarial, so challenge the approach and its assumptions, not
  only the line-level defects;
- where findings go: in the reply, or on a ledger (give the review id, the
  raiser label, and whether to conclude the pass);
- what Codex must not change.

For ledger reviews, point Codex at the project's own review workflow (for
example doctrine's `code-review` skill and `doctrine review raise`) instead of
restating it.

## Reporting back

Relay `rawOutput` faithfully and attribute it to Codex. Keep Codex's claims
apart from what you have verified. Do not act on its findings unless the user
asked you to. On a non-zero `status`, report the error text as-is.
