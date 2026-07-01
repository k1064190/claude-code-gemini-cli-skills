# Stage 1 — codex/agy subagent "freeze / no response" fix

## Why
Doctor Cho reported that the `codex-subagent` and `antigravity-subagent` (`agy`) skills
"often don't get the request through and seem to freeze." The skills were the only
supported subagents (gemini is deprecated), so the stalls blocked real work.

## What
Rewrote the invocation guidance in both skills to eliminate the freeze/no-response class.
Empirically confirmed root causes and the fixes for each:

1. **stdin blocking (primary).** `codex exec "prompt"` and `agy -p "prompt"` read stdin
   whenever it is a non-TTY. If the harness hands over an open pipe with no EOF, the call
   blocks forever. `2>/dev/null` hid codex's `Reading additional input from stdin...` line,
   so it looked like a silent freeze. Confirmed by OpenAI codex issue #20919; official
   workaround is `< /dev/null`. → Added `< /dev/null` to every non-resume invocation.
2. **`2>/dev/null` black-holed errors.** Auth/rate-limit/flag errors produced empty stdout
   and no diagnostics. → Capture stderr to `ERRLOG=$(mktemp)` and surface it on non-zero
   exit / empty stdout.
3. **No hard timeout.** codex skill had none; a slow high-effort run looked frozen. →
   `timeout -k 10 <N>` wrapper: SIGTERM at the cap, SIGKILL 10s later, so the call cannot
   hang indefinitely. Exit 124 = timed out, 137 = force-killed.
4. **`--full-auto` deprecated** in codex → replaced with `--sandbox workspace-write`.
5. **agy `-p` without `--dangerously-skip-permissions`** writes edits to a scratch sandbox
   (`~/.gemini/antigravity-cli/scratch/`), not the project → documented so "reported success
   but files unchanged" is diagnosable.
6. **`codex exec resume` flag claim** ("no flags allowed") was wrong per `resume --help`
   (`resume [OPTIONS] [SESSION_ID] [PROMPT]`) → corrected.

## How
Investigated with `superpowers:systematic-debugging` (root cause before fix). Every claim was
reproduced deterministically (open-pipe stdin → exit 124; `< /dev/null` → exit 0; bad flag →
captured stderr; `timeout` bounded even inside `$(...)`; `timeout -k` kills a SIGTERM-ignoring
child at exit 137). Cross-checked against official codex docs (developers.openai.com/codex)
and the installed CLIs' own `--help`.

## Code locations
- `codex-subagent/SKILL.md` — new "Avoiding hangs and silent failures" section; steps 3–6,
  Quick Reference, Error Handling updated; canonical safe-invocation block.
- `antigravity-subagent/SKILL.md` — new "Avoiding hangs and silent failures (read first)"
  section + scratch-sandbox note; Output-format / Mode 1 / `@`-syntax / resume / parallel /
  error-handling examples updated; canonical block.

## Review loop
- `code-reviewer-pro` subagent + `codex` review (dogfooding the fixed pattern) both ran on the
  diff. Convergent findings, all addressed: undefined `$ERRLOG` in inline snippets (hard bash
  failure — verified `2>"$UNSET"` errors) → added an explicit standing-convention note;
  parallel-block stderr inconsistency → aligned; `status` var collides with fish/zsh read-only
  `$status` → renamed to `rc`; resume "no flags" claim → corrected after checking `resume --help`.
- Dismissed: codex's claim that `echo "p" | codex exec resume --last` needs `-` — TEST 5
  proved the piped text is used as the resumed prompt on codex 0.139.0.
- The `agy` review was left in the pool and stopped early; agy was already dogfooded
  successfully elsewhere, and its `< /dev/null` run started (did not hang) — it was just slow
  reasoning, bounded by the outer `timeout`.

## Retrospective
The user's instinct that "even long reasoning can't take 5 hours" was the key prompt to prove
the timeout is a *hard* bound (and to upgrade `timeout` → `timeout -k`), not just usually-works.
Carry forward: for any "it froze" report, first check whether the process is blocked on stdin
and whether stderr is being discarded — both turn ordinary conditions into invisible hangs.
