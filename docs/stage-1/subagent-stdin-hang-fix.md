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
Three reviewers per CLAUDE.md: `code-reviewer-pro` subagent, `codex` (GitHub bot on PR #3), and
`antigravity-subagent`/`agy`.

- **Round 1 (code-reviewer-pro + codex):** convergent findings addressed — undefined `$ERRLOG` in
  inline snippets (hard bash failure — verified `2>"$UNSET"` errors) → standing-convention note;
  parallel-block stderr inconsistency → aligned; `status` collides with fish/zsh read-only
  `$status` → renamed `rc`; resume "no flags" claim → corrected via `resume --help`. Codex bot P2
  on parallel `$ERRLOG` → made each parallel call self-contained. **Dismissed with evidence:**
  codex `--add-dir` P2 (TEST: `--dangerously-skip-permissions` alone lands edits in the real cwd)
  and codex's `resume` stdin `-` claim (TEST 5: piped text is used as the resumed prompt).
- **Round 2 (agy review):** running agy on the diff surfaced two *real* defects I introduced —
  (a) the positional resume form `resume --last "prompt"` lacked `< /dev/null` (same stdin hang);
  (b) resume examples still used `2>/dev/null`, contradicting the capture-stderr rule. Both fixed.
- **The big one — `agy` "review takes 40 minutes" hang.** Running agy with `@final.diff` from a
  scratch dir made agy's agent launch `find /home/cwh -name final.diff` (a full `$HOME` scan) to
  locate the file. `timeout -k` killed `agy` but not that grandchild, which inherited the
  `RESULT=$(…)` pipe's write end and kept command substitution blocked for ~40 min. Root cause
  confirmed via `/proc/<pid>/fd` pipe-inode ownership. Two fixes: (1) **capture agy stdout to a
  file, never `$(agy …)`** — proven with a synthetic grandchild (30 s hang via `$()` vs 0 s via
  `> file`); (2) **don't make agy hunt for files** — inline small content or use a cwd-relative
  `@path`. The old canonical block's "survives command substitution" claim was wrong and was
  corrected.

## Retrospective
Two user push-backs drove the most important fixes. "Even long reasoning can't take 5 hours" →
upgrade `timeout` to a *hard* bound (`timeout -k`). "Making agy traverse `$HOME` is nonsense" →
found the real 40-min-hang mechanism (agentic `find` grandchild holding a command-substitution
pipe). Carry forward: for any subagent "it froze" report, check three things — blocked on stdin,
stderr discarded, and **a lingering agentic grandchild holding a `$(…)` pipe past the timeout**;
capture long agentic-CLI output to a file, not command substitution.
