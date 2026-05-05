# shared-ai — AGENTS.md

## What is this repository

A prompt and instruction library for orchestrating a **structured AI-assisted development pipeline**.

It contains no application code. It is prompt infrastructure installed as global commands in Claude Code, opencode, and GitHub Copilot.

## Main pipeline

```
spec(1) → plan(2) → implement(3) → review(4) → [security(5) | performance(6) | accessibility(7)]
                                    ↓
                              commit / pr (on-demand)
```

Each phase produces an artifact in `plans/{feature-name}/` that feeds the next. Runs in **Isolation Mode**: every command starts with no inherited context, reading only the artifact it needs.

## Repository structure

| Directory | Purpose |
|-----------|---------|
| `instructions/` | Actual content for each phase. Plain markdown with Isolation Mode + TASK block. Fetched by wrappers. |
| `claude/commands/` | Wrappers for Claude Code. YAML frontmatter (`description`, `argument-hint`, `model`, `effort`) + fetch to `instructions/`. |
| `opencode/commands/` | Wrappers for opencode. YAML frontmatter (`description`, `model`) + fetch to `instructions/`. |
| `github/prompts/` | Prompts for GitHub Copilot. Equivalent to the commands above. |

Wrappers are **thin** — they only specify the model and fetch the markdown from `instructions/`. The logic lives in `instructions/`.

## Critical conventions

### Isolation Mode
Every file in `instructions/` starts with:
```
# Isolation Mode
- Ignore all previous conversation.
- Use only the data inside <TASK>.
- If required information is missing, ask for it.
- If you are about to use external or prior context, STOP and say: "Potential context pollution detected, stopping, open a new chat".
```
Never remove or modify this block.

### Language Policy
All agents MUST think and reason internally in English, regardless of the user's input language. This reduces token consumption (English is more token-efficient than Spanish and most other languages) and ensures consistent reasoning quality.

- **User-facing chat:** respond in the language the user writes in (default to English if unclear).
- **Generated artifacts** (`spec.md`, `plan.md`, `review.md`, `security.md`, `performance.md`, `accessibility.md`, commit messages, PR bodies, code, technical explanations): written in English unless the user explicitly requests otherwise.

### Caveman Communication Mode
All instructions include `instructions/caveman.md`. Default is **lite**: no filler, pleasantries, or hedging. Fragments allowed in `full` only for internal reasoning.
Flag `--full-caveman` in `$ARGUMENTS` activates full mode.

### GLOSSARY.md
- `spec.md`: reads `GLOSSARY.md`, updates it inline, challenges ambiguous terms.
- `plan.md`: uses glossary terms for identifiers.
- `review.md`: validates language consistency in new code.
- If it does not exist, bootstraps it with the first resolved term.
- Format: fetches `instructions/glossary-format.md`.

### RED → GREEN
Integrated in `plan.md` and `implement.md`:
- Plan includes a RED block (failing test) before GREEN (minimal implementation).
- Implement runs RED, verifies failure, writes GREEN, verifies pass.
Not pure TDD; it is test quality verification.

### ADR/DDR Proposal Check
In `plan.md` Step 1.5. Evaluates 3 criteria before generating the plan:
1. Does it affect >2 teams or systems?
2. Is it hard to revert once deployed?
3. Does it introduce a new critical dependency or change a public contract?
Only proposes creating an ADR/DDR if the project already has an ADR culture or the user approves.

### Triage in review
`ai-4-review` does not perform SAST/profiling/axe. It detects the touched surface and recommends audits:
- Security surface → `ai-5-security`
- Performance surface → `ai-6-performance`
- Accessibility surface (`.tsx`/`.jsx`/`.astro`/`.html`/`.vue`/`.svelte`/`.css`) → `ai-7-accessibility`

## Recommended models by phase

| Phase | Opencode | Claude Code | Copilot |
|-------|----------|-------------|---------|
| spec (1) | `opencode/claude-opus-4-6` | `claude-opus-4-7` High | Claude Opus 4.6 |
| plan (2) | `opencode-go/kimi-k2.6` | `claude-sonnet-4-6` | Claude Sonnet 4.6 |
| implement (3) | `opencode-go/deepseek-v4-flash` | `claude-haiku-4-5` | GPT-5 mini |
| review (4) | `opencode-go/kimi-k2.6` | `claude-sonnet-4-6` | Claude Sonnet 4.6 |
| security (5) | `opencode/gpt-5.3-codex` | `claude-opus-4-7` High | Claude Opus 4.6 |
| performance (6) | `opencode-go/kimi-k2.6` | `claude-sonnet-4-6` | Claude Sonnet 4.6 |
| accessibility (7) | `opencode-go/kimi-k2.6` | `claude-sonnet-4-6` | Claude Sonnet 4.6 |
| commit | `opencode-go/deepseek-v4-flash` | `claude-haiku-4-5` | GPT-5 mini |
| pr | `opencode-go/deepseek-v4-flash` | `claude-haiku-4-5` | GPT-5 mini |

## Installation

Commands are **user globals**, not per-project.

- **Claude Code**: `~/.claude/commands/`
- **opencode**: `~/.config/opencode/commands/`

Project-local commands (`.claude/commands/` or `.opencode/commands/` at repo root) override by name.

## Generated artifacts

```
plans/{feature-name}/
├── spec.md          # ai-1-spec
├── plan.md          # ai-2-plan
├── review.md        # ai-4-review
├── security.md      # ai-5-security (if applicable)
├── performance.md   # ai-6-performance (if applicable)
└── accessibility.md # ai-7-accessibility (if applicable)
```

## How to modify this repo

### Add / modify an instruction
1. Edit the file in `instructions/`.
2. If the recommended model changes, update the wrappers in `claude/commands/` and `opencode/commands/`.
3. Keep `github/prompts/` in sync if the change affects the base prompt.

### Add a new command
1. Create the instruction in `instructions/{name}.md` with Isolation Mode + TASK block.
2. Create wrappers in `claude/commands/ai-{name}.md` and `opencode/commands/ai-{name}.md`.
3. Optional: create prompt in `github/prompts/ai-{name}.prompt.md`.
4. Update README.md with the phase in the corresponding table.

### Format conventions
- Never use `any` in TypeScript (even though there is no TS here, it applies to code examples in instructions).
- Generated artifacts (`spec.md`, `plan.md`, etc.) are in English unless the user explicitly requests otherwise.
- Fetch URLs point to `https://github.com/mmadariaga/prompts/blob/main/instructions/...` (`experimental` branch).
