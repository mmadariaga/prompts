# Claude Code — Installation

| OS | Destination |
|----|---------|
| Linux / macOS | `~/.claude/commands/` |
| Windows | `%USERPROFILE%\.claude\commands\` |

**Linux / macOS:**
```bash
mkdir -p ~/.claude/commands
cp claude/commands/*.md ~/.claude/commands/

# Copy instructions
if [ -d ~/.claude/commands/instructions ]; then
    echo "Overwriting ~/.claude/commands/instructions/"
fi
mkdir -p ~/.claude/commands/instructions
cp instructions/*.md ~/.claude/commands/instructions/
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\commands"
Copy-Item claude\commands\*.md "$env:USERPROFILE\.claude\commands\"

# Copy instructions
$instructionsDir = "$env:USERPROFILE\.claude\commands\instructions"
if (Test-Path $instructionsDir) {
    Write-Host "Overwriting $instructionsDir"
}
New-Item -ItemType Directory -Force -Path $instructionsDir | Out-Null
Copy-Item instructions\*.md $instructionsDir\
```
