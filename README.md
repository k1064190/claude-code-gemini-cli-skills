# claude-code-gemini-cli-skills

Claude Code skills for delegating tasks to CLI agents — Google Antigravity (`agy`), Google Gemini, OpenAI Codex, and Claude Code itself (`claude -p`) — running as subagents.

> **Heads up:** Google is retiring the Gemini CLI (free tier ended 2026-06-18) in favor of the **Antigravity CLI** (`agy`). New work should prefer [`antigravity-subagent`](./antigravity-subagent/SKILL.md); `gemini-subagent` is kept for environments still on the old `gemini` binary.

## Skills

### [`antigravity-subagent`](./antigravity-subagent/SKILL.md)

Delegate tasks from Claude to Google Antigravity CLI (`agy`), the Gemini CLI's successor, running as an independent agent. Useful for:

- **Large codebase analysis** — Antigravity's large context window handles entire repos that overflow Claude's context
- **Multi-model second opinion** — One CLI fronts Gemini 3.x, Claude Sonnet/Opus 4.6, and GPT-OSS
- **Parallel execution** — Run multiple `agy` instances concurrently via parallel Bash calls
- **Plain-text output** — `agy -p` prints the final answer directly; no JSON/`jq` step

### [`gemini-subagent`](./gemini-subagent/SKILL.md)

Delegate tasks from Claude to Gemini CLI running as an independent agent. Useful for:

- **Large codebase analysis** — Gemini's massive context window handles entire repos that overflow Claude's context
- **Parallel execution** — Run multiple Gemini instances concurrently via parallel Bash calls
- **Google Search grounding** — Leverage Gemini's built-in web search
- **Second opinion** — Compare results from two different AI models

### [`codex-subagent`](./codex-subagent/SKILL.md)

Delegate tasks from Claude to OpenAI Codex CLI running as an independent agent. Useful for:

- **Code analysis and refactoring** — Hand off review, refactor, or automated edit work to Codex
- **Cross-model second opinion** — Compare Codex's take against Claude's own conclusions
- **Resumable sessions** — Continue prior Codex sessions with `codex exec resume --last`
- **Sandboxed execution** — Run with `read-only`, `workspace-write`, or `danger-full-access` modes

### [`claude-subagent`](./claude-subagent/SKILL.md)

Delegate tasks to a fresh Claude Code CLI run (`claude -p`). Useful for:

- **Clean context** — A cold instance reads the code without the current conversation's assumptions
- **Scoped permissions** — `--tools` bounds which tools exist, `--permission-mode dontAsk` denies the rest, and `--setting-sources user` drops the target repo's own settings and hooks, which otherwise run shell commands outside the tool boundary
- **Parallel execution** — Run several `claude -p` instances concurrently via parallel Bash calls
- **Orchestration from other CLIs** — Codex or Antigravity can call Claude for a subtask

Not a second opinion: the subagent is the same model family, so it shares your blind spots. Use `codex-subagent` or `antigravity-subagent` for a genuine cross-model check.

## Defaults

These skills pin a single default model per agent and switch only when the user explicitly asks for a different one. This keeps behavior predictable across sessions:

- **Antigravity**: always invoke with `--model "Gemini 3.1 Pro (High)"`. Switch to a Flash, Claude, or GPT-OSS model only when the user explicitly requests it. Run `agy models` to see the exact strings available.
- **Gemini**: always invoke with `-m pro` (`gemini-3.1-pro-preview`). Switch to `-m flash` only when the user explicitly requests speed/flash.
- **Codex**: always invoke with model `gpt-5.6-sol` and reasoning effort `high`. Switch only when the user explicitly names a different model or effort (e.g., "use gpt-5.6-luna", "set effort to xhigh"). Always pass the explicit tier id — the bare `gpt-5.6` alias is rejected on ChatGPT-account auth.
- **Claude**: always invoke with `--model opus` and **without `--bare`**. Bare mode never reads OAuth or the keychain, so on a subscription account it fails with `"Not logged in · Please run /login"`; it is safe only with a non-OAuth credential source (`ANTHROPIC_API_KEY`, an `apiKeyHelper` via `--settings`, or Bedrock / Google Cloud / Foundry credentials). Bound the tools with **`--tools` plus `--permission-mode dontAsk --setting-sources user`** — `--allowedTools` alone only suppresses prompts (under a permissive ambient mode an unlisted `Edit` or `Bash` just runs), and the target repo's own `.claude/settings.json` hooks execute shell commands outside the tool boundary unless the project settings are dropped. Switch to `sonnet`/`haiku` only when the user explicitly asks. Worth surfacing before a large fan-out: a trivial `opus` call still costs ~$0.40, since a non-bare run loads CLAUDE.md, plugins, and skills.

## Requirements

### Antigravity subagent

- [Antigravity CLI](https://antigravity.google) (`agy`) installed and authenticated
- `tmux` (optional — only needed for explicit background execution)

```bash
# Install Antigravity CLI (installs to ~/.local/bin/agy)
curl -fsSL https://antigravity.google/cli/install.sh | bash

# Ensure ~/.local/bin is on PATH, then open a new shell
agy install

# Authenticate (one-time interactive sign-in), then verify
agy
agy --version

# One-time: import existing Gemini CLI config as plugins (non-destructive)
agy plugin import gemini
```

### Gemini subagent

- [Gemini CLI](https://github.com/google-gemini/gemini-cli) installed and authenticated
- `jq` for JSON parsing
- `tmux` (optional — only needed for explicit background execution)

```bash
# Install Gemini CLI
npm install -g @google/gemini-cli

# Authenticate
gemini
```

### Codex subagent

- [Codex CLI](https://github.com/openai/codex) installed and authenticated

```bash
# Install Codex CLI
npm install -g @openai/codex

# Authenticate (one-time interactive login)
codex
```

### Claude subagent

- [Claude Code](https://code.claude.com/docs/en/setup) installed and authenticated
- `jq` for JSON parsing

```bash
# Verify the binary and the auth state
claude --version
claude auth status

# Authenticate (one-time interactive login)
claude auth login
```

## Installation

Copy the skill folders into your Claude Code skills directory:

```bash
cp -r antigravity-subagent ~/.claude/skills/
cp -r gemini-subagent      ~/.claude/skills/
cp -r codex-subagent       ~/.claude/skills/
cp -r claude-subagent      ~/.claude/skills/
```

Or symlink them:

```bash
ln -s $(pwd)/antigravity-subagent ~/.claude/skills/antigravity-subagent
ln -s $(pwd)/gemini-subagent      ~/.claude/skills/gemini-subagent
ln -s $(pwd)/codex-subagent       ~/.claude/skills/codex-subagent
ln -s "$(pwd)/claude-subagent"    ~/.claude/skills/claude-subagent
```

## Quick start

### Antigravity

```bash
# Default invocation — always Gemini 3.1 Pro (High); plain-text output, no jq
agy --model "Gemini 3.1 Pro (High)" -p "Working directory: /path/to/project. Analyze the architecture and suggest improvements." --dangerously-skip-permissions 2>/dev/null

# Large codebase analysis (@ syntax, no --dangerously-skip-permissions needed)
cd /path/to/project
agy --model "Gemini 3.1 Pro (High)" -p "@src/ @tests/ Explain the architecture and identify missing test coverage" 2>/dev/null

# Continue the most recent conversation
agy -c -p "follow-up question" --dangerously-skip-permissions 2>/dev/null
```

### Gemini

```bash
# Default invocation — always -m pro
gemini -m pro -p "Working directory: /path/to/project. Analyze the architecture and suggest improvements." --yolo --output-format json 2>/dev/null | jq -r '.response'

# Large codebase analysis (@ syntax, no --yolo needed)
cd /path/to/project
gemini -m pro -p "@src/ @tests/ Explain the overall architecture and identify any missing test coverage"

# Explicit speed mode — only when the user asks for it
gemini -m flash -p "What is the capital of France?" --output-format json 2>/dev/null | jq -r '.response'
```

### Codex

```bash
# Default invocation — gpt-5.6-sol with high reasoning effort, read-only sandbox.
# `< /dev/null` is required: without it codex can block forever on a non-TTY stdin.
# Capture stderr to a file instead of discarding it (see codex-subagent/SKILL.md).
ERRLOG=$(mktemp)
codex exec --skip-git-repo-check \
  -m gpt-5.6-sol \
  --config model_reasoning_effort="high" \
  --sandbox read-only \
  "Review this diff for correctness and security issues." < /dev/null 2>"$ERRLOG"

# Resume the most recent session (inherits model / effort / sandbox; echo | supplies EOF)
echo "follow-up question" | codex exec --skip-git-repo-check resume --last 2>"$ERRLOG"
```

### Claude

```bash
# Default invocation — opus, no --bare, read-only tools.
# `< /dev/null` matters: claude -p waits for EOF whenever stdin carries data.
# --output-format json emits a JSON *array* of events, so take the last one.
ERRLOG=$(mktemp); OUT=$(mktemp)
timeout -k 10 600 claude -p "Working directory: /path/to/project. Analyze the architecture and suggest improvements." \
  --model opus \
  --tools "Read,Glob,Grep" --allowedTools "Read,Glob,Grep" \
  --permission-mode dontAsk --setting-sources user --strict-mcp-config \
  --output-format json \
  < /dev/null >"$OUT" 2>"$ERRLOG"
rc=$?

# A run can fail while exiting 0 — gate on all three signals before trusting .result
LAST=$(jq -c 'if type=="array" then last else . end' "$OUT" 2>/dev/null)
[ -z "$LAST" ] && LAST='{}'
IS_ERR=$(printf '%s' "$LAST" | jq -r 'if .is_error == false then "false" else "true" end')
DENIALS=$(printf '%s' "$LAST" | jq -r '(.permission_denials // []) | length')
if [ "$rc" -ne 0 ] || [ "$IS_ERR" != false ] || [ "$DENIALS" != 0 ]; then
  echo "FAILED — exit=$rc is_error=$IS_ERR denials=$DENIALS" >&2
  printf '%s' "$LAST" | jq -r '.result // "(no result)"' >&2   # error text, not an answer
  cat "$ERRLOG" >&2
  rm -f "$OUT" "$ERRLOG"
  exit 1
fi

printf '%s' "$LAST" | jq -r '.result'
SID=$(printf '%s' "$LAST" | jq -r '.session_id')   # keep for --resume
rm -f "$OUT" "$ERRLOG"
```

Two rules the snippet encodes, both learned the hard way:

- **Never end a `claude -p` call with `| jq`.** The pipeline's exit code becomes jq's, so a run killed by `timeout` prints nothing and exits 0 — indistinguishable from a clean run with nothing to say. Redirect to a file, capture `rc`, then run `jq` against the file.
- **Never fall through to `.result` after a failed gate, and never exit 0 on one.** The failure text (`"Not logged in · Please run /login"`, or an observed false `"DONE"` from a denied edit) is a plausible-looking string; printed on stdout it reads as the answer.

`--resume "$SID"` continues the session and `git diff main | claude -p …` reviews a diff (a real pipe supplies EOF, so drop `< /dev/null` — and `set -o pipefail`, or a failing `git diff` silently becomes empty stdin). Both need the same gate; the runnable forms are in [`claude-subagent/SKILL.md`](./claude-subagent/SKILL.md).

A `claude -p` run can fail while still exiting 0 — check `.is_error` (auth failures land in `.result` as text) and `.permission_denials` (a tool missing from `--allowedTools` is denied silently). See [`claude-subagent/SKILL.md`](./claude-subagent/SKILL.md).

## Models

### Antigravity

Run `agy models` for the exact, currently-installed strings (pass them verbatim to `--model`):

| `--model` value | When to use |
|-----------------|-------------|
| `"Gemini 3.1 Pro (High)"` | **Default for every task.** |
| `"Gemini 3.1 Pro (Low)"` | Pro quality, less reasoning budget. |
| `"Gemini 3.5 Flash (Medium\|Low\|High)"` | Only when the user explicitly requests Flash/speed. |
| `"Claude Sonnet 4.6 (Thinking)"`, `"Claude Opus 4.6 (Thinking)"` | Only when the user explicitly asks for Claude. |
| `"GPT-OSS 120B (Medium)"` | Only when the user explicitly asks for GPT-OSS. |

### Gemini

| Flag | Model | When to use |
|------|-------|-------------|
| `-m pro` | `gemini-3.1-pro-preview` | **Default for every task.** |
| `-m flash` | `gemini-3-flash-preview` | Only when the user explicitly requests flash/speed. |

### Codex

| Model | When to use |
|-------|-------------|
| `gpt-5.6-sol` | **Default for every task.** (Flagship 5.6 tier; pass the explicit id, never the bare `gpt-5.6` alias.) |
| `gpt-5.6-terra`, `gpt-5.6-luna` | Cheaper / fastest 5.6 tiers — only when the user explicitly names one. |
| `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex-spark`, `gpt-5.3-codex` | Only when the user explicitly names one. |

Reasoning effort defaults to `high`; valid values are `xhigh`, `high`, `medium`, `low`, `minimal`. Override only on explicit user request.

### Claude

| `--model` value | When to use |
|-----------------|-------------|
| `opus` | **Default for every task.** |
| `sonnet` | Balanced. Only when the user asks — for bulk or mechanical work, propose it and let them decide. |
| `haiku` | Cheapest/fastest. Only when the user asks. |
| `fable` | Only when the user explicitly asks for it. |

Aliases resolve to the latest model in each family; full ids (`claude-opus-4-8`, `claude-sonnet-5`, `claude-haiku-4-5`) also work.

## Gemini key patterns

For Codex-specific patterns (sandbox modes, resume semantics, when to push back on Codex output), see [`codex-subagent/SKILL.md`](./codex-subagent/SKILL.md).

### Get clean output (final answer only)

```bash
gemini -m flash -p "TASK" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

`--output-format json` separates the final answer from intermediate tool-call narrations. `.response` contains only the model's conclusion.

### Parallel execution

Run multiple Gemini instances by sending multiple Bash tool calls in a single response:

```bash
# Call 1 (runs concurrently with Call 2):
gemini -m flash -p "TASK 1" --yolo --output-format json 2>/dev/null | jq -r '.response'

# Call 2 (runs concurrently with Call 1):
gemini -m flash -p "TASK 2" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

### Background execution with tmux (when explicitly requested)

```bash
SESSION="gemini-$(date +%s)"
LOG="/tmp/${SESSION}.log"
DONE_MARKER="${LOG}.done"

tmux new-session -d -s "$SESSION"
tmux send-keys -t "$SESSION" \
  "gemini -m pro -p 'TASK' --yolo --output-format json > '$LOG' 2>/dev/null; touch '$DONE_MARKER'" C-m

# Claude continues working here...

# Check and collect result later
[ -f "$DONE_MARKER" ] && jq -r '.response' "$LOG"
tmux kill-session -t "$SESSION" 2>/dev/null
rm -f "$LOG" "$DONE_MARKER"
```

### Pass files with `@` syntax

```bash
# No --yolo needed for read-only analysis
gemini -m pro -p "@src/ @docs/ Summarize the project architecture"
```

### Multi-turn sessions

```bash
RESULT=$(gemini -m flash -p "initial task" --yolo --output-format json 2>/dev/null)
echo "$RESULT" | jq -r '.response'
SESSION_ID=$(echo "$RESULT" | jq -r '.session_id')

gemini --resume "$SESSION_ID" -p "follow-up" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

## Tested on

- Antigravity CLI (`agy`) v1.0.10
- Gemini CLI v0.41.1
- Codex CLI v0.130.0
- Claude Code CLI v2.1.209 (`claude -p`, as both host and subagent)
- Claude Code (Sonnet 4.6, Opus 4.7, Opus 4.8)
