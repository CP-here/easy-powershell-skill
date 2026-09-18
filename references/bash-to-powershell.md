# Bash → PowerShell → CMD Translation Tables

Read this file when translating bash/Linux commands into Windows commands, when a CMD equivalent is required, or when looking up administration commands.

## File System

| Bash | PowerShell | CMD |
|------|------------|-----|
| `ls` | `Get-ChildItem` | `DIR` |
| `ls -la` | `Get-ChildItem -Force` | `DIR /Q` |
| `ls -R` | `Get-ChildItem -Recurse` | `DIR /S /B` |
| `cd path` | `Set-Location path` | `cd path` |
| `pwd` | `(Get-Location).Path` | `cd` (no arguments: prints the current directory) |
| `cp src dst` | `Copy-Item src dst` | `copy src dst` |
| `mv src dst\` | `Move-Item src dst\` | `move src dst\` |
| `rm file.txt` | `Remove-Item file.txt -WhatIf` | `del file.txt` |
| `rm -rf dir` | `Remove-Item -Recurse -Force dir -WhatIf` | `rmdir /S /Q dir` |
| `mkdir -p a/b` | `New-Item -ItemType Directory -Path a\b -Force` | `mkdir a\b` |
| `find /d -name '*.log'` | `Get-ChildItem d -Filter '*.log' -Recurse -File` | `dir /S /B d\*.log` |
| `touch f` | No safe equivalent (see pitfalls.md) | `type nul > f` |

## Text Processing (the highest-frequency failure area)

| Bash | PowerShell |
|------|------------|
| `cat f` | `Get-Content f` |
| `grep PAT f` | `Select-String -Path f -Pattern PAT` |
| `grep -r PAT d` | `Get-ChildItem d -Recurse -File \| Select-String -Pattern PAT` |
| `sed -i 's/a/b/g' f` | `(Get-Content f) -replace 'a','b' \| Set-Content f -Encoding UTF8` |
| `awk '{print $1}' f` | `Get-Content f \| ForEach-Object { ($_ -split '\s+')[0] }` |
| `head -n 10 f` | `Get-Content f -TotalCount 10` |
| `tail -n 10 f` | `Get-Content f -Tail 10` |
| `sort f` | `Get-Content f \| Sort-Object` |
| `sort \| uniq` | `Sort-Object -Unique` |
| `wc -l f` | `(Get-Content f).Count` |
| `xargs -I{} c {}` | `... \| ForEach-Object { c $_ }` |

Note: `Select-String` emits `MatchInfo` **objects**, not lines of text. To obtain the matching line text, append `| Select-Object -ExpandProperty Line`.

## Processes and Services

| Bash | PowerShell | CMD |
|------|------------|-----|
| `ps aux` | `Get-Process` | `tasklist` |
| `kill 1234` | `Stop-Process -Id 1234 -WhatIf` | `taskkill /PID 1234 /F` |
| `killall name` | `Stop-Process -Name name -WhatIf` | `taskkill /IM name.exe /F` |
| `systemctl status s` | `Get-Service -Name s` | `sc query s` |
| `systemctl start s` | `Start-Service -Name s -WhatIf` | `sc start s` |
| `systemctl stop s` | `Stop-Service -Name s -WhatIf` | `sc stop s` |

## Network

| Bash | PowerShell | CMD |
|------|------------|-----|
| `ping -c 4 H` | `Test-Connection -ComputerName H -Count 4` | `ping -n 4 H` |
| `curl URL` | `Invoke-RestMethod -Uri URL` | `curl` (if installed) |
| `wget URL -O f` | `Invoke-WebRequest -Uri URL -OutFile f` | `curl -O f URL` |
| `nslookup d` | `Resolve-DnsName -Name d` | `nslookup d` |
| `ip addr` | `Get-NetIPConfiguration` | `ipconfig /all` |
| `netstat -tlnp` | `Get-NetTCPConnection -State Listen` | `netstat -ano` |

## Environment Variables and Archives

| Purpose | Bash | PowerShell | CMD |
|------|------|------------|-----|
| Read | `echo $PATH` | `$env:PATH` | `echo %PATH%` |
| Set for the current session | `export V=x` | `$env:V = 'x'` | `set V=x` |
| Persist | Write to a profile | `[Environment]::SetEnvironmentVariable('V','x','User')` | `setx V x` (truncates above 1024 characters) |
| Archive | `tar -czf a.tgz d` | `Compress-Archive -Path d -DestinationPath a.zip` | No native command |
| Extract | `tar -xzf a.tgz` | `Expand-Archive -Path a.zip -DestinationPath .` | No native command |

## Key Differences (six to remember)

1. **Path separators.** PowerShell accepts both `/` and `\`; CMD generally requires `\` or quoting.
2. **Quoting.** In PowerShell, single quotes are literal and double quotes expand variables; in CMD, single quotes carry no special meaning and double quotes group text.
3. **Variables.** PowerShell uses `$name` and `$env:NAME`; CMD uses `%name%` (delayed expansion: `!name!`).
4. **Pipelines.** PowerShell passes objects; bash and CMD pass lines of text.
5. **Case sensitivity.** PowerShell cmdlets and CMD commands are largely case-insensitive.
6. **Wildcards.** PowerShell supports `*`, `?`, and character ranges such as `[a-z]`; CMD supports only `*` and `?`.

## Syntax Boundaries (version-sensitive)

- `&&` and `||` are pipeline-chain operators available only in PowerShell 7 and later. On 5.1 they raise `ParserError`: *the token '&&' is not a valid statement separator*. Use `;` or test `if ($LASTEXITCODE -eq 0)` instead.
- Other 7-only syntax that breaks 5.1 at parse time: the ternary `$a ? $b : $c` and null-coalescing `$a ?? $b`. Both raise `ParserError` with an unexpected-token message. These are **parse-time** failures, so `try/catch` cannot contain them — detect the version first:

  ```powershell
  if ($PSVersionTable.PSVersion.Major -ge 7) { ... } else { ... }
  ```
- bash idioms — POSIX command substitution, `rm -rf`, bare-path assumptions — never apply to PowerShell.
- The PowerShell subexpression `$(...)` is valid syntax; use it according to PowerShell semantics.

## Error Triage

| Symptom | Cause and fix |
|------|-----------|
| `running scripts is disabled on this system` | Blocked by the execution policy. One-shot bypass: `pwsh -ExecutionPolicy Bypass -File .\s.ps1`; persistent ceiling: `RemoteSigned -Scope CurrentUser` |
| `A parameter cannot be found that matches parameter name 'rf'` | A Unix parameter was used. Switch to `Remove-Item -Recurse -Force` |
| `The term 'grep' is not recognized as the name of a cmdlet...` | PowerShell has no `grep`. Use `Select-String`, enumerating with `Get-ChildItem -Recurse -File` first |
| `find` runs but returns the wrong results | The call resolved to `find.exe`, a text-search utility. Use `Get-ChildItem -Recurse` to locate files |
| `<` redirection raises a ParserError | PowerShell has no input redirection. Use `Get-Content f \| prog` |
| `$? -eq 1` never detects an external program failure | `$?` is a Boolean. Use `$LASTEXITCODE -ne 0` for external exit codes |
| `log[1].txt` is reported as a non-existent path | Square brackets are wildcards under `-Path`. Use `-LiteralPath` for exact paths |
| `-h127.0.0.1` arrives at the external program as two arguments | The compact argument was split. Write `-h 127.0.0.1`, `--host=...`, or quote the argument as a whole |
| `The token '&&' is not a valid statement separator` | PowerShell 5.1 does not support `&&`. Use `;` or test `$LASTEXITCODE` |
| `unexpected token '?'` / `unexpected token '??'` | Ternary and null-coalescing are PowerShell 7 syntax. Guard on `$PSVersionTable.PSVersion.Major`; `try/catch` will not help, because these fail at parse time |
| `ConvertTo-Json` output contains `@{...}` values | The depth was insufficient. 5.1 fails silently; 7 emits a warning. See pitfalls.md §3 |
| A `.ps1` containing CJK text is mojibake under 5.1 | PowerShell 5.1 **decodes the bytes** using the ANSI code page. Save as UTF-8 with BOM, or run under pwsh 7 |
| External program emits CJK that arrives garbled | The console code page is decoding it wrongly. Check `[Console]::OutputEncoding.CodePage` first; on 936, setting `[Console]::OutputEncoding` alone cannot recover already mis-decoded bytes |
