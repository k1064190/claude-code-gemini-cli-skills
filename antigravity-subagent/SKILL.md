---
name: antigravity-subagent
description: Delegate tasks to Google Antigravity CLI (`agy`) running as a subagent. Use this skill whenever the user says "use antigravity", "ask agy", "run this with antigravity", "delegate to agy", wants a second opinion from a different AI model, needs parallel AI execution, or wants to offload an agentic task to a separate process. Antigravity CLI is Google's successor to the Gemini CLI (Gemini CLI's free tier was retired 2026-06-18); it exposes Gemini 3.x, Claude Sonnet/Opus 4.6, and GPT-OSS models behind the single `agy` command. Also trigger when the codebase or files are too large for Claude's context window and Antigravity's large context is needed for full codebase analysis.
---

# Antigravity Subagent

Spawn and communicate with a Google Antigravity CLI (`agy`) agent to execute tasks independently. `agy` is the successor to `gemini` — it has its own agentic tool suite and runs as a separate process.

> **Binary is `agy`, not `antigravity`.** Always script with `agy`. It installs to `~/.local/bin/agy`; if a fresh shell reports `command not found`, run `agy install` (or ensure `~/.local/bin` is on PATH) and open a new terminal.

---

## Choosing a model

`agy models` lists the exact, currently-installed model strings. Pass them **verbatim** (with the parenthetical suffix, quoted) to `--model`.

**Default: always invoke with `--model "Gemini 3.1 Pro (High)"`.** Pass it on every call unless the user explicitly asks for a different model — e.g., "use flash", "use Claude", "switch to Opus". Do not silently downgrade to a Flash model to save time or tokens; quality is the default.

| `--model` value | When to use |
|-----------------|-------------|
| `"Gemini 3.1 Pro (High)"` | **Default for every task.** |
| `"Gemini 3.1 Pro (Low)"` | Pro quality, less reasoning budget. |
| `"Gemini 3.5 Flash (Medium)"` / `"... (Low)"` / `"... (High)"` | Only when the user explicitly requests Flash/speed. |
| `"Claude Sonnet 4.6 (Thinking)"` | Only when the user explicitly asks for Claude. |
| `"Claude Opus 4.6 (Thinking)"` | Only when the user explicitly asks for Opus. |
| `"GPT-OSS 120B (Medium)"` | Only when the user explicitly asks for GPT-OSS. |

> Run `agy models` first if unsure — the available strings can change between releases. Quote the value exactly; the spaces and parentheses are part of the name.

---

## Output format

Unlike the old Gemini CLI, **`agy --print` writes the final answer as plain text to stdout** — there is no `--output-format json` flag and no `jq` step. Capture stdout directly.

```bash
RESULT=$(agy --model "Gemini 3.1 Pro (High)" -p "TASK PROMPT" 2>/dev/null)
```

> `2>/dev/null` suppresses startup/log noise on stderr, leaving only the model's answer on stdout.

---

## Choosing execution mode

**Default: direct bash execution.** Run `agy -p` directly in the Bash tool. Simple, reliable, output is immediately available.

**Direct headless (`-p` / `--print`)** — use for all normal tasks:
- Q&A, summarization, code generation
- File reads/writes, shell commands (with `--dangerously-skip-permissions`)
- Any task where you wait for the result before proceeding

**Parallel execution** — use multiple Bash tool calls in a single response. Each runs a separate `agy` command simultaneously; results come back independently — no polling or marker files needed.

**Tmux background session** — use ONLY when the user explicitly asks to run something in the background ("run this in the background", "don't wait for it").

---

## Mode 1: Direct headless execution (`-p` / `--print`)

```bash
# Read-only / analysis (no tool approvals needed for @-included files)
agy --model "Gemini 3.1 Pro (High)" -p "TASK PROMPT" 2>/dev/null

# Agentic: let agy run tools (write files, run shell). Auto-approve every tool request:
agy --model "Gemini 3.1 Pro (High)" -p "TASK PROMPT" --dangerously-skip-permissions 2>/dev/null
```

Flags that matter for subagent use:

| Flag | Purpose |
|------|---------|
| `-p`, `--print`, `--prompt` | Run one prompt non-interactively and print the response, then exit. |
| `--model "<exact name>"` | Model for this session. See the table above. |
| `--dangerously-skip-permissions` | Auto-approve all tool permission requests (the `--yolo` equivalent). Needed only when `agy` will write files or run shell commands. |
| `--sandbox` | Run with terminal restrictions enabled. |
| `--add-dir <DIR>` | Add a directory to the workspace (repeatable). |
| `--print-timeout <DUR>` | Timeout for print-mode wait (default `5m0s`). |
| `-c`, `--continue` | Continue the most recent conversation. |
| `--conversation <ID>` | Resume a previous conversation by ID. |
| `-i`, `--prompt-interactive` | Run an initial prompt, then stay interactive (NOT for headless use). |

> Only pass `--dangerously-skip-permissions` when the task genuinely needs `agy` to mutate files or run commands. Pure analysis via `@` syntax (below) needs no approvals.

### Example

```bash
agy --model "Gemini 3.1 Pro (High)" \
  -p "Working directory: /home/cwh/project. Read src/main.py and write unit tests for all functions. Save tests to tests/test_main.py" \
  --dangerously-skip-permissions 2>/dev/null
```

---

## Passing files and directories with `@` syntax

`agy`'s `@` syntax inlines file content into the prompt at construction time — no agentic tool call, so `--dangerously-skip-permissions` is **not** required for read-only inclusion. Paths are relative to the invocation directory.

```bash
# Single file
agy --model "Gemini 3.1 Pro (High)" -p "@src/main.py Explain this file's purpose and structure" 2>/dev/null

# Multiple files
agy --model "Gemini 3.1 Pro (High)" -p "@package.json @src/index.js Analyze the dependencies used" 2>/dev/null

# Entire directory
agy --model "Gemini 3.1 Pro (High)" -p "@src/ Summarize the architecture of this codebase" 2>/dev/null

# Whole project
agy --model "Gemini 3.1 Pro (High)" -p "@./ Give me an overview of this entire project" 2>/dev/null
```

### Large codebase analysis

Antigravity's large context can absorb whole codebases that overflow Claude's context. Use when files total more than ~100KB, or for project-wide pattern/security checks.

```bash
cd /path/to/project
agy --model "Gemini 3.1 Pro (High)" -p "@src/ @lib/ Has dark mode been implemented? Show relevant files and functions" 2>/dev/null
agy --model "Gemini 3.1 Pro (High)" -p "@src/ @api/ Are SQL injection protections in place? Show how user inputs are sanitized" 2>/dev/null
```

You can also widen the agent's writable workspace with `--add-dir` (repeatable) for agentic, multi-directory tasks.

---

## Mode 2: Tmux background session (only when explicitly requested)

Use this mode **only** when the user explicitly asks to run `agy` in the background.

### Start the session

```bash
SESSION="agy-$(date +%s)"
LOG="/tmp/${SESSION}.log"
DONE_MARKER="${LOG}.done"

tmux new-session -d -s "$SESSION"
tmux send-keys -t "$SESSION" \
  "agy --model 'Gemini 3.1 Pro (High)' -p 'TASK PROMPT' --dangerously-skip-permissions > '$LOG' 2>/dev/null; touch '$DONE_MARKER'" \
  C-m

echo "Session: $SESSION"
echo "Log: $LOG"
```

> Use a separate `.done` marker file rather than appending a sentinel to the log, so the captured answer stays clean.

### Monitor and collect

```bash
# NEVER put echo or any output inside the while loop — silent wait only.
while [ ! -f "$DONE_MARKER" ]; do sleep 3; done
echo "DONE"
cat "$LOG"
tmux kill-session -t "$SESSION" 2>/dev/null
rm -f "$LOG" "$DONE_MARKER"
```

---

## Session resumption (multi-turn)

`agy` print mode is stateless per call, but you can chain turns onto a prior conversation:

```bash
# First call (starts a conversation):
agy --model "Gemini 3.1 Pro (High)" -p "initial task" --dangerously-skip-permissions 2>/dev/null

# Follow-up on the MOST RECENT conversation:
agy -c -p "follow-up question" --dangerously-skip-permissions 2>/dev/null

# Resume a SPECIFIC conversation by ID:
agy --conversation <ID> -p "follow-up question" --dangerously-skip-permissions 2>/dev/null
```

> When resuming with `-c`/`--conversation`, the model and settings from the original conversation carry over — don't re-pass `--model` unless the user explicitly wants to switch.

---

## Error handling and timeouts

```bash
# Hard wall-clock cap (exit 124 = timed out). Also tune agy's own wait with --print-timeout.
timeout 180 agy --model "Gemini 3.1 Pro (High)" -p "TASK" --print-timeout 150s 2>/dev/null

# Watchdog for tmux sessions
sleep 180 && tmux kill-session -t "$SESSION" 2>/dev/null &
WATCHDOG_PID=$!
# ... wait for DONE ...
kill $WATCHDOG_PID 2>/dev/null
```

Common failure causes:
- `agy: command not found` in a fresh shell → `~/.local/bin` not on PATH. Run `agy install`, or add `~/.local/bin` to PATH, then open a new terminal.
- Calling `antigravity` instead of `agy` → the binary is `agy`.
- Not authenticated → run `agy` interactively once to sign in, then retry headless.
- Task too large for context → split into subtasks, or use `@` syntax instead of agentic file reading.

---

## Writing effective `agy` task prompts

`agy` is agentic — it decides which tools to call. Write prompts as high-level task descriptions.

**Include:**
- Working directory (absolute path) when using agentic mode
- What files to read or create
- Expected output/deliverable
- Any constraints (language, format, style)

**Prompt template:**
```
Working directory: {abs_path}
Task: {clear description of what to produce}
Output: {where to save results, or "respond in plain text"}
Constraints: {any specific requirements}
```

---

## Parallel execution patterns

### Multiple `agy` agents in parallel

Send multiple Bash tool calls in a **single** response. Each runs independently and returns its own stdout — no tmux, no polling.

```bash
# Bash call 1 (runs simultaneously with call 2):
agy --model "Gemini 3.1 Pro (High)" -p "TASK 1" 2>/dev/null

# Bash call 2 (runs simultaneously with call 1):
agy --model "Gemini 3.1 Pro (High)" -p "TASK 2" 2>/dev/null
```

### Claude + `agy` in parallel

Start `agy` in one Bash call while Claude does its own work (reading files, editing code) in other tool calls within the same message. All tool calls in a single response execute concurrently.

---

## Migrating from the Gemini CLI

The Gemini CLI free tier was retired on 2026-06-18; `agy` is its successor. Import existing Gemini CLI configuration (settings, extensions as plugins) non-destructively — the original `~/.gemini/` is preserved:

```bash
agy plugin import gemini   # one-time migration
agy models                 # confirm the model strings available to you
agy plugin list            # see imported/installed plugins
```

---

## When to use `agy` vs Claude

| Use `agy` | Use Claude |
|-----------|------------|
| Codebase too large for Claude's context | Task fits within current context |
| Want a second opinion from a different model (Gemini/Claude/GPT-OSS) | Task requires this conversation's history |
| Long background task while Claude continues | Task is quick (<30s) — delegation overhead not worth it |
| Parallel workload to split execution | User explicitly wants Claude to handle it |
