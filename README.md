# shared-ai

Claude Code, GitHub Copilot and opencode commands for a structured development workflow: **spec → plan → implement → review → audits → PR**.

Each command is a thin wrapper that fetches its instructions from `instructions/` in this repo and runs them in *Isolation Mode* (no inherited context from previous chats). Each phase produces an artifact in `plans/{feature-name}/` that feeds into the next.

## Sequential pipeline (numbered)

| Command | Recommended models | Input | Output | Purpose |
|---------|---------|---------|--------|-----------|
| `/ai-1-spec` | **claude-opus-4-7**<br>\| Claude Opus 4.6 (Copilot)<br>\| opencode-go/kimi-k2.6 | feature description | `plans/{f}/spec.md` | Deconstructs the feature into testable steps, design decisions, expert profile, required docs |
| `/ai-2-plan` | **opencode-go/kimi-k2.6**<br>\| claude-sonnet-4-6<br>\| Claude Sonnet 4.6 (Copilot) | `spec.md` | `plans/{f}/plan.md` | Implementation plan with code, checkboxes, automated/human verification, STOP & COMMIT per step |
| `/ai-3-implement` | **opencode-go/deepseek-v4-flash**<br>\| claude-haiku-4-5<br>\| GPT-5 mini (Copilot) | `plan.md` | code | Executes the plan step by step, checks off boxes, asks for authorization on git ops |
| `/ai-4-review` | **opencode-go/kimi-k2.6**<br>\| claude-sonnet-4-6<br>\| Claude Sonnet 4.6 (Copilot) | `spec.md` + diff | `plans/{f}/review.md` | Holistic code review (correctness, maintainability, testing, consistency). **Triage router** → recommends follow-up audits if surface changed |
| `/ai-5-security` | **claude-opus-4-7**<br>\| Claude Opus 4.6 (Copilot)<br>\| opencode-go/kimi-k2.6 | `spec.md` + diff | `plans/{f}/security.md` | SAST + SCA, CWE/CVE mapping, OWASP/PCI/GDPR. file:line + taint flow required |
| `/ai-6-performance` | **opencode-go/kimi-k2.6**<br>\| claude-sonnet-4-6<br>\| Claude Sonnet 4.6 (Copilot) | `spec.md` + diff | `plans/{f}/performance.md` | Audit by tier (backend / frontend / db / queue). Evidence-based, no speculation |
| `/ai-7-accessibility` | **opencode-go/kimi-k2.6**<br>\| claude-sonnet-4-6<br>\| Claude Sonnet 4.6 (Copilot) | `spec.md` + diff | `plans/{f}/accessibility.md` | Static WCAG 2.2 AA (+ optional axe/Lighthouse with `--runtime`) |

## On-demand commands (unnumbered)

| Command | Models | Purpose |
|---------|---------|-----------|
| `/ai-commit` | - claude: claude-haiku-4-5<br>- copilot: GPT-5 mini<br>- opencode: opencode-go/deepseek-v4-flash | Generates a Conventional Commits message from `git diff --cached`. Subject ≤50 chars, body only when the *why* is not obvious. `git commit` with explicit authorization |
| `/ai-pr` | - claude: claude-haiku-4-5<br>- copilot: GPT-5 mini<br>- opencode: opencode-go/deepseek-v4-flash | Synthesizes PR title + body from spec/plan/review/security/performance/accessibility + git log. Saves draft and opens PR via `gh` with explicit authorization |

## Triage in `ai-4-review`

`ai-4-review` does not perform deep SAST, profiling, or axe analysis. It detects the touched surface and recommends the specific audit:

- **Security surface** (auth, input parsing, dynamic queries, crypto, HTTP boundary, deps, logging) → `/ai-5-security`
- **Performance surface** (new queries, endpoints, consumers, hot components, deps, loops over unbounded input, caching) → `/ai-6-performance`
- **Accessibility surface** (`.tsx`/`.jsx`/`.astro`/`.html`/`.vue`/`.svelte`/`.css`) → `/ai-7-accessibility`

## Global installation (multi-project)

Commands are designed as **user globals**, not per project. A single copy in the CLI's global directory makes them available in any repo.

### Claude Code

| OS | Destination |
|----|---------|
| Linux / macOS | `~/.claude/commands/` |
| Windows | `%USERPROFILE%\.claude\commands\` |

**Linux / macOS:**
```bash
mkdir -p ~/.claude/commands
cp claude/commands/*.md ~/.claude/commands/
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\commands"
Copy-Item claude\commands\*.md "$env:USERPROFILE\.claude\commands\"
```

### opencode

| OS | Destination |
|----|---------|
| Linux / macOS | `~/.config/opencode/command/` |
| Windows | `%APPDATA%\opencode\command\` |

**Linux / macOS:**
```bash
mkdir -p ~/.config/opencode/command
cp opencode/commands/*.md ~/.config/opencode/command/
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$env:APPDATA\opencode\command"
Copy-Item opencode\commands\*.md "$env:APPDATA\opencode\command\"
```

> Per-project commands are still possible via `.claude/commands/` or `.opencode/command/` at the repo root — useful when a project needs specific variants. Globals act as a base; locals override by name.

## Typical usage

```
/ai-1-spec Add OAuth2 authentication
/ai-2-plan plans/oauth2-auth/spec.md
/ai-3-implement plans/oauth2-auth/plan.md

# git add ... && /ai-commit  (after each step / STOP & COMMIT in the plan)

/ai-4-review plans/oauth2-auth/spec.md
# If ai-4-review recommends audits:
/ai-5-security plans/oauth2-auth/spec.md
/ai-6-performance plans/oauth2-auth/spec.md
/ai-7-accessibility plans/oauth2-auth/spec.md

/ai-pr plans/oauth2-auth/spec.md
```

> **Important:** open a new chat between commands. Reasons:
> - **Token savings** — each phase only inherits the artifact it needs, not the full history.
> - **Clean, replicable context** — each phase starts from scratch (Isolation Mode), making it easy to debug and replay steps in isolation.
> - **Model isolation** — each phase uses the most cost-effective model for its task.

## Conventions

- **Numbered (1→7)** = sequential pipeline. Expected order.
- **Unnumbered** (`ai-pr`, `ai-commit`) = run at variable position.
- All audits (5, 6, 7) require `spec.md` and respect explicitly accepted trade-offs (marked as *Acknowledged*, not as findings).
- All audits are diff-scoped by default vs parent branch. Support `--full` or `--path {dir}` to expand scope.
- No audit modifies code. Writes only to `plans/{feature-name}/`.

## Repo structure

```
instructions/      ← actual content for each agent (plain markdown, Isolation Mode + TASK)
claude/commands/   ← wrappers for Claude Code (model + effort + fetch to instructions/)
opencode/commands/ ← wrappers for opencode (model + fetch to instructions/)
```
