# Gemini 3.8 Flash writing skill research

## Research question

Does a public Codex/Agent Skill already exist whose specific job is to invoke Google Antigravity CLI (`agy`) with Gemini 3.8 Flash for writing-only work such as drafting, rewriting, copyediting, tone adjustment, or translation, rather than for general coding or agent delegation?

## Scope and decision informed

This investigation covers:

- public `SKILL.md` files and public skill registries discoverable on the web;
- official Google Antigravity documentation for current model availability and headless CLI syntax;
- the locally installed `agy` executable and the repository's existing general-purpose `antigravity-subagent` skill;
- whether this repository should add a separate writing skill or expand the general delegation skill.

“Exact” has two useful interpretations:

1. **Functional match:** a skill is narrowly about writing, invokes `agy`, and selects Gemini 3.8 Flash in the current environment.
2. **Literal trigger match:** its metadata explicitly names all of drafting, rewriting/copyediting, tone work, and translation, as well as Gemini 3.8 Flash.

The decision is whether to create a separate narrow skill, reuse/fork a public one, or adapt `antigravity-subagent`.

## Investigation date and environment

- Investigation: `2026-09-22` (commands captured at `2026-09-22T01:59:19Z`)
- Host: `Amazon Linux 2023.12.20260720`
- Kernel/architecture: `Linux 6.12.94-123.190.amzn2023.x86_64`, `x86_64`
- Repository: `/home/ec2-user/projects/claude-code-gemini-cli-skills`
- Branch: `feature/gemini-38-flash-writing`
- `agy` executable: `/home/ec2-user/.local/bin/agy`
- `agy` version: `1.2.7`
- Executable SHA-256: `9991515b6d5307bcf701069622b0537b6b206e605f3c891c0cf3a3d208dea8b0`
- No model request was sent. Only `agy --version`, `agy models`, and `agy --help` were run.
- No local lint or tests were run; project instructions prohibit them.

## Existing repository evidence

No prior file existed under `docs/*/research/*.md` when the investigation began.

The repository's [`antigravity-subagent/SKILL.md`](../../../antigravity-subagent/SKILL.md) is intentionally general-purpose:

- its metadata delegates any task to `agy` and emphasizes general second opinions, parallel execution, and large-codebase analysis (lines 2–8);
- it defaults every task to `Gemini 3.1 Pro (High)` and lists Gemini 3.5 Flash, not 3.8 Flash (lines 42–57);
- it says JSON output is unavailable (lines 61–69), while installed `agy 1.2.7 --help` exposes `--output-format text|json|stream-json`;
- it says `--print-timeout` defaults to five minutes (lines 21 and 110), while installed `agy 1.2.7 --help` reports `default 0s`.

Those are verified local-version discrepancies, not a claim about all released `agy` versions. The official online headless documentation still reports a five-minute default, so the documentation and installed `1.2.7` binary disagree on this point.

## Exact local command evidence

Commands were run from the repository root.

### Version

```console
$ agy --version
1.2.7
```

### Available models

```console
$ agy models
Fetching available models...
gemini-3.8-flash-high    Gemini 3.8 Flash (High)
gemini-3.8-flash-medium  Gemini 3.8 Flash (Medium)
gemini-3.8-flash-low     Gemini 3.8 Flash (Low)
gemini-3.7-flash-high    Gemini 3.7 Flash (High)
gemini-3.7-flash-medium  Gemini 3.7 Flash (Medium)
gemini-3.7-flash-low     Gemini 3.7 Flash (Low)
gemini-3.6-flash-high    Gemini 3.6 Flash (High)
gemini-3.6-flash-medium  Gemini 3.6 Flash (Medium)
gemini-3.6-flash-low     Gemini 3.6 Flash (Low)
gemini-3.1-pro-high      Gemini 3.1 Pro (High)
gemini-3.1-pro-low       Gemini 3.1 Pro (Low)
claude-sonnet-4-6        Claude Sonnet 4.6 (Thinking)
claude-opus-4-6-thinking Claude Opus 4.6 (Thinking)
gpt-oss-120b-medium      GPT-OSS 120B (Medium)
```

Whitespace above is normalized to a single visual separator; the IDs and display names are exact. The installed output establishes three exact 3.8 Flash IDs, including `gemini-3.8-flash-medium`.

### Relevant help output

```console
$ agy --help
Usage of agy:
  --add-dir                       Add a directory to the workspace (repeatable) (default [])
  --dangerously-skip-permissions  Auto-approve all tool permission requests without prompting
  --disable-slash-commands        Disable slash command and skill expansion in print mode
  --effort                        Reasoning effort for the current CLI session (low|medium|high)
  --input-format                  Input format for print mode (text, stream-json). stream-json reads one NDJSON message per line from stdin and runs a turn for each; it requires --output-format stream-json (default text)
  --json-schema                   Optional JSON schema string or path to a schema file to enforce structured output (for stream-json, only applicable to the final result)
  --mode                          Set the agent execution mode for this session (accept-edits, plan)
  --model                         Model for the current CLI session
  --output-format                 Output format for print mode (text, json, stream-json) (default text)
  -p                              Short alias for --print
  --print                         Run a single prompt non-interactively and print the response
  --print-timeout                 Optional time limit for print mode; 0 waits until the turn completes (default 0s)
  --sandbox                       Run in a sandbox with terminal restrictions enabled
```

## Web searches performed

Searches were performed on `2026-09-22`. Exact queries:

1. `Google Antigravity CLI agy Gemini 3.8 Flash official`
2. `site:developers.google.com antigravity CLI agy models`
3. `site:github.com "Gemini 3.8 Flash" "SKILL.md" agy`
4. `site:skills.sh agy "Gemini 3.8 Flash" writing skill`
5. `site:github.com/*/SKILL.md "agy" (writing OR rewriting OR copyediting OR translation)`
6. `site:github.com "agy" "copyediting" "SKILL.md"`
7. `site:github.com "Gemini 3.8 Flash" (drafting OR rewriting OR copyediting OR tone OR translation) skill`
8. `site:skills.sh (agy OR Antigravity) (writing OR copyediting OR translation)`
9. `"gemini-write" "agy" skill`
10. `site:skills.sh "gemini-write"`
11. `site:skillhub.club "agy" "writing"`
12. `site:skillsmp.com "agy" "Gemini 3.8 Flash"`
13. `site:skills.sh Antigravity agy "Gemini 3.8 Flash" writing skill`
14. `site:github.com SKILL.md "Gemini 3.8 Flash" agy`
15. `site:github.com "agy" "Gemini 3.8 Flash" drafting rewriting`
16. `site:gist.github.com SKILL.md agy "drafting" "gemini-3.8-flash"`
17. `"gemini-3.8-flash" "proofreading" "agy --model"`
18. `"Gemini 3.8 Flash" "translation" "agy" "SKILL.md"`
19. `site:github.com/wildriver/agy-skills "SKILL.md" "agy-text"`
20. `site:github.com/wildriver/agy-skills "SKILL.md" "agy-review"`

Additional exact retrieval command:

```console
$ curl -fsSL https://api.github.com/gists/06f57b212db3a96bc57bfe8c4816e3c1
```

Relevant API fields, transcribed exactly:

```text
id: 06f57b212db3a96bc57bfe8c4816e3c1
public: true
created_at: 2026-09-13T09:46:16Z
updated_at: 2026-09-17T12:27:13Z
file: SKILL.md
raw revision URL SHA: 92a0cfeae37e6674d023e15038d00455dad5b12b
latest history version: 4bde90e02ce46363a703b215001652dfe7aa7ff0
```

The strongest current match was then verified directly at an immutable commit:

```console
$ git ls-remote https://github.com/wildriver/agy-skills.git HEAD
0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2 HEAD

$ curl -fsSL https://raw.githubusercontent.com/wildriver/agy-skills/0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2/plugins/agy-skills/skills/agy-text/SKILL.md
# relevant exact fields translated below
name: agy-text
default model: gemini-3.8-flash-high
writing cases: drafts, email, summaries, translations, titles/copy, FAQ, structured JSON

$ curl -fsSL https://raw.githubusercontent.com/wildriver/agy-skills/0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2/plugins/agy-skills/skills/agy-review/SKILL.md
# relevant exact fields translated below
name: agy-review
default model: gemini-3.8-flash-high
modes: proofread | critique | rewrite | custom
custom examples: translation checking, reviewer role, summarization, title ideas

$ curl -fsSL https://raw.githubusercontent.com/wildriver/agy-skills/0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2/LICENSE
MIT License
Copyright (c) 2026 wildriver
```

The original files are Japanese. The English descriptions above are translations for indexing; the source links below preserve exact text. Direct inspection of the helper scripts confirmed both build an argument vector containing `--output-format json`, an explicit timeout, `--model <selected model>`, and `-p=<prompt>`, then invoke it with Python `subprocess.run`.

## Sources

All sources were accessed `2026-09-22`.

| Source | Type | What it establishes |
|---|---|---|
| [Google Antigravity: Headless mode](https://www.antigravity.google/docs/cli/headless/) | Official Google documentation | `agy models`; model slugs including `gemini-3.8-flash-high` and `gemini-3.8-flash-medium`; `agy -p`; `--model`; `--effort`; output formats; exit/status handling; permission behavior. |
| [Google Antigravity: Models](https://www.antigravity.google/docs/models/) | Official Google documentation | Gemini 3.8 Flash is an available Antigravity reasoning model across listed plans. |
| [Google Antigravity changelog](https://www.antigravity.google/changelog) | Official Google changelog | Gemini 3.8 Flash availability and its August/September 2026 rollout context. |
| [`wildriver/agy-skills`](https://github.com/wildriver/agy-skills/tree/0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2) | Public MIT-licensed primary artifact | Current repository containing dedicated generation and writing-review skills; commit pinned for reproducibility. |
| [`agy-text/SKILL.md`](https://github.com/wildriver/agy-skills/blob/0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2/plugins/agy-skills/skills/agy-text/SKILL.md) and [helper](https://github.com/wildriver/agy-skills/blob/0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2/plugins/agy-skills/skills/agy-text/scripts/agy_text.py) | Public primary artifact | Defaults to `gemini-3.8-flash-high`; narrowly generates drafts, emails, summaries, translations, titles/copy, FAQ, and structured text through `agy`. |
| [`agy-review/SKILL.md`](https://github.com/wildriver/agy-skills/blob/0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2/plugins/agy-skills/skills/agy-review/SKILL.md) and [helper](https://github.com/wildriver/agy-skills/blob/0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2/plugins/agy-skills/skills/agy-review/scripts/agy_review.py) | Public primary artifact | Defaults to `gemini-3.8-flash-high`; covers proofreading, critique, rewrite, tone/context-sensitive work, and translation checking through `agy`. |
| [`wildriver/agy-skills` MIT license](https://github.com/wildriver/agy-skills/blob/0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2/LICENSE) | Public primary artifact | Permits use and modification subject to retaining the copyright and permission notice. |
| [laiso/gemini-write `SKILL.md` gist](https://gist.github.com/laiso/06f57b212db3a96bc57bfe8c4816e3c1) | Public primary artifact | A narrow public writing skill exists and invokes authenticated `agy`; current revision delegates drafting, rewriting, or proofreading and dynamically prefers the newest available Gemini Flash at Medium reasoning. |
| [Current raw `gemini-write` revision](https://gist.githubusercontent.com/laiso/06f57b212db3a96bc57bfe8c4816e3c1/raw/92a0cfeae37e6674d023e15038d00455dad5b12b/SKILL.md) | Public primary artifact, revision-pinned | Exact current skill content returned by the GitHub Gist API. |
| [richfrem `agy-cli-agent/SKILL.md`](https://github.com/richfrem/agent-plugins-skills/blob/main/plugins/cli-agents/skills/agy-cli-agent/SKILL.md) | Public primary artifact | Supports `agy` and Gemini 3.8 Flash, but is a general sub-agent dispatcher oriented toward agentic/coding tasks, not a writing-only skill. |
| [OctavianTocan `agy-review/SKILL.md`](https://github.com/OctavianTocan/agy-review/blob/main/SKILL.md) | Public primary artifact | Can critique a piece of writing, but it is an adversarial review skill, defaults to Gemini 3.5 Flash in the inspected revision, and does not draft/rewrite/translate. |
| [SkillHub: `agy-delegate`](https://www.skillhub.club/skills/amelnagdy-delegate-skills-agy-delegate) | Public registry entry | Registry evidence for a coding-task delegation skill; not a writing-only match. |
| [`google/skills` Gemini API skill](https://github.com/google/skills/blob/main/skills/cloud/gemini-api/SKILL.md) | Official Google repository | Recommends Gemini 3.8 Flash for API use, but calls the Gemini API rather than `agy` and is not a writing-only delegation skill. |

## Findings

### Verified facts

1. **Current public, literally specific skills exist.** At immutable commit `0d4a2fa833fbc3a86d38eba868df9b4bc5c0f2b2`, the MIT-licensed `wildriver/agy-skills` repository contains `agy-text` and `agy-review`. Both explicitly invoke `agy`, default to `gemini-3.8-flash-high`, and are confined to text generation or writing review rather than coding delegation.
2. **Together they cover the requested writing-only scope explicitly.** `agy-text` names drafts, emails, summaries, translation, titles/copy, FAQ, and structured text. `agy-review` names proofreading, critique, rewriting, audience/purpose/style context, and translation checking. The helper scripts actually execute `agy` with the chosen model and parse its JSON envelope.
3. **The public `gemini-write` Gist is a second, narrower precedent.** Its description covers drafting, rewriting, and proofreading through authenticated `agy`. Its initial `2026-09-13` revision explicitly pinned `gemini-3.8-flash-medium`; its current revision dynamically chooses the newest available Flash at Medium reasoning, which resolves to 3.8 in this installed environment.
4. **The Gist alone is a functional but not literal all-keywords match.** It does not put `copyediting`, `tone`, or `translation` in its metadata. In contrast, the `wildriver` pair explicitly covers translation, contextual tone/style, proofreading, and rewriting.
5. **Other discovered `agy` skills are adjacent, not exact.** `agy-cli-agent` is broad agent dispatch; OctavianTocan's `agy-review` is adversarial review rather than drafting/rewrite execution and defaults to an older model in the inspected revision; `agy-delegate` is for coding implementation.
6. **Current CLI syntax supports a robust narrow workflow.** Official documentation and installed help agree on `agy -p/--print`, model selection, JSON output, sandboxing, permission controls, and nonzero error handling. The installed binary also exposes `--disable-slash-commands`.
7. **The general local skill has conflicting defaults and stale CLI facts for this use case.** It defaults to Gemini 3.1 Pro High, omits 3.8, denies JSON output support, and carries extensive agentic/coding behavior that a content-preserving writing flow does not need.
8. **Reuse paths differ legally.** `wildriver/agy-skills` is MIT-licensed and may be adapted with its notice retained. No license is shown in the `gemini-write` Gist content or API metadata inspected; public readability alone does not grant redistribution rights.

### Interpretation and inference

- Under both the **functional** and strict **literal-trigger** definitions, exact public precedents exist in the `wildriver` `agy-text` + `agy-review` pair.
- `gemini-write` remains useful design evidence for a single-skill alternative, but tone adjustment, copyediting, and translation support in that Gist is inferred from its general writing instructions rather than explicit trigger metadata.
- The `wildriver` skills are packaged for Claude Code and refer to `${CLAUDE_SKILL_DIR}`. Their standard `SKILL.md` structure is portable in principle, but direct Codex use needs path/tool-declaration adaptation and has not been verified here.
- A dedicated writing skill is a better module boundary than adding writing policy to `antigravity-subagent`: model choice, isolation, preservation of meaning, prompt/source separation, and output delivery rules differ substantially from general agent delegation.

## Decision and recommendation

**Conclusion:** exact public skills already exist. The strongest current match is the MIT-licensed `wildriver/agy-skills` pair: `agy-text` for drafting/translation/copy and `agy-review` for proofreading/rewriting/tone-sensitive review, both defaulting to `gemini-3.8-flash-high`. The public `gemini-write` Gist is a second single-skill precedent and resolves to `gemini-3.8-flash-medium` in this environment.

For this repository, **keep writing as one or more separate narrow skills rather than adapting the general `antigravity-subagent` skill**. The lowest-duplication option is to port/adapt the MIT-licensed `wildriver` skills for Codex, retaining the MIT notice and replacing Claude-specific `${CLAUDE_SKILL_DIR}`/tool metadata. If one unified skill is preferred, implement a narrow wrapper with explicit triggers for `draft`, `rewrite`, `copyedit`, tone adjustment, and translation, informed by both public precedents. Use current `agy 1.2.7` JSON/status handling and verify the requested model against `agy models`.

Do not copy or vendor the unlicensed `gemini-write` Gist unless its licensing is clarified. Keep the general skill focused on agentic delegation and update its stale model/CLI claims separately only if that work is authorized.

## Unresolved questions

1. Should this repository port the two-skill `wildriver` split (`agy-text` + `agy-review`) or expose one unified writing skill?
2. Should the repository's writing skill return generated text to the caller, edit a designated file directly, or support both modes? The `wildriver` skills return text; the current public Gist edits a designated file.
3. Should `gemini-3.8-flash-high` be pinned as in `wildriver`, `gemini-3.8-flash-medium` be used for lower reasoning cost, or should the skill dynamically follow the newest available Flash?
4. What Codex-native replacement should be used for Claude Code's `${CLAUDE_SKILL_DIR}` and `allowed-tools` declaration, and should the Python helper scripts be retained?
5. Is there an intended license for `laiso/gemini-write`, and may it be redistributed or modified in this repository?
6. Should translation preserve formatting and terminology via explicit glossaries, and should tone transformation include a required meaning-preservation check?
7. Why does official online documentation still state a five-minute `--print-timeout` default while installed `agy 1.2.7 --help` states `0s`? Explicitly setting a timeout avoids relying on either default.
