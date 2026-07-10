# Docs index

## Stage 1 — codex/agy subagent freeze fix
- [subagent-stdin-hang-fix](stage-1/subagent-stdin-hang-fix.md) — hardened the `codex-subagent` and `antigravity-subagent` skills against the "freeze / no response" class: stdin redirect (`< /dev/null`), stderr capture with surfacing, hard `timeout -k` bound, file-redirect capture for agy (command-substitution pipe trap), and the "don't make agy hunt for files" caveat.

## Stage 2 — codex default model bump to gpt-5.6-sol
- [codex-default-gpt-5.6-sol](stage-2/codex-default-gpt-5.6-sol.md) — upgraded codex CLI to 0.144.1 and moved the skill + `~/.codex/config.toml` default from `gpt-5.5` to `gpt-5.6-sol`; documented the 5.6 tier list and the "bare `gpt-5.6` alias is rejected on ChatGPT-account auth" caveat.
