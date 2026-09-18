# easy-powershell-skill

**Languages:** [English](README.en.md) · [中文](README.md)

> Command-execution rules and quick reference for AI agents working in Windows PowerShell. Select the correct native command on the first attempt, rather than trying a Linux command, hitting an error, and then reworking.

When working on Windows, AI agents habitually reach for `grep`, `ls -la`, and `rm -rf`, and only switch to `Select-String`, `Get-ChildItem`, and `Remove-Item` after they fail. This skill codifies "select the right command up front" into hard rules, and consolidates high-frequency command translations, syntax traps, invocation patterns, and encoding issues into reference files that load on demand.

## Features

- **Rule-driven**: five hard rules — no Unix commands, no aliases, no retrying a failure unchanged — stop the trial-and-error loop at its source.
- **Minimal core**: `SKILL.md` fits on a single page and covers the core rules, a quick reference, and the mandatory safeguards for destructive operations.
- **On-demand references**: six files cover translations, traps, external invocation, encoding, safety, and Windows administration; each is read only when needed, so no context is wasted.
- **Field-tested traps**: parenthesized operators, `foreach` pipeline parsing, `ConvertTo-Json -Depth`, `$?` versus `$LASTEXITCODE`, and compact arguments split by native programs — the pitfalls agents actually hit.
- **Safety guardrails**: a high-impact operation confirmation table, recursive-delete preflight, one-shot execution-policy bypass, and failure classification and retry discipline.
- **CJK-aware**: encoding guidance for Chinese output and Chinese paths (UTF-8 with BOM, `PYTHONIOENCODING`, `$OutputEncoding`).

## Directory Layout

```
easy-powershell-skill/
├── SKILL.md                        # Core: hard rules + quick reference + destructive-operation safeguards
├── README.md                       # Main README (Chinese)
├── README.en.md                    # This file (English)
└── references/
    ├── bash-to-powershell.md       # bash → PowerShell → CMD translation tables + error triage
    ├── pitfalls.md                 # 14 syntax traps (parentheses, JSON depth, null values, exit codes, and more)
    ├── native-invocation.md        # External executable calls, argument passing, temporary .ps1, Start-Process
    ├── encoding.md                 # CJK output, BOM, PowerShell 5.1 / 7 differences
    ├── safety.md                   # High-impact confirmation, recursive-delete preflight, retry discipline
    └── windows-admin.md            # Registry, services, network, users, scheduled tasks, event logs
```

## Installation

### Option A: `npx skills` (recommended)

```bash
npx skills add <this-repo-url> --skill easy-powershell-skill
```

### Option B: Copy into a project's skills directory

To have the agent follow these rules **inside a specific Windows project**, copy the entire `easy-powershell-skill/` directory into that project's agent skills directory:

| Agent | Project-level path |
|---|---|
| Claude Code | `<project>/.claude/skills/easy-powershell-skill/` |
| Codex | `<project>/.codex/skills/easy-powershell-skill/` |
| Cursor | `<project>/.cursor/skills/easy-powershell-skill/` |
| Other frameworks | Any location where your agent discovers `SKILL.md` |

A project-level installation travels with the repository: once the skill is committed, every agent that opens the project follows these rules automatically.

### Option C: Copy into the user-level skills directory

| Agent | User-level path |
|---|---|
| Claude Code | `~/.claude/skills/easy-powershell-skill/` |
| Codex | `~/.codex/skills/easy-powershell-skill/` |

Restart the agent session after installation so the skill index refreshes.

## Usage

The skill triggers automatically in the scenarios described in the `description` field of `SKILL.md`:

- Running commands in a Windows terminal, or writing and debugging PowerShell scripts
- Translating bash/Linux commands into PowerShell
- Troubleshooting commands that fail on Windows
- Destructive operations such as deletion, overwriting, stopping services, and registry changes

Routine commands (directory listing, `git status`, version checks) execute directly without triggering the skill.

## Design Principles

1. **The core is discipline.** `SKILL.md` keeps only what must be followed every time, and fits on a single page.
2. **Extensions are manuals.** Every detail lives in `references/`, where each file has an explicit trigger scenario and is read on demand.
3. **Personal and trimmable.** The skill is built for personal use: dropping references you do not need leaves the core intact, and finer administration commands can be appended to `windows-admin.md` as required.

## Credits

This skill synthesizes guidance from the following open-source projects (all MIT licensed):

- [jsrgjcy/powershell-windows-skill](https://github.com/jsrgjcy/powershell-windows-skill) — high-impact operation confirmation flow, 12-domain administration reference
- [UncertaintyDeterminesYou4ndMe/powershell-windows-cli-agent-skill](https://github.com/UncertaintyDeterminesYou4ndMe/powershell-windows-cli-agent-skill) — bash-to-PowerShell translation, `-WhatIf` safety patterns
- [agent-shells/powershell-skills](https://github.com/agent-shells/powershell-skills) — pattern catalog and a corpus of real failures
- [Misaka-Mikoto-Tech/agent-skills](https://github.com/Misaka-Mikoto-Tech/agent-skills) — `powershell-safe-invocation` native invocation patterns
- [sickn33/antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills) — operator parentheses, ASCII-only output, null-check traps
- [GuanKr/pwsh-pitfalls](https://github.com/GuanKr/pwsh-pitfalls) — the two exit-code systems, the `find` trap, encoding boundaries
- [dfinke/powershell-ai-skills](https://github.com/dfinke/powershell-ai-skills) — PowerShell maintenance practices

## License

MIT
