# Docs index

## Stage 1 — codex/agy subagent freeze fix
- [subagent-stdin-hang-fix](stage-1/subagent-stdin-hang-fix.md) — hardened the `codex-subagent` and `antigravity-subagent` skills against the "freeze / no response" class: stdin redirect (`< /dev/null`), stderr capture with surfacing, hard `timeout -k` bound, file-redirect capture for agy (command-substitution pipe trap), and the "don't make agy hunt for files" caveat.

## Stage 2 — codex default model bump to gpt-5.6-sol
- [codex-default-gpt-5.6-sol](stage-2/codex-default-gpt-5.6-sol.md) — upgraded codex CLI to 0.144.1 and moved the skill + `~/.codex/config.toml` default from `gpt-5.5` to `gpt-5.6-sol`; documented the 5.6 tier list and the "bare `gpt-5.6` alias is rejected on ChatGPT-account auth" caveat.

## Stage 3 — claude-subagent skill (`claude -p`)
- [claude-subagent-skill](stage-3/claude-subagent-skill.md) — added a fourth subagent skill for Claude Code itself, verified against CLI 2.1.209 rather than the docs: `--bare` is unusable on subscription auth, `--output-format json` emits a JSON *array* (so the documented `jq -r '.result'` fails), and a run can fail while exiting 0 (`.is_error`, `.permission_denials`). Canonical block gates on all three signals; a dogfood pass by a fresh `claude -p` caught five defects in the first draft.
