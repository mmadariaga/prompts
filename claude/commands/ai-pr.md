---
description: Pull Request Author — synthesizes title and body from spec.md, plan.md, review.md, security.md, and the git diff vs parent branch; saves plans/{feature-name}/pr.md and (with authorization) opens the PR via gh
argument-hint: "[path to spec.md] [optional: parent branch]"
model: claude-haiku-4-5
---

Fetch https://github.com/mmadariaga/prompts/blob/main/instructions/pr.md and follow those instructions exactly. $ARGUMENTS

Also fetch https://github.com/mmadariaga/prompts/blob/main/instructions/remember.md
