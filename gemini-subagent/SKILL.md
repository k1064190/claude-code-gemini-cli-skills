---
name: gemini-subagent
description: Delegate tasks to Gemini CLI running as a subagent. Use this skill whenever the user says "use gemini", "ask gemini", "run this with gemini", "delegate to gemini", wants a second opinion from a different AI model, needs parallel AI execution, or wants to offload a long-running agentic task to run in the background while Claude continues working. Also trigger when the user wants to leverage Gemini's built-in Google Search, wants to compare results from two different AI models, or when the codebase or files are too large for Claude's context window and Gemini's massive context is needed for full codebase analysis.
---

# Gemini Subagent

Spawn and communicate with a Gemini CLI agent to execute tasks independently. Gemini has its own tool suite and runs as a separate process.

---

## Choosing a model

**Default: always invoke with `-m pro`** (resolves to `gemini-3.1-pro-preview`). Pass `-m pro` on every Gemini call unless the user explicitly asks for a different model — e.g., "use flash", "switch to flash", or names a specific model by flag/version. Do not silently downgrade to `flash` to save time or tokens; quality is the default.

| Flag | Resolves to | When to use |
|------|-------------|-------------|
| `-m pro` | `gemini-3.1-pro-preview` | **Default for every task.** |
| `-m flash` | `gemini-3-flash-preview` | Only when the user explicitly requests flash/speed. |

---

## Choosing execution mode

**Default: direct bash execution.** Run Gemini directly in the Bash tool. Simple, reliable, and the output is immediately available.

**Direct headless** — use for all normal tasks:
- Q&A, summarization, code generation
- File reads/writes, shell commands
- Any task where you wait for the result before proceeding

**Parallel execution** — use multiple Bash tool calls in a single response:
- Each Bash call runs a separate `gemini` command simultaneously
- Results come back independently — no polling or marker files needed

**Tmux background session** — use ONLY when the user explicitly asks to run something in the background:
- User says "run this in the background", "background task", "don't wait for it"
- Very long-running tasks where the user wants Claude to continue other work while Gemini runs

---

## Passing files and directories with `@` syntax

Gemini's `@` syntax is the fastest way to pass file content directly into the prompt context — no tool calls needed, so `--yolo` is not required for read-only tasks.

Paths are relative to the directory where you invoke `gemini`.

```bash
# Single file
gemini -p "@src/main.py Explain this file's purpose and structure"

# Multiple files
gemini -p "@package.json @src/index.js Analyze the dependencies used in the code"

# Entire directory
gemini -p "@src/ Summarize the architecture of this codebase"

# Multiple directories
gemini -p "@src/ @tests/ Analyze test coverage for the source code"

# Current directory (entire project)
gemini -p "@./ Give me an overview of this entire project"

# Or pass the current directory explicitly
gemini -p "@./ Analyze the project structure and dependencies"
```

> When using `@` syntax, Gemini reads files at prompt-construction time — no agentic tool calls occur for the inclusion itself. You only need `--yolo` if Gemini will also write files or run shell commands during the task.

### Large codebase analysis

Gemini's context window can handle entire codebases that would overflow Claude's context. Use this pattern when:
- Files total more than ~100KB
- You need project-wide pattern analysis
- You want to verify if a feature/pattern is implemented across the whole repo

```bash
# Check if a feature is implemented
cd /path/to/project
gemini -p "@src/ @lib/ Has dark mode been implemented? Show relevant files and functions"

# Verify security measures
gemini -p "@src/ @api/ Are SQL injection protections in place? Show how user inputs are sanitized"

# Understand architecture
gemini -p "@./ Explain the overall architecture, key modules, and data flow"
```

---

## Mode 1: Direct headless execution

**Always use `--output-format json | jq -r '.response'` for subagent use.** This extracts only the final answer — intermediate tool-call narrations ("I will read the file...") are excluded.

```bash
gemini -m flash -p "TASK PROMPT" --yolo --output-format json 2>/dev/null | jq -r '.response'
gemini -m pro -p "TASK PROMPT" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

> `--yolo` auto-approves all tool calls (file reads/writes, shell commands, web search).
> `2>/dev/null` removes MCP startup noise from stderr.
> `.response` in the JSON output contains only the model's final answer.

If you need the raw JSON (e.g., to also inspect token usage or session_id):
```bash
gemini -m flash -p "TASK PROMPT" --yolo --output-format json 2>/dev/null
# Fields: .response (final answer), .session_id, .stats (token usage, latency)
```

Pass file context via stdin:
```bash
cat /path/to/file | gemini -m flash -p "TASK PROMPT using the above file content" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

Pass working directory context explicitly in the prompt when using agentic mode:
```
"Working directory: /path/to/project. Task: ..."
```

### Example
```bash
gemini -m pro -p "Working directory: /home/cwh/project. Read src/main.py and write unit tests for all functions. Save tests to tests/test_main.py" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

---

## Mode 2: Tmux background session (only when explicitly requested)

Use this mode **only** when the user explicitly asks to run Gemini in the background.

### Start the session

```bash
SESSION="gemini-$(date +%s)"
LOG="/tmp/${SESSION}.log"
DONE_MARKER="${LOG}.done"

tmux new-session -d -s "$SESSION"
tmux send-keys -t "$SESSION" \
  "gemini -m flash -p 'TASK PROMPT' --yolo --output-format json > '$LOG' 2>/dev/null; touch '$DONE_MARKER'" \
  C-m

echo "Session: $SESSION"
echo "Log: $LOG"
```

> Use a separate `.done` marker file instead of appending a sentinel to the log. The JSON output may not end with a newline, so appending text to the same file corrupts the JSON.

### Monitor progress

```bash
tail -c 200 "$LOG"   # peek at recent output (JSON in progress)
[ -f "$DONE_MARKER" ] && echo "DONE" || echo "STILL RUNNING"
```

### Read result and cleanup

```bash
while [ ! -f "$DONE_MARKER" ]; do
  sleep 3
done

jq -r '.response' "$LOG"

tmux kill-session -t "$SESSION" 2>/dev/null
rm -f "$LOG" "$DONE_MARKER"
```

---

## Error handling and timeouts

```bash
# Hard timeout: kill after N seconds (exit code 124 = timed out)
timeout 120 gemini -m flash -p "TASK" --yolo --output-format json 2>/dev/null | jq -r '.response'

# Watchdog for tmux sessions
sleep 120 && tmux kill-session -t "$SESSION" 2>/dev/null &
WATCHDOG_PID=$!
# ... wait for DONE ...
kill $WATCHDOG_PID 2>/dev/null
```

Common failure causes:
- `settings.json` JSON parse error → fix the file before running
- No auth → run `gemini` interactively once to re-authenticate
- Task too large for context → split into subtasks, or use `@` syntax instead of agentic file reading

---

## Writing effective Gemini task prompts

Gemini is agentic — it decides which tools to call. Write prompts as high-level task descriptions.

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

## Gemini's built-in tools

These are available in every Gemini CLI installation without extra setup:

- `read_file`, `write_file`, `replace` — file operations
- `glob`, `list_directory`, `grep_search` — file discovery
- `run_shell_command` — shell execution
- `google_web_search`, `web_fetch` — web access
- `ask_user` — interactive clarification
- `save_memory` — persistent memory
- `enter_plan_mode` / `exit_plan_mode` — structured planning

### Optional tools (require MCP installation)

These are **not** built-in — they must be installed separately as MCP servers:

- `mcp_playwright_browser` — full browser automation (Playwright)
- `sequential_thinking` — structured multi-step reasoning

---

## Passing images and files

Gemini supports multimodal input. For analysis tasks, prefer `@` syntax. For tasks where Gemini should also process/transform files, use agentic mode with `--yolo`:

```bash
# Analysis only — no --yolo needed
gemini -p "@screenshots/ui.png Describe any UI issues you see"

# Agentic: read image and write a report
gemini -p "Working directory: /home/cwh. Read screenshots/ui.png and save a bug report to bugs/ui-report.md" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

---

## Session resumption (multi-turn)

```bash
# List available sessions
gemini --list-sessions 2>/dev/null

# Resume latest
gemini --resume latest -p "follow-up question" --yolo --output-format json 2>/dev/null | jq -r '.response'

# Resume by ID
gemini --resume {session_id} -p "follow-up" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

Save session ID for multi-call workflows — or read it from the JSON output directly:
```bash
# First call: save session_id from JSON output
RESULT=$(gemini -m flash -p "initial task" --yolo --output-format json 2>/dev/null)
echo "$RESULT" | jq -r '.response'
SESSION_ID=$(echo "$RESULT" | jq -r '.session_id')

# Follow-up reusing same session
gemini --resume "$SESSION_ID" -p "next question" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

---

## Parallel execution patterns

### Multiple Gemini agents in parallel

Use multiple Bash tool calls in a single response. Each call runs independently and returns its own result — no tmux, no polling, no marker files.

```
# Bash call 1 (runs simultaneously with call 2):
gemini -m flash -p "TASK 1" --yolo --output-format json 2>/dev/null | jq -r '.response'

# Bash call 2 (runs simultaneously with call 1):
gemini -m flash -p "TASK 2" --yolo --output-format json 2>/dev/null | jq -r '.response'
```

> Send both Bash tool calls in the **same message**. The harness runs them concurrently.

### Claude + Gemini in parallel

Start Gemini in one Bash call while Claude does its own work (e.g., reading files, editing code) in other tool calls within the same message. All tool calls in a single response execute concurrently.

### Background execution (only when user explicitly requests)

If the user says "run in the background" or "don't wait for it", use tmux:

```bash
SESSION="gemini-$(date +%s)"
LOG="/tmp/${SESSION}.log"
DONE_MARKER="${LOG}.done"
tmux new-session -d -s "$SESSION"
tmux send-keys -t "$SESSION" \
  "gemini -m flash -p 'TASK' --yolo --output-format json > '$LOG' 2>/dev/null; touch '$DONE_MARKER'" C-m
echo "Session: $SESSION | Log: $LOG"
```

Collect result later:

```bash
# NEVER put echo or any output inside the while loop — silent wait only.
while [ ! -f "$DONE_MARKER" ]; do sleep 3; done
echo "DONE"
jq -r '.response' "$LOG"
tmux kill-session -t "$SESSION" 2>/dev/null
rm -f "$LOG" "$DONE_MARKER"
```

---

## When to use Gemini vs Claude

| Use Gemini (`-m pro` or `-m flash`) | Use Claude |
|--------------------------------------|------------|
| Codebase too large for Claude's context | Task fits within current context |
| Need Google Search grounding | Task requires this conversation's history |
| Want a second opinion on code/analysis | Task is quick (<30s) — delegation overhead not worth it |
| Long background task while Claude continues | User explicitly wants Claude to handle it |
| Parallel workload to split execution | |
