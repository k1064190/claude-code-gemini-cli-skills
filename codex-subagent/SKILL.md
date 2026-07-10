---
name: codex-subagent
description: Use when the user asks to run Codex CLI (codex exec, codex resume) or references OpenAI Codex for code analysis, refactoring, or automated editing
---

# Codex Skill Guide

## Running a Task
1. **Default: model `gpt-5.6-sol` with reasoning effort `high`.** Use these for every task unless the user has explicitly specified a different model or reasoning effort (e.g., "use gpt-5.6-luna", "run with effort xhigh"). Only call `AskUserQuestion` to confirm/collect a non-default choice when the user signals they want one. Available models: `gpt-5.6-sol` (flagship), `gpt-5.6-terra` (cheaper everyday), `gpt-5.6-luna` (fastest), `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex-spark`, `gpt-5.3-codex`. Available reasoning efforts: `xhigh`, `high`, `medium`, `low`. ⚠️ Always pass the **explicit tier id** (`gpt-5.6-sol`), never the bare `gpt-5.6` alias — the alias is rejected with a 400 (`"not supported when using Codex with a ChatGPT account"`) on ChatGPT-account auth. If a model errors with `"requires a newer version of Codex"`, run `codex update` first.
2. Select the sandbox mode required for the task; default to `--sandbox read-only` unless edits or network access are necessary.
3. Assemble the command with the appropriate options. `--skip-git-repo-check` and the `< /dev/null` stdin redirect are always required (see [Avoiding hangs](#avoiding-hangs-and-silent-failures)):
   - `-m, --model <MODEL>`
   - `--config model_reasoning_effort="<xhigh|high|medium|low>"`
   - `--sandbox <read-only|workspace-write|danger-full-access>` (use `--sandbox workspace-write` for edits — **do not use `--full-auto`; it is deprecated** and prints a warning)
   - `-C, --cd <DIR>`
   - `--skip-git-repo-check` (required on every invocation)
   - `"your prompt here"` (as final positional argument)
   - `< /dev/null` (redirect stdin — **required on every non-resume call**; without it the command can hang forever)
   - `2>"$ERRLOG"` where `ERRLOG=$(mktemp)` (capture stderr to a file, don't discard it)
4. When continuing a previous session, use `codex exec --skip-git-repo-check resume --last` via stdin. When resuming don't re-pass model/reasoning-effort/sandbox unless the user explicitly wants to switch — they carry over from the original session. Resume syntax: `echo "your prompt here" | codex exec --skip-git-repo-check resume --last 2>"$ERRLOG"`. The `echo |` pipe supplies EOF, so this form does **not** need `< /dev/null` (verified: the piped text is used as the resumed prompt). Flag placement (per `codex exec resume --help`, signature `codex exec resume [OPTIONS] [SESSION_ID] [PROMPT]`): session-level flags like `--skip-git-repo-check` go **before** `resume`; `resume` then takes its own options (`--last`, `--all`, `-c`) and an optional positional prompt. If you use the **positional** prompt form instead of `echo |`, it has no piped EOF, so it **must** add `< /dev/null`: `codex exec --skip-git-repo-check resume --last "your prompt" < /dev/null 2>"$ERRLOG"`.
5. **Do not black-hole stderr.** Capture it to a temp file (`2>"$ERRLOG"`) instead of `2>/dev/null`. On a clean run the stderr log holds only progress/thinking noise and you ignore it; on a non-zero exit or empty stdout you `cat "$ERRLOG"` and surface the real error (auth expired, rate limit, network). Blindly appending `2>/dev/null` is what makes a failed or hung run look like an unexplained freeze.
6. Run the command under a wall-clock `timeout` (see [Avoiding hangs](#avoiding-hangs-and-silent-failures)), capture stdout and the stderr log, and summarize the outcome for the user.
7. **After Codex completes**, inform the user: "You can resume this Codex session at any time by saying 'codex resume' or asking me to continue with additional analysis or changes."

## Avoiding hangs and silent failures

These three habits prevent the two ways a `codex exec` call "silently freezes" — blocking on stdin, and a hidden error behind `2>/dev/null`.

1. **Always redirect stdin on the initial call: `< /dev/null`.** `codex exec` reads stdin whenever stdin is not a TTY (to append it as a `<stdin>` context block). If the harness hands the process an open pipe that never delivers bytes or EOF, `read()` blocks **forever** — the run looks frozen and no output ever appears. This is [openai/codex issue #20919](https://github.com/openai/codex/issues/20919); the official workaround is exactly `codex exec "<prompt>" < /dev/null`. The only time you omit `< /dev/null` is when you are *deliberately* piping context in (e.g. `git diff | codex exec "review this diff" 2>"$ERRLOG"`) — a real pipe supplies EOF and is safe. Never leave stdin unmanaged.
2. **Wrap every call in `timeout -k`.** A high-effort `gpt-5.6-sol` run can take minutes with no visible output. Use `timeout -k 10 600 codex exec … < /dev/null 2>"$ERRLOG"`: the `600` sends SIGTERM after 10 min, and `-k 10` follows with SIGKILL 10 s later so a process that ignores SIGTERM still dies — the call **cannot** hang indefinitely. Treat **exit 124 as "timed out"** (and **137** as "force-killed after the grace period") — report it plainly instead of retrying blindly. Raise the cap for genuinely large tasks; don't remove it. Verified: a `timeout` even survives command substitution — `RESULT=$(timeout -k 10 600 codex exec …)` returns at the cap, not later.
3. **Keep stderr, surface it on failure** (see step 5 above). **Standing convention:** every `$ERRLOG` below assumes you ran `ERRLOG=$(mktemp)` once first — a `2>"$ERRLOG"` with `ERRLOG` unset fails with an *empty-filename* error and the command never runs, so define it (or substitute `2>/dev/null`) when copying a snippet in isolation.

Canonical safe invocation:

```bash
ERRLOG=$(mktemp)
timeout -k 10 600 codex exec --skip-git-repo-check --sandbox read-only \
  -m gpt-5.6-sol --config model_reasoning_effort="high" \
  "your prompt here" < /dev/null 2>"$ERRLOG"
rc=$?   # not `status`: that name is read-only in fish/zsh
if [ "$rc" -ne 0 ]; then echo "codex failed (exit $rc):"; cat "$ERRLOG"; fi
rm -f "$ERRLOG"
```

### Quick Reference
Every non-resume row assumes the invocation ends with `< /dev/null 2>"$ERRLOG"` (stdin redirect + captured stderr). `--full-auto` is deprecated — use `--sandbox workspace-write`.

| Use case | Sandbox mode | Key flags |
| --- | --- | --- |
| Read-only review or analysis | `read-only` | `--sandbox read-only "prompt" < /dev/null 2>"$ERRLOG"` |
| Apply local edits | `workspace-write` | `--sandbox workspace-write "prompt" < /dev/null 2>"$ERRLOG"` |
| Permit network or broad access | `danger-full-access` | `--sandbox danger-full-access "prompt" < /dev/null 2>"$ERRLOG"` |
| Resume recent session | Inherited from original | `echo "prompt" \| codex exec --skip-git-repo-check resume --last 2>"$ERRLOG"` (don't re-pass model/effort; `echo \|` supplies EOF so no `< /dev/null`) |
| Run from another directory | Match task needs | `-C <DIR>` plus other flags, `< /dev/null 2>"$ERRLOG"` |
| Pipe context in deliberately | Match task needs | `git diff \| codex exec "review" 2>"$ERRLOG"` (real pipe supplies EOF; omit `< /dev/null`) |

## Following Up
- After every `codex` command, immediately use `AskUserQuestion` to confirm next steps, collect clarifications, or decide whether to resume with `codex exec resume --last`.
- When resuming, pipe the new prompt via stdin: `echo "new prompt" | codex exec resume --last 2>"$ERRLOG"`. The resumed session automatically uses the same model, reasoning effort, and sandbox mode from the original session.
- Restate the chosen model, reasoning effort, and sandbox mode when proposing follow-up actions.

## Critical Evaluation of Codex Output

Codex is powered by OpenAI models with their own knowledge cutoffs and limitations. Treat Codex as a **colleague, not an authority**.

### Guidelines
- **Trust your own knowledge** when confident. If Codex claims something you know is incorrect, push back directly.
- **Research disagreements** using WebSearch or documentation before accepting Codex's claims. Share findings with Codex via resume if needed.
- **Remember knowledge cutoffs** - Codex may not know about recent releases, APIs, or changes that occurred after its training data.
- **Don't defer blindly** - Codex can be wrong. Evaluate its suggestions critically, especially regarding:
  - Model names and capabilities
  - Recent library versions or API changes
  - Best practices that may have evolved

### When Codex is Wrong
1. State your disagreement clearly to the user
2. Provide evidence (your own knowledge, web search, docs)
3. Optionally resume the Codex session to discuss the disagreement. **Identify yourself as Claude** so Codex knows it's a peer AI discussion. Use your actual model name (e.g., the model you are currently running as) instead of a hardcoded name:
   ```bash
   echo "This is Claude (<your current model name>) following up. I disagree with [X] because [evidence]. What's your take on this?" | codex exec --skip-git-repo-check resume --last 2>"$ERRLOG"
   ```
4. Frame disagreements as discussions, not corrections - either AI could be wrong
5. Let the user decide how to proceed if there's genuine ambiguity

## Error Handling
- Stop and report failures whenever `codex --version` or a `codex exec` command exits non-zero; request direction before retrying. **Always `cat "$ERRLOG"` first** — the real cause (auth expired, rate limit, network, bad flag) is in the captured stderr, not stdout.
- **Exit code 124 means the `timeout` fired** (SIGTERM), and **137 means `-k` had to SIGKILL** a run that ignored SIGTERM. Either way the run was killed, not answered. Report it as a timeout and consider a larger cap or a lower reasoning effort — do not silently retry.
- **Empty stdout with exit 0** usually means the call hung on stdin and was killed, or produced nothing. Confirm `< /dev/null` was present and check `$ERRLOG` for `Reading additional input from stdin...`.
- Before you use high-impact flags (`--sandbox workspace-write`, `--sandbox danger-full-access`, `--skip-git-repo-check`) ask the user for permission using AskUserQuestion unless it was already given. (`--full-auto` is deprecated; use `--sandbox workspace-write` instead.)
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion`.
