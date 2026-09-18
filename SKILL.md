---
name: easy-powershell-skill
description: Command-execution rules and quick reference for AI agents working in Windows PowerShell. Use when running commands in a Windows terminal, writing or debugging PowerShell scripts, translating bash/Linux commands (grep, ls, cat, rm, find, sed, awk, and similar) into PowerShell, or troubleshooting command failures on Windows. Also use before destructive operations such as deletion, overwriting, stopping services, or registry changes. Core goal: select the correct native PowerShell command on the first attempt, rather than trying a Linux command, hitting an error, and then reworking.
---

# easy-powershell-skill

Rules for executing commands in a Windows environment. **Select the right command before executing it. Do not try a Linux command first and switch only after it fails.**

## Hard Rules

1. **Never use Unix commands.** Do not use `grep`, `ls`, `cat`, `rm`, `sed`, `awk`, `head`, `tail`, `find`, `touch`, `ps`, or `kill`. Even where PowerShell defines an alias with the same name, the parameters are incompatible, so `rm -rf` and `grep -r` are guaranteed to fail.
2. **`find` is a trap.** A bare `find` resolves to Windows `find.exe`, a text-search utility, not a file-search utility. Always use `Get-ChildItem -Recurse` to locate files.
3. **No aliases in scripts or examples.** Use `Get-ChildItem` rather than `ls`, `Where-Object` rather than `?`, and `ForEach-Object` rather than `%`.
4. **Prefer the agent's built-in tools.** Use built-in tools for file and string search, directory listing, and file reading; do not reach for the command line.
5. **Never retry a failure unchanged.** Read the error first and classify it (parsing, quoting, path, or encoding), then change exactly one thing per attempt. After two failures, stop and report.

## Quick Reference

| Task | Command |
|---|---|
| List a directory / list recursively | `Get-ChildItem` / `Get-ChildItem -Recurse` |
| Find files recursively | `Get-ChildItem -Path dir -Filter '*.log' -Recurse -File` |
| Search file content | `Select-String -Path f -Pattern 'xx'` (no `-Recurse`; enumerate first, then pipe) |
| Read a file / first 10 lines / last 10 lines | `Get-Content f` / `-TotalCount 10` / `-Tail 10` |
| Delete a file / delete a directory | `Remove-Item` / `Remove-Item -Recurse -Force` (preview with `-WhatIf` first) |
| Processes / services | `Get-Process` / `Get-Service` |
| Connectivity / port | `Test-Connection` / `Test-NetConnection -Port` |
| Read an environment variable / set a session variable | `$env:NAME` / `$env:NAME = 'x'` |

## Destructive Operations (Mandatory)

1. For deletion, overwriting, stopping services, and registry changes, **preview with `-WhatIf` and execute only after confirmation.** This is mandatory without exception when `Remove-Item -Recurse` is combined with a wildcard.
2. Label operations that require administrator privileges as "run as administrator" and ask the user to confirm.
3. Bypass an execution-policy error for a single invocation only: `pwsh -ExecutionPolicy Bypass -File .\s.ps1`. Never use `Unrestricted`; for a persistent setting, the maximum permitted is `RemoteSigned -Scope CurrentUser`.
4. Never pass untrusted input to `Invoke-Expression` (command-injection risk).

## On-Demand References

| Scenario | Load |
|---|---|
| Translating bash commands, looking up the CMD equivalent, or issuing system administration commands | `references/bash-to-powershell.md` |
| A command fails, or a syntax trap appears (parentheses, JSON depth, null values, exit codes) | `references/pitfalls.md` |
| Calling an external executable, passing arguments, running complex commands | `references/native-invocation.md` |
| CJK output, encoding issues, PowerShell 5.1 vs 7 differences | `references/encoding.md` |
| Preflight for destructive operations, failure classification and retry, high-impact confirmation | `references/safety.md` |
| Registry, services, network, users, scheduled tasks, event logs | `references/windows-admin.md` |

Routine commands (directory listing, `git status`, version checks, commands that have already succeeded) execute directly without loading these rules.
