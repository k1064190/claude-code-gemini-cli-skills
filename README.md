# claude-code-gemini-cli-skills

Claude Code skills for delegating tasks to external CLI agents — Google Antigravity (`agy`), Google Gemini, and OpenAI Codex — running as subagents.

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

## Defaults

These skills pin a single default model per agent and switch only when the user explicitly asks for a different one. This keeps behavior predictable across sessions:

- **Antigravity**: always invoke with `--model "Gemini 3.1 Pro (High)"`. Switch to a Flash, Claude, or GPT-OSS model only when the user explicitly requests it. Run `agy models` to see the exact strings available.
- **Gemini**: always invoke with `-m pro` (`gemini-3.1-pro-preview`). Switch to `-m flash` only when the user explicitly requests speed/flash.
- **Codex**: always invoke with model `gpt-5.6-sol` and reasoning effort `high`. Switch only when the user explicitly names a different model or effort (e.g., "use gpt-5.6-luna", "set effort to xhigh"). Always pass the explicit tier id — the bare `gpt-5.6` alias is rejected on ChatGPT-account auth.

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

## Installation

Copy the skill folders into your Claude Code skills directory:

```bash
cp -r antigravity-subagent ~/.claude/skills/
cp -r gemini-subagent      ~/.claude/skills/
cp -r codex-subagent       ~/.claude/skills/
```

Or symlink them:

```bash
ln -s $(pwd)/antigravity-subagent ~/.claude/skills/antigravity-subagent
ln -s $(pwd)/gemini-subagent      ~/.claude/skills/gemini-subagent
ln -s $(pwd)/codex-subagent       ~/.claude/skills/codex-subagent
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
- Claude Code (Sonnet 4.6, Opus 4.7, Opus 4.8)
