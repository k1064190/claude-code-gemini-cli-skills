# Stage 3 — `claude-subagent` skill (`claude -p`)

## Why

The repo delegated to three external CLIs (`codex`, `gemini`, `agy`) but not to Claude Code itself. A `claude -p` subagent is worth having for **isolation**, not for a second opinion: a cold instance with no conversation history, scoped `--allowedTools`, and its own process — useful for parallel subtasks, permission-limited edits, and orchestration from Codex/Antigravity.

An older private copy existed at `~/.claude/skills/claude-subagent` (written 2026-05-03, never in this repo). It was checked against the live CLI and found wrong in three load-bearing ways, so it was rewritten from the current docs plus measurement, and the old copy was deleted.

## What

- **`claude-subagent/SKILL.md`** — new skill. Default: `--model opus`, no `--bare`, `--output-format json`, read-only `--allowedTools "Read,Glob,Grep"`.
- **`README.md`** — Skills / Defaults / Requirements / Installation / Quick start / Models / Tested on now cover Claude.
- Three findings that contradict both the old skill and the official docs, each reproduced before being written down:

| Finding | Evidence |
| --- | --- |
| **`--bare` is unusable on subscription auth.** Bare mode skips OAuth and keychain reads entirely; the docs recommend it for scripts, but that assumes API-key auth. | `claude --bare -p …` → exit 1, `apiKeySource: "none"`, `.result` = `"Not logged in · Please run /login"`. |
| **`--output-format json` emits a JSON _array_ of events** on 2.1.209, not the single object the docs describe. `jq -r '.result'` — the official form, and the old skill's — fails with `Cannot index array with string "result"`. | Reproduced in a clean env (`env -i`), so it is not an artifact of nesting inside Claude Code. Fix: `jq -r 'if type=="array" then last else . end \| .result'`. |
| **A run can fail while exiting 0.** `.is_error` can be `true` with `subtype: "success"`, and a tool missing from `--allowedTools` is denied silently. | Auth failure puts the error prose in `.result`. A denied edit answered **`"DONE"` with the file untouched** — `.permission_denials` was the only honest signal. |
| **`--allowedTools` is not a sandbox.** It suppresses prompts; it does not remove tools. Unlisted tools fall through to the ambient permission mode, and a project's own settings/hooks run outside the tool boundary altogether. | Against a repo whose `.claude/settings.json` set `defaultMode: bypassPermissions`, a "read-only" call with `--allowedTools "Read,Glob,Grep"` was offered **32 tools, edited the file, and reported `permission_denials: 0`**. A `PreToolUse` hook in that repo **executed a shell command** during the run. The bound needs four flags: `--tools`, `--permission-mode dontAsk`, `--setting-sources user`, `--strict-mcp-config`. |

## How

Evidence first, prose second: every claim in the skill was run before it was written. Five core tests (Doctor Cho's scope call), plus a dogfood pass.

1. **T1 canonical call** (`--model opus`) → exit 0, `is_error=false`, `$0.41` for a trivial prompt (a non-bare run loads CLAUDE.md, plugins, skills — hence the `haiku`/`sonnet` guidance).
2. **T2 `--bare`** → the auth failure above.
3. **T3 stdin.** The first design was **wrong**: `sleep 300 | claude …` timed out, but `timeout` was wrapping the whole pipeline, so the 124 came from `sleep`, not from Claude. Redone with a FIFO held open by a background writer so `timeout` wrapped Claude alone:
   - stdin open but **empty** → exits cleanly (10 s). Unlike `codex`, which hangs here.
   - stdin **carrying data** with no EOF → **blocks forever before any work happens** (0 bytes out, killed at 60 s).
   - normal pipe (EOF) → works, and the piped text reaches the model.
   So the rule is `< /dev/null` on every call except a deliberate pipe — same conclusion as stage 1, but for a narrower reason, and the narrower reason is what got written down.
4. **T4 `--allowedTools`** → denied edit leaves the file untouched, `permission_denials: 1`; allowed edit changes the file. The child session runs at `permissionMode: default` and does **not** inherit the parent's `auto`.
5. **T5 `--resume`** → `session_id` captured from the result event; the follow-up recalled a codeword from the first call and reused the same id.

**Dogfood.** The skill's own canonical invocation was then used to have a fresh `claude -p` audit `SKILL.md`. It found five real defects — undefined `$ERRLOG` in a copy-pasteable snippet, a resume example reading a file the canonical snippet had already `rm`'d, cleanup destroying the error log the doc tells you to read, and, worst, **the canonical block performing none of the three checks the doc demands** (it was itself the naive reader it warns about). It also caught that ending a call with `| jq` replaces Claude's exit code with jq's, so a `timeout` kill prints nothing and exits 0 — a hang indistinguishable from a quiet success.

**Self-inflicted bug, caught by testing.** The rewritten block used `.is_error // true` as a fail-closed default. jq's `//` replaces `false` as well as `null`, so a *clean* run reported `is_error=true`. Extracting the block straight out of `SKILL.md` and running it on four paths (success / auth failure / denial / timeout) surfaced it immediately; the fix is an explicit `if .is_error == false`.

## Code locations

- `claude-subagent/SKILL.md` — the skill. Canonical block (three-signal gate, stdin redirect, `timeout -k`, cleanup after reading) under "Canonical safe invocation"; the traps in "Result contract"; the stdin/timeout/pipe rules in "Avoiding hangs and silent failures".
- `README.md:54` (Defaults → Claude), `:185` (Quick start → Claude), `:257` (Models → Claude), `:328` (Tested on).
- Outside the repo (local install, not in the diff): the stale private copy at `~/.claude/skills/claude-subagent/` was deleted and replaced by a symlink to `claude-subagent/` here, so the installed skill and the tracked one can no longer drift apart.

## Review loop

| Reviewer | Outcome |
| --- | --- |
| `code-reviewer` subagent | 1 finding, **fixed**: the README quick-start still ended its calls with `\| jq`, violating the rule the same diff had just introduced (the rule was added late and the README was not re-checked). |
| `codex-subagent` | 6 findings. **Fixed 5**: README snippet claimed to "check the exit code" but never captured `rc` or the other two signals; `git diff \| claude` swallowed a failing `git diff` (added `set -o pipefail`); the model policy said "switch only on explicit request" while also licensing a silent cost-based downgrade (now: *propose*, never switch silently); the Quick Reference preamble contradicted two of its own rows; stale README line numbers in this doc. **Dismissed 1**: it objected to recording the `~/.claude/skills/` cleanup here because that path is outside the diff. Stage docs are a log of what was done, not a description of the diff, so the entry stays — reworded to say plainly that it is a local action outside the repo. |
| `antigravity-subagent` | `NO DEFECTS` — but only on the fourth attempt. Three `@claude-subagent/SKILL.md` runs died with `Error: timeout waiting for response` (including at `--print-timeout 15m`), while a trivial `-p` probe answered instantly. Inlining the file's *content* into the prompt worked immediately. This re-confirms the stage-1 caveat already in `antigravity-subagent/SKILL.md`: don't make `agy` hunt for files. |

### Codex GitHub bot (PR #6)

- **Round 1 — 9 findings, all valid, all fixed.** One theme dominated: the snippets detected a failure and then printed `.result` anyway and exited 0, so an orchestrator could not branch on the exit status. They now report the diagnosis on stderr and `exit 1` on any of the three signals. Also: `--tools` corrected (built-ins only; MCP needs `--disallowedTools "mcp__*"` / `--strict-mcp-config`), the `--bare` precondition broadened past `ANTHROPIC_API_KEY` to the other documented non-OAuth sources, and the model policy stopped licensing a silent cost-based downgrade.
- **Round 2 — 3 findings, and the P2 was the most valuable review comment of the stage.** It claimed the read-only default was not read-only. Reproducing it was worse than the claim (see the table above), and the fix is the four-flag bound.
- **Round 3 — never ran.** The bot replied `"You have reached your Codex usage limits for code reviews."` So the round-2 permission-model change carries no Codex sign-off; it was reviewed locally instead (`code-reviewer` found the Quick Reference rows had dropped `--permission-mode dontAsk`; `agy` found the project-hook escape hatch that `--setting-sources user` closes, and its "jq crashes on a truncated `$OUT`" finding was **dismissed** — the `[ -z "$LAST" ] && LAST='{}'` guard already handles it, as the timeout test shows).

**Blocked, reported, not fixed here:** the `codex-subagent` default from stage 2, `gpt-5.6-sol`, is now rejected outright — `400 … "The 'gpt-5.6-sol' model is not supported when using Codex with a ChatGPT account"` (codex-cli 0.144.3, and it is also the default in `~/.codex/config.toml`). `gpt-5.5` works, and the local codex review above ran on it. Fixing the stage-2 default is out of scope for this stage and needs its own decision.

## Retrospective

Most of the headline findings contradict the official documentation, and none were discoverable by reading it — the JSON-array shape, the exit-0 failures, and the permission model all only appear when you run the command and inspect what came back. The review layers each caught a different class, and each caught something the others missed: the dogfood pass found that a skill preaching "check all three signals" shipped a canonical snippet checking none; the Codex bot found that the read-only default was not read-only; `agy` found that even the hardened version let a repo's own hooks run shell commands.

The through-line is that every fix here started as an *assumption stated as fact*. `--allowedTools` sounded like a sandbox, so it was documented as one. Reproducing the reviewer's claim — rather than accepting or rejecting it — is what turned it into a bound. Carry forward: extract snippets from the doc and execute them (a retyped copy quietly fixes what the doc got wrong), and treat "denied" and "not offered" as different states — `.permission_denials` only records denials, so it can never tell you the subagent was over-privileged.
