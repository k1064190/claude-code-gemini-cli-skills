---
name: antigravity-subagent
description: Delegate tasks to Google Antigravity CLI (`agy`) running as a subagent. Use this skill whenever the user says "use antigravity", "ask agy", "run this with antigravity", "delegate to agy", wants a second opinion from a different AI model, needs parallel AI execution, or wants to offload an agentic task to a separate process. Antigravity CLI is Google's successor to the Gemini CLI (Gemini CLI's free tier was retired 2026-06-18); it exposes Gemini 3.x, Claude Sonnet/Opus 4.6, and GPT-OSS models behind the single `agy` command. Also trigger when the codebase or files are too large for Claude's context window and Antigravity's large context is needed for full codebase analysis.
---

# Antigravity Subagent

Spawn and communicate with a Google Antigravity CLI (`agy`) agent to execute tasks independently. `agy` is the successor to `gemini` — it has its own agentic tool suite and runs as a separate process.

> **Binary is `agy`, not `antigravity`.** Always script with `agy`. It installs to `~/.local/bin/agy`; if a fresh shell reports `command not found`, run `agy install` (or ensure `~/.local/bin` is on PATH) and open a new terminal.

---

## Avoiding hangs and silent failures (read first)

Two failure modes make an `agy -p` call look like it "never responds / freezes." Both are avoidable:

1. **Redirect stdin on every `-p` call: `< /dev/null`.** Like `codex exec`, `agy -p` reads stdin whenever stdin is not a TTY. If the harness hands the process an open pipe that never delivers bytes or EOF, `agy` blocks **forever** and prints nothing. Verified: `sleep 100 | agy -p "hi"` hangs until killed; the same command with `< /dev/null` returns immediately. **Append `< /dev/null` to every `-p` invocation in this guide** — initial calls, `@`-syntax calls, and `-c`/`--conversation` resumes alike. (The tmux mode runs inside a pty, so its stdin is already a TTY and does not need this.)
2. **Don't black-hole stderr with `2>/dev/null`.** Capture it: `ERRLOG=$(mktemp)` then `2>"$ERRLOG"`. On a clean run the log is just startup noise you ignore; on a non-zero exit or empty stdout, `cat "$ERRLOG"` reveals the real cause (not authenticated, model string typo, timeout). Silently discarding stderr is what turns a plain failure into a mystery "freeze."
   > **Standing convention:** every `$ERRLOG` in the snippets below assumes you ran `ERRLOG=$(mktemp)` once at the start of your shell session. A `2>"$ERRLOG"` with `ERRLOG` unset fails with an *empty-filename* error and the command never runs — so if you copy a single snippet in isolation, define `ERRLOG` first (or substitute `2>/dev/null`).
3. **`--print-timeout` defaults to 5m and cuts off silently.** For long tasks raise it (`--print-timeout 15m`); also keep an outer `timeout -k 10 <N>` wall-clock cap — the `<N>` sends SIGTERM to `agy` and `-k 10` follows with SIGKILL 10 s later for a process that ignores SIGTERM. Treat **exit 124 as "timed out"** and **137 as "force-killed."** ⚠️ The cap bounds `agy` itself, but `timeout` does **not** kill agy's grandchildren — so this only guarantees the *caller* returns promptly when you capture output to a file (point 4). If you capture with `RESULT=$(agy …)` instead, a lingering agy child can hold the pipe open and defeat the timeout (that is the real "review took 40 minutes" hang). Always use the file-redirect form below.

4. **Capture agy's stdout to a FILE, not with `RESULT=$(agy …)`.** `agy` is agentic: it can spawn child processes (e.g. a `find` to locate a file — see the file caveat below). A command substitution `$(…)` doesn't return until **every** process holding the pipe's write end closes it, and `timeout` kills only its direct child (`agy`), not agy's grandchildren. So an orphaned agy child keeps `$(…)` blocked for as long as *it* runs — the outer `timeout` is defeated and the call appears to hang for many minutes. Redirect to a file instead (`> "$OUT"`); a file has no pipe to hold open, so control returns the instant `timeout` fires. Verified: with a lingering grandchild, `RESULT=$(timeout 3 …)` blocked 30s; `timeout 3 … > file` returned at 3s.

Canonical safe invocation (use this shape everywhere below):

```bash
OUT=$(mktemp); ERRLOG=$(mktemp)
timeout -k 10 600 agy --model "Gemini 3.1 Pro (High)" -p "TASK PROMPT" < /dev/null > "$OUT" 2>"$ERRLOG"
rc=$?   # not `status`: that name is read-only in fish/zsh
RESULT=$(cat "$OUT")   # read from the file — safe; a lingering agy child can't block this
if [ "$rc" -ne 0 ] || [ -z "$RESULT" ]; then echo "agy failed (exit $rc):"; cat "$ERRLOG"; fi
rm -f "$OUT" "$ERRLOG"
```

> **Don't make agy hunt for files — inline the content or use a cwd-relative `@path`.** If you reference a file agy can't resolve relative to its working directory, its agent may launch an expensive filesystem search to find it by name (observed: `agy -p "@some.diff …"` run from the wrong directory spawned `find /home/… -name some.diff`, which ran for ~40 min and — via the pipe trap above — hung the caller). For small inputs like a diff, paste the content straight into the prompt; for `@` inclusion, `cd` into the file's directory first and use a path that resolves from there.

> **Edits land in a scratch sandbox unless you pass `--dangerously-skip-permissions`.** In `-p` mode without it, `agy` runs agentic file writes in an isolated workspace (`~/.gemini/antigravity-cli/scratch/…`), **not** your project — so the user's files appear unchanged even though `agy` reported success. For any task that must modify real project files, pass `--dangerously-skip-permissions` (and `--add-dir <DIR>` for paths outside the cwd). Read-only analysis via `@` syntax does not need it.

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

Unlike the old Gemini CLI, **`agy --print` writes the final answer as plain text to stdout** — there is no `--output-format json` flag and no `jq` step. Capture stdout to a **file**, then read it (not via `$(agy …)` — see the canonical block for why):

```bash
OUT=$(mktemp); ERRLOG=$(mktemp)
timeout -k 10 600 agy --model "Gemini 3.1 Pro (High)" -p "TASK PROMPT" < /dev/null > "$OUT" 2>"$ERRLOG"
RESULT=$(cat "$OUT"); rm -f "$OUT" "$ERRLOG"
```

> `< /dev/null` prevents the stdin hang and `> "$OUT"` prevents a lingering agy child from blocking the caller (both explained in [Avoiding hangs](#avoiding-hangs-and-silent-failures-read-first)); `2>"$ERRLOG"` keeps startup/log noise off stdout while preserving the error text for when something goes wrong.

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
agy --model "Gemini 3.1 Pro (High)" -p "TASK PROMPT" < /dev/null 2>"$ERRLOG"

# Agentic: let agy run tools (write files, run shell) IN YOUR REAL WORKSPACE.
# Without --dangerously-skip-permissions the edits go to a scratch sandbox, not your project.
agy --model "Gemini 3.1 Pro (High)" -p "TASK PROMPT" --dangerously-skip-permissions < /dev/null 2>"$ERRLOG"
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
  --dangerously-skip-permissions < /dev/null 2>"$ERRLOG"
```

---

## Passing files and directories with `@` syntax

`agy`'s `@` syntax inlines file content into the prompt at construction time — no agentic tool call, so `--dangerously-skip-permissions` is **not** required for read-only inclusion. Paths are relative to the invocation directory.

```bash
# Single file
agy --model "Gemini 3.1 Pro (High)" -p "@src/main.py Explain this file's purpose and structure" < /dev/null 2>"$ERRLOG"

# Multiple files
agy --model "Gemini 3.1 Pro (High)" -p "@package.json @src/index.js Analyze the dependencies used" < /dev/null 2>"$ERRLOG"

# Entire directory
agy --model "Gemini 3.1 Pro (High)" -p "@src/ Summarize the architecture of this codebase" < /dev/null 2>"$ERRLOG"

# Whole project
agy --model "Gemini 3.1 Pro (High)" -p "@./ Give me an overview of this entire project" < /dev/null 2>"$ERRLOG"
```

### Large codebase analysis

Antigravity's large context can absorb whole codebases that overflow Claude's context. Use when files total more than ~100KB, or for project-wide pattern/security checks.

```bash
cd /path/to/project
agy --model "Gemini 3.1 Pro (High)" -p "@src/ @lib/ Has dark mode been implemented? Show relevant files and functions" < /dev/null 2>"$ERRLOG"
agy --model "Gemini 3.1 Pro (High)" -p "@src/ @api/ Are SQL injection protections in place? Show how user inputs are sanitized" < /dev/null 2>"$ERRLOG"
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
agy --model "Gemini 3.1 Pro (High)" -p "initial task" --dangerously-skip-permissions < /dev/null 2>"$ERRLOG"

# Follow-up on the MOST RECENT conversation:
agy -c -p "follow-up question" --dangerously-skip-permissions < /dev/null 2>"$ERRLOG"

# Resume a SPECIFIC conversation by ID:
agy --conversation <ID> -p "follow-up question" --dangerously-skip-permissions < /dev/null 2>"$ERRLOG"
```

> When resuming with `-c`/`--conversation`, the model and settings from the original conversation carry over — don't re-pass `--model` unless the user explicitly wants to switch.

---

## Error handling and timeouts

```bash
# Hard wall-clock cap (124 = SIGTERM/timed out, 137 = SIGKILL after -k grace). Also tune agy's own wait with --print-timeout.
timeout -k 10 180 agy --model "Gemini 3.1 Pro (High)" -p "TASK" --print-timeout 150s < /dev/null 2>"$ERRLOG"

# Watchdog for tmux sessions
sleep 180 && tmux kill-session -t "$SESSION" 2>/dev/null &
WATCHDOG_PID=$!
# ... wait for DONE ...
kill $WATCHDOG_PID 2>/dev/null
```

Common failure causes:
- **Command hangs / no output at all** → stdin was left unmanaged. Add `< /dev/null` (see [Avoiding hangs](#avoiding-hangs-and-silent-failures-read-first)). This is the single most common cause of "agy froze."
- **`agy` reported success but the project files are unchanged** → the run wrote to the scratch sandbox because `--dangerously-skip-permissions` was missing. Re-run with it (and `--add-dir` if needed).
- **Exit 124 (SIGTERM) or 137 (SIGKILL after `-k` grace)** → the outer `timeout` fired (or `--print-timeout` is too low for the task). Raise the cap; don't blindly retry. This is the guarantee that a slow run ends in minutes, not hours.
- **Empty stdout, non-zero exit** → `cat "$ERRLOG"` for the real error (auth, bad model string, network).
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

Send multiple Bash tool calls in a **single** response. Each runs independently and returns its own stdout — no tmux, no polling. Because each call is a **separate shell**, it cannot share an `$ERRLOG` defined elsewhere; define the temp log *inside each command* (otherwise `2>"$ERRLOG"` fails on an unset variable before `agy` even starts).

```bash
# Bash call 1 (runs simultaneously with call 2) — self-contained:
ERRLOG=$(mktemp); agy --model "Gemini 3.1 Pro (High)" -p "TASK 1" < /dev/null 2>"$ERRLOG"; [ $? -ne 0 ] && cat "$ERRLOG"; rm -f "$ERRLOG"

# Bash call 2 (runs simultaneously with call 1) — self-contained:
ERRLOG=$(mktemp); agy --model "Gemini 3.1 Pro (High)" -p "TASK 2" < /dev/null 2>"$ERRLOG"; [ $? -ne 0 ] && cat "$ERRLOG"; rm -f "$ERRLOG"
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
