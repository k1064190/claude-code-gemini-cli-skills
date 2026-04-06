# claude-code-gemini-cli-skills

Claude Code skills for delegating tasks to Google Gemini CLI as a subagent.

## Skills

### [`gemini-subagent`](./gemini-subagent/SKILL.md)

Delegate tasks from Claude to Gemini CLI running as an independent agent. Useful for:

- **Large codebase analysis** — Gemini's massive context window handles entire repos that overflow Claude's context
- **Parallel execution** — Run multiple Gemini instances concurrently via parallel Bash calls
- **Google Search grounding** — Leverage Gemini's built-in web search
- **Second opinion** — Compare results from two different AI models

## Requirements

- [Gemini CLI](https://github.com/google-gemini/gemini-cli) installed and authenticated
- `jq` for JSON parsing
- `tmux` (optional — only needed for explicit background execution)

```bash
# Install Gemini CLI
npm install -g @google/gemini-cli

# Authenticate
gemini
```

## Installation

Copy the skill folder into your Claude Code skills directory:

```bash
cp -r gemini-subagent ~/.claude/skills/
```

Or symlink it:

```bash
ln -s $(pwd)/gemini-subagent ~/.claude/skills/gemini-subagent
```

## Quick start

```bash
# Simple query (flash model)
gemini -m flash -p "What is the capital of France?" --output-format json 2>/dev/null | jq -r '.response'

# Complex task (pro model)
gemini -m pro -p "Working directory: /path/to/project. Analyze the architecture and suggest improvements." --yolo --output-format json 2>/dev/null | jq -r '.response'

# Large codebase analysis (@ syntax, no --yolo needed)
cd /path/to/project
gemini -m pro -p "@src/ @tests/ Explain the overall architecture and identify any missing test coverage"
```

## Models

| Flag | Model | Best for |
|------|-------|----------|
| `-m flash` | `gemini-3-flash-preview` | Speed — Q&A, summaries, quick codegen |
| `-m pro` | `gemini-3.1-pro-preview` | Quality — reasoning, architecture, complex analysis |
| *(omit)* | `auto-gemini-3` | Auto-routing based on task complexity |

## Key patterns

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

- Gemini CLI v0.34.0
- Claude Code (Sonnet 4.6)
