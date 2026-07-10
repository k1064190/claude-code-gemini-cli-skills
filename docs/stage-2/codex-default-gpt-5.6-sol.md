# Stage 2 — bump codex-subagent default model to gpt-5.6-sol

## Why
OpenAI released the GPT-5.6 family (tiers: Sol flagship / Terra / Luna) on 2026-07-09.
The skill pins its default model explicitly (by design — nothing auto-tracks new releases),
so it kept using `gpt-5.5` until bumped. Doctor Cho asked whether new models are picked up
automatically; the answer was no, on two layers (skill default + `~/.codex/config.toml`),
plus the installed CLI (0.139.0) predated the model.

## What
- Upgraded codex CLI 0.139.0 → 0.144.1 (`codex update`); 0.139.0 rejected the model with
  400 `"requires a newer version of Codex"`.
- `~/.codex/config.toml`: `model = "gpt-5.5"` → `"gpt-5.6-sol"`.
- `codex-subagent/SKILL.md`: default → `gpt-5.6-sol` (effort `high` unchanged); model list
  refreshed with the three 5.6 tiers; canonical example and prose updated; added two caveats —
  never use the bare `gpt-5.6` alias (400 `"not supported when using Codex with a ChatGPT
  account"` on ChatGPT-account auth), and `"requires a newer version"` → run `codex update`.
- Strategy decision (user): the preferred "alias semi-auto" option (`-m gpt-5.6`) was
  empirically blocked on ChatGPT-account auth, so fell back to the explicit `gpt-5.6-sol` pin;
  revisit if OpenAI enables the alias for ChatGPT accounts.

## How
Every claim verified against the live CLI before editing: alias rejected (400), explicit
`gpt-5.6-sol`/`-terra`/`-luna` all return OK, efforts `high` and `xhigh` valid on sol, and a
no-`-m` run resolves to sol via the updated config. All tests used the stage-1 safe pattern
(`< /dev/null`, `$ERRLOG`, `timeout -k`, file-redirect) — the alias failure surfaced instantly
as a clean 400 instead of a silent freeze, validating the stage-1 work.

## Code locations
- `codex-subagent/SKILL.md` — step 1 (default + model list + alias/upgrade caveats),
  "Avoiding hangs" point 2 (model name), canonical safe-invocation block (`-m gpt-5.6-sol`).
- `~/.codex/config.toml` (outside repo) — `model = "gpt-5.6-sol"`.

## Review loop
All three reviewers ran on the diff and returned clean: `code-reviewer-pro` (0 issues — checked
no stale `gpt-5.5` default remains), `codex` (no blocking issues; the review itself executed on
the new `gpt-5.6-sol` default), `agy` (no blocking issues). Nothing to fix or dismiss.

## Retrospective
"Is it automatic?" was the right question — the pin is deliberate, but it lives in *three*
places (skill, config.toml, CLI version), and all three had to move together. Carry forward:
when bumping a model default, test the exact id on the installed CLI first; vendor aliases can
be auth-tier-gated even when the underlying model works.
