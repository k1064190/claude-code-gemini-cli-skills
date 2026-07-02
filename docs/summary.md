# Docs index

## Stage 1 — codex/agy subagent freeze fix
- [subagent-stdin-hang-fix](stage-1/subagent-stdin-hang-fix.md) — hardened the `codex-subagent` and `antigravity-subagent` skills against the "freeze / no response" class: stdin redirect (`< /dev/null`), stderr capture with surfacing, hard `timeout -k` bound, file-redirect capture for agy (command-substitution pipe trap), and the "don't make agy hunt for files" caveat.
