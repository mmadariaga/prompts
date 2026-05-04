# shared-ai

Comandos de Claude Code, GitHub Copilot y opencode para un flujo de trabajo de desarrollo estructurado: **spec → plan → implement → review → audits → PR**.

Cada comando es un wrapper fino que descarga sus instrucciones desde `instructions/` en este repo y las ejecuta en *Isolation Mode* (sin herencia de contexto previo). Cada fase produce un artefacto en `plans/{feature-name}/` que alimenta a la siguiente.

## Pipeline secuencial (numerado)

| Comando | Claude Code | GitHub Copilot | Opencode Go | Entrada | Salida | Propósito |
|---------|-------------|----------------|----------|---------|--------|-----------|
| `/ai-1-spec` | claude-opus-4-7 | Claude Opus 4.6 (copilot) | opencode-go/kimi-k2.6 | feature description | `plans/{f}/spec.md` | Deconstruye la feature en pasos testables, decisiones de diseño, perfil de experto, docs requeridas |
| `/ai-2-plan` | claude-sonnet-4-6 | Claude Sonnet 4.6 (copilot) | opencode-go/kimi-k2.6 | `spec.md` | `plans/{f}/plan.md` | Plan de implementación con código, checkboxes, verificación automatizada/humana, STOP & COMMIT por step |
| `/ai-3-implement` | claude-haiku-4-5 | GPT-5 mini (copilot) | opencode-go/deepseek-v4-flash | `plan.md` | código | Ejecuta el plan paso a paso, marca checkboxes, pide autorización para git ops |
| `/ai-4-review` | claude-sonnet-4-6 | Claude Sonnet 4.6 (copilot) | opencode-go/kimi-k2.6 | `spec.md` + diff | `plans/{f}/review.md` | Code review holístico (correctness, maintainability, testing, consistency). **Router de triage** → recomienda los siguientes audits si la superficie cambió |
| `/ai-5-security` | claude-opus-4-7 | Claude Opus 4.6 (copilot) | opencode-go/kimi-k2.6 | `spec.md` + diff | `plans/{f}/security.md` | SAST + SCA, mapeo CWE/CVE, OWASP/PCI/GDPR. file:line + taint flow obligatorios |
| `/ai-6-performance` | claude-sonnet-4-6 | Claude Sonnet 4.6 (copilot) | opencode-go/kimi-k2.6 | `spec.md` + diff | `plans/{f}/performance.md` | Audit por tier (backend / frontend / db / queue). Evidence-based, sin especulación |
| `/ai-7-accessibility` | claude-sonnet-4-6 | Claude Sonnet 4.6 (copilot) | opencode-go/kimi-k2.6 | `spec.md` + diff | `plans/{f}/accessibility.md` | WCAG 2.2 AA estático (+ axe/Lighthouse opcional con `--runtime`) |

## Comandos on-demand (sin número)

| Comando | Claude Code | GitHub Copilot | opencode | Propósito |
|---------|-------------|----------------|----------|-----------|
| `/ai-commit` | claude-haiku-4-5 | GPT-5 mini (copilot) | opencode-go/deepseek-v4-flash | Genera mensaje Conventional Commits desde `git diff --cached`. Subject ≤50 chars, body solo cuando el *why* no es obvio. `git commit` con autorización explícita |
| `/ai-pr` | claude-haiku-4-5 | GPT-5 mini (copilot) | opencode-go/deepseek-v4-flash | Sintetiza título + body de PR desde spec/plan/review/security/performance/accessibility + git log. Guarda draft y abre PR vía `gh` con autorización explícita |

## Triage en `ai-4-review`

`ai-4-review` no hace SAST profundo, profiling ni axe. Detecta superficie tocada y recomienda el audit específico:

- **Security surface** (auth, input parsing, dynamic queries, crypto, HTTP boundary, deps, logging) → `/ai-5-security`
- **Performance surface** (queries nuevas, endpoints, consumers, hot components, deps, loops sobre input no acotado, caching) → `/ai-6-performance`
- **Accessibility surface** (`.tsx`/`.jsx`/`.astro`/`.html`/`.vue`/`.svelte`/`.css`) → `/ai-7-accessibility`

## Instalación global (multi-proyecto)

Los comandos están pensados como **globales del usuario**, no por proyecto. Una sola copia en el directorio global del CLI los hace disponibles en cualquier repo.

### Claude Code

| OS | Destino |
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

| OS | Destino |
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

> Comandos por proyecto siguen siendo posibles vía `.claude/commands/` o `.opencode/command/` en la raíz del repo — útil cuando un proyecto necesita variantes específicas. Los globales actúan como base; los locales sobreescriben por nombre.

## Uso típico

```
/ai-1-spec Añadir autenticación con OAuth2
/ai-2-plan plans/oauth2-auth/spec.md
/ai-3-implement plans/oauth2-auth/plan.md

# git add ... && /ai-commit  (después de cada step / STOP & COMMIT del plan)

/ai-4-review plans/oauth2-auth/spec.md
# Si ai-4-review recomienda audits:
/ai-5-security plans/oauth2-auth/spec.md
/ai-6-performance plans/oauth2-auth/spec.md
/ai-7-accessibility plans/oauth2-auth/spec.md

/ai-pr plans/oauth2-auth/spec.md
```

> **Importante:** abre un chat nuevo entre comando y comando. Motivos:
> - **Ahorro de tokens** — cada fase hereda solo el artefacto que necesita, no el historial completo.
> - **Contexto limpio y replicable** — cada fase arranca desde cero (Isolation Mode), lo que facilita depurar y repetir pasos de forma aislada.
> - **Aislamiento de modelo** — cada fase usa el modelo más coste-efectivo para su tarea.

## Convenciones

- **Numerados (1→7)** = pipeline secuencial. Orden esperado.
- **Sin número** (`ai-pr`, `ai-commit`) = se ejecuta en posición variable.
- Todos los audits (5, 6, 7) requieren `spec.md` y respetan trade-offs explícitamente aceptados (los marcan como *Acknowledged*, no como findings).
- Todos los audits son diff-scoped por defecto vs rama padre. Soportan `--full` o `--path {dir}` para ampliar el alcance.
- Ningún audit modifica código. Solo escribe en `plans/{feature-name}/`.

## Estructura del repo

```
instructions/      ← contenido real de cada agente (markdown plano, Isolation Mode + TASK)
claude/commands/   ← wrappers para Claude Code (modelo + effort + fetch a instructions/)
opencode/commands/ ← wrappers para opencode (modelo + fetch a instructions/)
```
