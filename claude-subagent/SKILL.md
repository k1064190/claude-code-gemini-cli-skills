---
name: claude-subagent
description: Use when the user asks to run Claude Code CLI non-interactively (claude -p, "spawn a claude instance", "delegate to claude", "use a fresh claude context") or when an isolated Claude Code run is needed for parallel work, scoped tool permissions, or a clean-context second pass. Also use when a Codex or Gemini/Antigravity orchestrator needs Claude's capabilities for a subtask — this skill has the exact command patterns.
---

# Claude Subagent Skill Guide

Spawns a fresh Claude Code CLI run via `claude -p`. The subagent has **no knowledge of the current conversation** — the prompt is the only context it gets.

## Running a Task

1. **Default: model `opus`, no `--bare`, `--output-format json`, read-only tools.** Use these for every task and switch **only when the user explicitly asks** (e.g., "use sonnet", "let it edit files"). Model aliases: `opus`, `sonnet`, `haiku`, `fable`; full ids also work (`claude-opus-4-8`, `claude-sonnet-5`, `claude-haiku-4-5`). Cost is worth surfacing before a big fan-out — a *trivial* `opus` prompt measured **$0.41**, because a non-bare run loads CLAUDE.md, plugins, and skills — so for bulk or mechanical work, *propose* `sonnet`/`haiku` and let the user decide. Never downgrade silently.
2. **Never pass `--bare` on OAuth (subscription) auth.** Bare mode skips OAuth and keychain reads entirely, so it fails with `apiKeySource: "none"` and the result string `"Not logged in · Please run /login"` (exit 1). Verified on Claude Code 2.1.209. `--bare` is safe *only* when the run has a non-OAuth credential source: `ANTHROPIC_API_KEY`, an `apiKeyHelper` supplied via `--settings`, or Bedrock / Google Cloud / Microsoft Foundry provider credentials. The official docs recommend `--bare` for scripts — that advice silently assumes one of those.
3. **`--tools` is the boundary; `--allowedTools` is only a prompt-suppressor.** `--allowedTools` auto-approves what it lists — it does **not** remove anything else. An unlisted tool falls through to the ambient permission mode, and if the machine's settings make that mode permissive (`permissions.defaultMode` of `acceptEdits`, `auto`, or `bypassPermissions`), the subagent is handed the full tool set and its writes are **approved, not denied** — nothing lands in `.permission_denials`, so the gate reports a clean success. Verified against a settings file with `defaultMode: bypassPermissions`: a "read-only" call carrying `--allowedTools "Read,Glob,Grep"` was offered **32 tools, edited the target file, and reported `permission_denials: 0`**. The same call with the flags below ran at `permissionMode: dontAsk` with exactly three tools and left the file untouched.

   A sandbox therefore needs **two** layers, and neither substitutes for the other:
   - `--tools "Read,Glob,Grep"` — bounds which built-in tools exist at all.
   - `--permission-mode dontAsk` — pins the ambient mode, so an unlisted call is denied instead of inheriting whatever the machine's settings allow. Without it, a run with `Bash` in `--tools` but only `Bash(git log *)` in `--allowedTools` **executed `touch pwned.txt`** under a permissive mode (`permission_denials: 0`); with it, the same call was denied (`permission_denials: 1`, no file).

   Add `--strict-mcp-config` to drop the project's MCP servers too (verified: `mcp_servers` goes 1 → 0) — `--tools` governs the built-in set, and the docs note MCP tools are not covered by it. Ask the user before widening to `Edit`, `Write`, or `Bash`, and widen `--tools` and `--allowedTools` together.
4. **Working directory is the shell's cwd** — there is no `-C` flag. `cd` into the target directory, or use `--add-dir <path>`, and state the absolute path in the prompt.
5. Run the command under a wall-clock `timeout`, with stdin redirected and stderr captured (see [Avoiding hangs](#avoiding-hangs-and-silent-failures)), then check all three failure signals before trusting the output.
6. **After the run completes**, tell the user the session can be resumed: report the `session_id` and offer a follow-up via `--resume`.

### Canonical safe invocation

Capture stdout to a file (never pipe straight into `jq` — that replaces `claude`'s exit code with `jq`'s, and a killed run then looks like a clean empty success), then gate on all three failure signals before printing anything as an answer.

```bash
ERRLOG=$(mktemp); OUT=$(mktemp)
timeout -k 10 600 claude -p "your prompt here" \
  --model opus \
  --tools "Read,Glob,Grep" \
  --allowedTools "Read,Glob,Grep" \
  --permission-mode dontAsk \
  --strict-mcp-config \
  --output-format json \
  < /dev/null >"$OUT" 2>"$ERRLOG"
rc=$?   # not `status`: that name is read-only in fish/zsh

LAST=$(jq -c 'if type=="array" then last else . end' "$OUT" 2>/dev/null)
[ -z "$LAST" ] && LAST='{}'   # timed-out runs leave $OUT empty or truncated
# `.is_error // true` would be WRONG: jq's `//` also replaces `false`, so a clean run reads as an error.
IS_ERR=$(printf '%s' "$LAST" | jq -r 'if .is_error == false then "false" else "true" end')
DENIALS=$(printf '%s' "$LAST" | jq -r '(.permission_denials // []) | length')
SID=$(printf '%s' "$LAST" | jq -r '.session_id // empty')   # keep for --resume

if [ "$rc" -ne 0 ] || [ "$IS_ERR" != false ] || [ "$DENIALS" != 0 ]; then
  echo "claude subagent FAILED — exit=$rc is_error=$IS_ERR denials=$DENIALS" >&2
  printf '%s' "$LAST" | jq -r '.result // "(no result)"' >&2   # error text, not an answer
  cat "$ERRLOG" >&2
  rm -f "$OUT" "$ERRLOG"
  exit 1            # never fall through to .result, and never exit 0 on a failed run
fi

printf '%s' "$LAST" | jq -r '.result'
rm -f "$OUT" "$ERRLOG"   # only after reading them — the error log is the diagnosis
```

The early `exit 1` matters twice over. It stops the failure text (`"Not logged in · Please run /login"`, or the observed false `"DONE"`) from being printed on stdout as if it were the answer, and it gives the caller an exit status they can branch on — a run that detects a failure and still exits 0 is exactly the hazard this skill exists to remove. In a function, `return 1`; if the snippet is inlined where `exit` would kill the parent shell, set a flag instead and check it.

## Result contract

`--output-format json` on 2.1.209 prints a **JSON array** of events (`system/init`, `assistant`, …, `result`), not the single object the docs describe. `jq -r '.result'` — the form used by the official docs — fails with `Cannot index array with string "result"`. Extract the last element instead: `jq -r 'if type=="array" then last else . end | .result'` handles both shapes.

A run only succeeded if **all three** of these hold. Checking only one lets a failure pass as an answer:

| Signal | Failure looks like |
| --- | --- |
| `rc` (exit code) | Non-zero. `124` = the `timeout` fired, `137` = `-k` had to SIGKILL it. |
| `.is_error` on the last event | `true` while `subtype` still says `"success"` — and `.result` holds the *error text*, e.g. `"Not logged in · Please run /login"`. A naive reader prints that as if it were the answer. |
| `.permission_denials \| length` | Non-zero while `is_error` is `false` and `rc` is `0`. The tool was not in `--allowedTools`, so it was denied without a prompt and the work never happened. `.result` may ask for permission — or **claim success anyway**: an observed run answered `"DONE"` with the target file untouched. |

The gate catches failures, not over-reach. `.permission_denials` only records tools that were **denied** — under a permissive ambient permission mode there is no denial to record, and an unlisted `Edit` or `Bash` simply runs (see step 3). Bound the tool surface with `--tools`; the gate cannot do it for you.

Other useful fields on the last event: `.session_id`, `.total_cost_usd`, `.num_turns`, and `.structured_output` (with `--json-schema`).

## Avoiding hangs and silent failures

1. **Redirect stdin: `< /dev/null`.** `claude -p` reads stdin and waits for **EOF** when stdin carries data. A writer that sends bytes and never closes blocks the run **forever, before any work happens** — verified: 0 bytes of output, killed by `timeout` at 60 s. (An *empty* open stdin does exit cleanly, but don't rely on the distinction.) The only time you omit the redirect is when you are *deliberately* piping context in — a real pipe supplies EOF and is safe:
   ```bash
   ERRLOG=$(mktemp); OUT=$(mktemp)
   set -o pipefail   # without it, a failing `git diff` is invisible and Claude just gets empty stdin
   git diff main | timeout -k 10 600 claude -p "Review this diff for bugs." \
     --model opus --output-format json >"$OUT" 2>"$ERRLOG"
   rc=$?   # the pipeline's status — with pipefail, non-zero if either git diff or claude failed
   ```
   Claude's stdout still goes to a file, not into a pipe, so `rc` is meaningful. Then apply the same three checks as the canonical block. Piped stdin is capped at 10 MB; for larger inputs, write a file and name its path in the prompt.
2. **Wrap every call in `timeout -k`.** `timeout -k 10 600` sends SIGTERM at 10 min and SIGKILL 10 s later, so the call **cannot** hang indefinitely. Treat exit `124`/`137` as "timed out" and report it — do not silently retry. Raise the cap for large tasks; don't remove it.
3. **Keep stderr, surface it on failure.** Capture with `2>"$ERRLOG"` (`ERRLOG=$(mktemp)` first) instead of `2>/dev/null`, and read it **before** any `rm`. On a clean run the log is empty; on a non-zero exit or empty stdout, `cat "$ERRLOG"` is where the real cause (auth, rate limit, network) shows up.
4. **Never end a `claude -p` call with `| jq`.** The pipeline's exit code becomes jq's: a run killed by `timeout` feeds jq empty input, jq prints nothing and exits 0, and the caller cannot tell a timeout from a successful run with nothing to say. Redirect to a file, capture `rc`, then run `jq` against the file.

## Quick Reference

Unless a row says otherwise, assume the canonical block: `< /dev/null`, stdout to `$OUT`, stderr to `$ERRLOG`, a `timeout -k` wrapper, `rc=$?`, and the three-signal gate. The diff row deliberately supplies stdin, and the structured-output row reads `.structured_output` instead of `.result`.

Every row also carries `--permission-mode dontAsk` — without it the rows below are suggestions, not limits (see step 3). `--tools` bounds what exists; `--allowedTools` suppresses the prompts for what you intend.

| Use case | Key flags |
| --- | --- |
| Read-only analysis (default) | `--model opus --tools "Read,Glob,Grep" --allowedTools "Read,Glob,Grep" --strict-mcp-config` |
| Apply local edits | `--tools "Read,Edit,Write,Glob,Grep" --allowedTools "Read,Edit,Write,Glob,Grep"` (ask the user first) |
| Code + shell | `--tools "Read,Edit,Bash" --allowedTools "Read,Edit,Bash"` (ask the user first) |
| Scoped shell only | `--tools "Read,Bash" --allowedTools "Bash(git log *),Bash(git diff *),Read"` — `--tools` keeps `Bash` in reach, `--allowedTools` narrows it to those commands, `dontAsk` denies the rest |
| Cheap mechanical task | `--model haiku` |
| Review a diff | `git diff main \| claude -p "review this diff" …` (real pipe: omit `< /dev/null`) |
| Structured result | `--json-schema '{…}'`, then read `.structured_output` |
| Cap the blast radius | `--max-turns 10`, `--max-budget-usd 1.00` |
| Custom role | `--append-system-prompt "You are a security engineer."` |
| Another directory | `--add-dir /abs/path` (there is no `-C`) |

## Session continuity

`$SID` comes from the canonical block above — read it out of `$OUT` **before** the `rm`, or the id is gone. An empty `$SID` does not fail loudly; `--resume ""` errors with `--resume requires a valid session ID`.

```bash
ERRLOG=$(mktemp); OUT=$(mktemp)
timeout -k 10 600 claude -p "follow-up question" --resume "$SID" \
  --output-format json < /dev/null >"$OUT" 2>"$ERRLOG"
rc=$?

LAST=$(jq -c 'if type=="array" then last else . end' "$OUT" 2>/dev/null)
[ -z "$LAST" ] && LAST='{}'
IS_ERR=$(printf '%s' "$LAST" | jq -r 'if .is_error == false then "false" else "true" end')
DENIALS=$(printf '%s' "$LAST" | jq -r '(.permission_denials // []) | length')
if [ "$rc" -ne 0 ] || [ "$IS_ERR" != false ] || [ "$DENIALS" != 0 ]; then
  echo "resume FAILED — exit=$rc is_error=$IS_ERR denials=$DENIALS" >&2
  printf '%s' "$LAST" | jq -r '.result // "(no result)"' >&2   # the cause often lives here, not in stderr
  cat "$ERRLOG" >&2
  rm -f "$OUT" "$ERRLOG"; exit 1
fi
printf '%s' "$LAST" | jq -r '.result'
rm -f "$OUT" "$ERRLOG"
```

The gate is repeated rather than referenced: a resumed run fails the same three ways, and each Bash invocation is a fresh shell, so a snippet that omits the gate omits it for real.

`--resume "$SID"` reuses the same session id and remembers the earlier turns (verified: the subagent recalled a codeword set in the first call). Run the follow-up **from the same directory** as the original — session lookup is scoped to the project directory and its git worktrees. `--continue` picks the most recent session in the cwd without capturing an id; `--fork-session` branches instead of appending.

## Parallel execution

Send multiple Bash tool calls in a single response; each `claude -p` runs independently. Right for independent subtasks — no shared files being edited, no ordering dependency. Give each its own `$OUT`/`$ERRLOG`.

## Writing effective subagent prompts

The subagent starts cold. Brief it like someone who just walked in:

```
Working directory: {abs_path}
Task: {what to produce}
Output: {where to save / how to return}
Constraints: {language, format, scope limits}
```

## Critical evaluation of the subagent's output

A Claude subagent is the **same model family you are** — it shares your blind spots, so agreement is weak evidence. Do not treat "the subagent agreed" as confirmation.

- Use it for **isolation** (clean context, scoped permissions, parallel work), not for a second opinion.
- For a genuine cross-model check, use the `codex-subagent` or `antigravity-subagent` skills instead.
- Verify claims about recent APIs, versions, or model names — the subagent has the same knowledge cutoff you do.

## Error Handling

- Stop and report whenever `claude --version` fails, `rc` is non-zero, `is_error` is `true`, or `permission_denials` is non-empty. `cat "$ERRLOG"` first — the cause is usually there, not in stdout.
- **Auth**: `"Not logged in · Please run /login"` means the run had no usable credentials. Check `claude auth status`; ask the user to run `! claude auth login` (it is interactive). If it only happens with `--bare`, drop `--bare` — that is the expected behavior on OAuth auth.
- **Exit 124/137**: the `timeout` killed the run. Report it as a timeout; consider a larger cap, a cheaper model, or `--max-turns`. Do not retry blindly.
- **Empty stdout, no error**: the run was killed while blocked on stdin. Confirm `< /dev/null` was present.
- Before using `--allowedTools` entries that write or execute (`Edit`, `Write`, `Bash`) or `--permission-mode` values that bypass prompts (`acceptEdits`, `bypassPermissions`), ask the user with `AskUserQuestion` unless permission was already given.
