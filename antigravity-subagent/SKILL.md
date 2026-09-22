---
name: antigravity-subagent
description: Use when the user explicitly asks to delegate a general analysis, coding, research, large-context, parallel, or second-opinion task to Google Antigravity CLI (`agy`). Do not use for writing-only requests handled by gemini-writing-subagent.
---

# Antigravity Subagent

Delegate a bounded task to Google Antigravity CLI and evaluate its result before using it. The binary is `agy`, not `antigravity`.

## Route writing elsewhere

Use `gemini-writing-subagent` for drafting, rewriting, proofreading, tone changes, or translation with Gemini 3.8 Flash. Keep this skill for general agentic delegation.

## Non-negotiable execution contract

- Run `agy models` when availability is uncertain and pass the exact model slug.
- Default to `gemini-3.1-pro-high`. Change models only when the user asks or the task explicitly requires a different family.
- Set both an `agy --print-timeout` and a slightly longer outer `timeout -k` bound. Installed `agy 1.2.7` and the online documentation disagree about the default, so never rely on it.
- Close stdin with `< /dev/null` for `-p` calls. An open non-TTY stdin can make the command wait indefinitely.
- Redirect stdout and stderr to files. Do not wrap `agy` itself in command substitution: a grandchild that inherits the stdout pipe can keep the caller blocked after `agy` is killed.
- Use JSON output and require both process exit `0` and result status `SUCCESS`. A plausible `response` from any other status is not a successful result.
- Treat a permission or tool soft-denial in stderr as an incomplete run, even if the process exits `0`.
- Never retry with `--dangerously-skip-permissions`, alter persistent permission settings, switch models, or change the prompt without fresh user authorization.

## Canonical headless invocation

Prepare the prompt in a UTF-8 file so user content remains shell data. Then run:

```bash
PROMPT_FILE=/absolute/path/to/prompt.txt
OUT=$(mktemp); ERRLOG=$(mktemp)

timeout -k 10 610 agy \
  --model gemini-3.1-pro-high \
  --sandbox \
  --output-format json \
  --print-timeout 10m \
  -p "$(cat "$PROMPT_FILE")" \
  < /dev/null > "$OUT" 2> "$ERRLOG"
rc=$?

if [ "$rc" -ne 0 ] || ! jq -e \
  '.status == "SUCCESS" and (.response | type == "string" and length > 0)' \
  "$OUT" >/dev/null 2>&1; then
  echo "agy failed (exit $rc)" >&2
  cat "$ERRLOG" >&2
  jq -r '.error // empty' "$OUT" 2>/dev/null >&2
  rm -f "$OUT" "$ERRLOG"
  exit 1
fi

if rg -qi 'soft[- ]denied|permission denied|requires approval|not allowed by policy' "$ERRLOG"; then
  echo "agy result rejected because a tool or permission was denied" >&2
  cat "$ERRLOG" >&2
  rm -f "$OUT" "$ERRLOG"
  exit 1
fi

jq -r '.response' "$OUT"
rm -f "$OUT" "$ERRLOG"
```

Reading `OUT` with `jq` after the process returns is safe; the forbidden pattern is `RESULT=$(agy ...)`.

## Model selection

Use slugs reported by `agy models`:

| Model slug | Use |
|---|---|
| `gemini-3.1-pro-high` | Default for complex analysis, coding, research, and final review |
| `gemini-3.1-pro-low` | Pro with a smaller reasoning budget, only when requested |
| `gemini-3.8-flash-high` | Fast high-effort work, only when requested; writing routes to `gemini-writing-subagent` |
| `gemini-3.8-flash-medium` | Faster/lower-effort Flash work, only when requested |
| `gemini-3.8-flash-low` | Lightweight Flash work, only when requested |
| `claude-sonnet-4-6` | Only when the user explicitly asks for Sonnet through `agy` |
| `claude-opus-4-6-thinking` | Only when the user explicitly asks for Opus through `agy` |
| `gpt-oss-120b-medium` | Only when the user explicitly asks for GPT-OSS |

Do not silently substitute another model when the selected slug is unavailable.

## Files, tools, and permissions

- `--sandbox` restricts terminal execution; it is not a substitute for user authorization.
- Headless permission behavior is influenced by user settings. Reads and writes inside the active workspace may be allowed, while commands that require interactive approval can be soft-denied.
- For read-only context, prefer cwd-relative `@path` inclusion and run from the directory that contains the path. Do not make `agy` search the filesystem for a missing file.
- For an authorized edit, run from the exact project directory, name the allowed files, and use `--mode accept-edits`. Inspect the resulting diff yourself.
- Do not pass `--dangerously-skip-permissions` by default. If scoped permissions block required work, stop and report the exact denial.
- Use `--add-dir` only for a directory the user placed in scope.

Example read-only inclusion:

```bash
cd /absolute/path/to/project
OUT=$(mktemp); ERRLOG=$(mktemp)
agy --model gemini-3.1-pro-high --sandbox --output-format json \
  --print-timeout 10m -p '@src/ Review the architecture and cite relevant files.' \
  < /dev/null > "$OUT" 2> "$ERRLOG"
```

Apply the same exit, JSON-status, stderr-denial, and temporary-file cleanup steps as the canonical invocation.

## Follow-ups and background work

- `agy -c -p "..."` continues the most recent conversation; `--conversation <ID>` resumes a specific one. Preserve the original task's permissions and apply the same safety gates.
- Use tmux only when the user explicitly asks for background execution. Write stdout, stderr, and a completion marker to separate files; do not poll noisily.
- Parallel calls are separate processes. Give each call its own output and stderr files.

## Failure handling

Report and stop on authentication failure, unknown model, timeout, non-`SUCCESS` status, malformed JSON, empty response, or permission denial. Do not present diagnostics as the model's answer and do not retry automatically.
