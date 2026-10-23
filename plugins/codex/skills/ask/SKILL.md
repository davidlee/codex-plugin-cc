---
name: ask
description: Send a prompt to Codex (OpenAI GPT models) and get its answer back, optionally continuing the same Codex thread. Use for adversarial review (including doctrine review ledgers), second opinions, and delegating bounded work. Runs Codex in the current project with the user's Codex config, MCP servers, AGENTS.md, and hooks.
---

# Ask Codex

One Bash call per Codex turn. Codex works in the project directly; you brief
it, wait, and relay its answer.

```bash
node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --json \
  [--write] [--model <model>] [--effort <level>] [--resume | --fresh] \
  [--cwd <project-root>] [--prompt-file <path> | "<prompt>"]
```

## Choose a mode

Pick a model tier, not a model name. This skill never names models, because
they change every few months. Resolve each tier from the user's own
instructions (CLAUDE.md, typically a GPT or Codex section), which they keep
current. If the user's instructions name no model for a tier, omit `--model`
and say which tier you wanted. Explicit instructions in the conversation
override everything here.

| mode | tier | effort | sandbox |
|---|---|---|---|
| Adversarial review | strong (the user's reviewer model) | `medium`; `high` for substantial changes or designs | read-only; `--write` when Codex records findings on a ledger |
| Adversarial review, very complex (large cross-cutting design, concurrency, security, data integrity) | strongest | `high` | as above |
| General delegation | default: omit `--model` (Codex config default) | omit | `--write` only if Codex should change files |
| General delegation, especially complex | strong | omit, or `high` | as above |

## Adversarial review

Codex has none of this conversation. The brief states:

- the subject: paths, diff range (`git diff <base>...HEAD`), design doc, or plan;
- the stance: challenge whether the approach is right, the assumptions it rests
  on, and how it fails under real conditions, not only line-level defects;
- the governing intent, if any (spec, design, acceptance criteria) to judge
  against;
- that every finding cites evidence (file and line, or a reproduction) and
  marks inference as inference;
- what Codex must not change (normally: everything except the ledger).

Without a ledger, Codex replies with findings ordered by severity, read-only.

### On a doctrine review ledger

Applies when the project uses doctrine (a `.doctrine/` directory). Codex is the
raiser; you are the responder.

1. Open or identify the RV ledger for the subject (`doctrine review new
   --facet <facet> --target <ref> --raiser codex`), unless one exists.
2. Brief Codex with `--write`: the RV id, raiser label `codex`, the facet, and
   an instruction to follow the project's doctrine review workflow (its
   doctrine skills, and the doctrine MCP tools or `doctrine review` CLI) to
   raise findings and conclude the pass. Do not restate doctrine's verbs.
3. Check the ledger yourself (`doctrine review show <RV>`); do not rely on
   Codex's summary of it.
4. Dispose the findings as responder.
5. For the raiser's next round (verify, contest, reopen), call again with
   `--resume` and tell Codex the dispositions are in.

## General delegation

State the goal, constraints, the files in scope, what "done" means, and how
Codex should report (for example: changes made, evidence, open questions).
Use `--write` only when Codex should edit; afterwards review its changes with
`git diff` before building on them.

## Flags

| flag | meaning |
|---|---|
| `--write` | Sandbox `workspace-write` instead of the default `read-only`. |
| `--model` | Codex model. Omitted: the Codex config default. Do not change the model on `--resume`; the thread keeps its own. |
| `--effort` | Reasoning effort, passed through to Codex. Supported levels vary by model (`codex debug models` lists them per model; `minimal` is rejected by current models). Omitted: the config default. |
| `--resume` | Continue the most recent finished Codex thread from this Claude session, whatever its mode. Run unrelated threads in sequence, not interleaved, or start fresh. |
| `--fresh` | Start a new thread (the default when neither flag is given). |
| `--cwd` | Project root. Defaults to the shell's working directory. |
| `--prompt-file` | Read the prompt from a file. Use it for long briefs (write the brief to the scratchpad first). |

## Running it

- Reviews and delegated work take minutes. Use Bash with
  `run_in_background: true` and wait for the completion notification; for
  short questions, a foreground call with a 600000 ms timeout.
- Output is JSON: `status` (0 on success), `threadId`, `rawOutput` (Codex's
  answer), `touchedFiles`, `reasoningSummary`. `touchedFiles` lists only
  Codex's own patch edits; files changed by commands it ran (for example the
  doctrine CLI) do not appear. Check those with `git status`.
- Codex sees the project's MCP servers and `.codex/config.toml` only when the
  project is trusted in `~/.codex/config.toml`. If expected tools are missing,
  say so rather than working around it.

## Reporting back

Relay `rawOutput` faithfully and attribute it to Codex. Keep Codex's claims
apart from what you have verified. On a non-zero `status`, report the error
text as-is and stop; do not substitute your own answer for Codex's.
