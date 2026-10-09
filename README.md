# claude-code-gemini-cli-skills

Cross-runtime Agent Skills for delegating tasks to CLI agents — Google Antigravity (`agy`), Google Gemini, and Claude Code itself (`claude -p`) — running as subagents. The standard `SKILL.md` folders work with Codex and Claude Code.

> **Heads up:** Google is retiring the Gemini CLI (free tier ended 2026-06-18) in favor of the **Antigravity CLI** (`agy`). New work should prefer [`antigravity-subagent`](./antigravity-subagent/SKILL.md); `gemini-subagent` is kept for environments still on the old `gemini` binary.

## Skills

### [`antigravity-subagent`](./antigravity-subagent/SKILL.md)

Delegate tasks from Claude to Google Antigravity CLI (`agy`), the Gemini CLI's successor, running as an independent agent. Useful for:

- **Large codebase analysis** — Antigravity's large context window handles entire repos that overflow Claude's context
- **Multi-model second opinion** — One CLI fronts Gemini 3.x, Claude Sonnet/Opus 4.6, and GPT-OSS
- **Parallel execution** — Run multiple `agy` instances concurrently via parallel Bash calls
- **Checked structured output** — JSON status, process exit, and permission diagnostics are all verified before accepting a response

### [`gemini-writing-subagent`](./gemini-writing-subagent/SKILL.md)

Use Gemini 3.8 Flash only for user-facing writing:

- **Drafting and copy** — prose, email, summaries, outlines, titles
- **Editing** — rewriting, proofreading, tone adjustment, copyediting
- **Translation** — faithful translation that preserves terminology and formatting
- **Isolation** — temporary working directory, sandboxed CLI, no project access or permission bypass

### [`gemini-subagent`](./gemini-subagent/SKILL.md)

Delegate tasks from Claude to Gemini CLI running as an independent agent. Useful for:

- **Large codebase analysis** — Gemini's massive context window handles entire repos that overflow Claude's context
- **Parallel execution** — Run multiple Gemini instances concurrently via parallel Bash calls
- **Google Search grounding** — Leverage Gemini's built-in web search
- **Second opinion** — Compare results from two different AI models

### [`claude-subagent`](./claude-subagent/SKILL.md)

Delegate tasks to a fresh Claude Code CLI run (`claude -p`). Useful for:

- **Clean context** — A cold instance reads the code without the current conversation's assumptions
- **Scoped permissions** — `--tools` bounds which tools exist, `--permission-mode dontAsk` denies the rest, and `--setting-sources user` drops the target repo's own settings and hooks, which otherwise run shell commands outside the tool boundary
- **Parallel execution** — Run several `claude -p` instances concurrently via parallel Bash calls
- **Orchestration from other CLIs** — Codex or Antigravity can call Claude for a subtask

Not a second opinion: the subagent is the same model family, so it shares your blind spots. Use `antigravity-subagent` (or, inside Claude Code, the official Codex plugin's `/codex:*` commands) for a genuine cross-model check.

## Defaults

These skills pin a single default model per agent and switch only when the user explicitly asks for a different one. This keeps behavior predictable across sessions:

- **Antigravity**: use `--model gemini-3.1-pro-high`. Switch to a Flash, Claude, or GPT-OSS slug only when the user explicitly requests it. Run `agy models` to see the exact current slugs.
- **Gemini writing**: use `gemini-3.8-flash-high`; only the High, Medium, or Low variants of Gemini 3.8 Flash are accepted, with no silent fallback.
- **Gemini**: always invoke with `-m pro` (`gemini-3.1-pro-preview`). Switch to `-m flash` only when the user explicitly requests speed/flash.
- **Claude**: always invoke with `--model opus` and **without `--bare`**. Bare mode never reads OAuth or the keychain, so on a subscription account it fails with `"Not logged in · Please run /login"`; it is safe only with a non-OAuth credential source (`ANTHROPIC_API_KEY`, an `apiKeyHelper` via `--settings`, or Bedrock / Google Cloud / Foundry credentials). Bound the tools with **`--tools` plus `--permission-mode dontAsk --setting-sources user`** — `--allowedTools` alone only suppresses prompts (under a permissive ambient mode an unlisted `Edit` or `Bash` just runs), and the target repo's own `.claude/settings.json` hooks execute shell commands outside the tool boundary unless the project settings are dropped. Switch to `sonnet`/`haiku` only when the user explicitly asks. Worth surfacing before a large fan-out: a trivial `opus` call still costs ~$0.40, since a non-bare run loads CLAUDE.md, plugins, and skills.

## Requirements

### Antigravity subagent

- [Antigravity CLI](https://antigravity.google) (`agy`) installed and authenticated
- `jq` for JSON status and response parsing
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

### Gemini writing subagent

- The Antigravity requirement above
- Python 3.10 or newer for the isolated runner

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

Copy the skill folders into `~/.agents/skills` for cross-runtime discovery, including Codex:

```bash
cp -r antigravity-subagent    ~/.agents/skills/
cp -r gemini-writing-subagent ~/.agents/skills/
cp -r gemini-subagent         ~/.agents/skills/
cp -r claude-subagent         ~/.agents/skills/
```

Or symlink them:

```bash
ln -s "$(pwd)/antigravity-subagent"    ~/.agents/skills/antigravity-subagent
ln -s "$(pwd)/gemini-writing-subagent" ~/.agents/skills/gemini-writing-subagent
ln -s "$(pwd)/gemini-subagent"         ~/.agents/skills/gemini-subagent
ln -s "$(pwd)/claude-subagent"         ~/.agents/skills/claude-subagent
```

Claude Code also discovers skills copied or linked under `~/.claude/skills`.

## Quick start

### Antigravity

```bash
# Default invocation — Gemini 3.1 Pro High, sandboxed, JSON result
OUT=$(mktemp); ERRLOG=$(mktemp)
timeout -k 10 610 agy --model gemini-3.1-pro-high --sandbox \
  --output-format json --print-timeout 10m \
  -p "Analyze the architecture and suggest improvements." \
  < /dev/null > "$OUT" 2> "$ERRLOG"
rc=$?
if [ "$rc" -ne 0 ] || ! jq -e \
  '.status == "SUCCESS" and (.response | type == "string" and length > 0)' \
  "$OUT" >/dev/null 2>&1; then
  cat "$ERRLOG" >&2
  rm -f "$OUT" "$ERRLOG"
  exit 1
fi
if rg -qi 'soft[- ]denied|permission denied|requires approval|not allowed by policy' "$ERRLOG"; then
  cat "$ERRLOG" >&2
  rm -f "$OUT" "$ERRLOG"
  exit 1
fi
jq -r '.response' "$OUT"
rm -f "$OUT" "$ERRLOG"

# Large codebase analysis with cwd-relative inclusion
cd /path/to/project
agy --model gemini-3.1-pro-high --sandbox --output-format json \
  --print-timeout 10m -p "@src/ @tests/ Explain the architecture" \
  < /dev/null > "$OUT" 2> "$ERRLOG"
```

See [`antigravity-subagent`](./antigravity-subagent/SKILL.md) for the complete exit/status/permission gates.

### Gemini writing

```bash
python3 gemini-writing-subagent/scripts/run_gemini_writing.py \
  --mode rewrite \
  --instructions-file /absolute/path/to/instructions.txt \
  --source /absolute/path/to/draft.md
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

Run `agy models` for the exact, currently-installed slugs (pass them verbatim to `--model`):

| `--model` value | When to use |
|-----------------|-------------|
| `gemini-3.1-pro-high` | **Default for general Antigravity delegation.** |
| `gemini-3.1-pro-low` | Pro quality, smaller reasoning budget; only when requested. |
| `gemini-3.8-flash-high` | Default for the writing-only skill; general use only when requested. |
| `gemini-3.8-flash-medium`, `gemini-3.8-flash-low` | Lower-effort 3.8 Flash; only when requested. |
| `gemini-3.7-flash-*`, `gemini-3.6-flash-*` | Older Flash generations; only when explicitly requested. |
| `claude-sonnet-4-6`, `claude-opus-4-6-thinking` | Only when the user explicitly asks for Claude. |
| `gpt-oss-120b-medium` | Only when the user explicitly asks for GPT-OSS. |

### Gemini

| Flag | Model | When to use |
|------|-------|-------------|
| `-m pro` | `gemini-3.1-pro-preview` | **Default for every task.** |
| `-m flash` | `gemini-3-flash-preview` | Only when the user explicitly requests flash/speed. |

### Claude

| `--model` value | When to use |
|-----------------|-------------|
| `opus` | **Default for every task.** |
| `sonnet` | Balanced. Only when the user asks — for bulk or mechanical work, propose it and let them decide. |
| `haiku` | Cheapest/fastest. Only when the user asks. |
| `fable` | Only when the user explicitly asks for it. |

Aliases resolve to the latest model in each family; full ids (`claude-opus-4-8`, `claude-sonnet-5`, `claude-haiku-4-5`) also work.

## Gemini key patterns

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

- Antigravity CLI (`agy`) v1.2.7
- Gemini CLI v0.41.1
- Claude Code CLI v2.1.209 (`claude -p`, as both host and subagent)
- Claude Code (Sonnet 4.6, Opus 4.7, Opus 4.8)
