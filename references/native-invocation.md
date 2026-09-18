# External Program Invocation and Complex Command Execution

Read this file when invoking a native executable (`git`, `node`, `python`, `ffmpeg`, and similar), passing arguments, or running a complex command.

## 1. Invoke an External Program with `& $exe @argList`

**Do not assemble a large command string.** Place the arguments in an array, one element per argument, and execute it with the call operator `&` plus splatting:

```powershell
$exe = 'C:\Path With Spaces\tool.exe'
$argList = @(
    '--input'
    'C:\Data Folder\input.json'
    '--flag'
)

& $exe @argList
$exitCode = $LASTEXITCODE
if ($exitCode -ne 0) {
    throw "$exe failed with exit code $exitCode"
}
```

Rules:

- Each native argument is one array element; a path containing spaces needs no manual quoting.
- Invoke an executable held in a variable with `&`.
- Capture `$LASTEXITCODE` **immediately** after the call.
- Do not use `$args` as the name of your own argument array; it is a PowerShell automatic variable. Use `$argList` or `$nativeArgs`.
- Do not use `Invoke-Expression`.
- Do not wrap the executable launch in `cmd.exe /c`.
- Do not use bash-style `\"` escaping.

## 2. Use Hashtable Splatting for Cmdlets

```powershell
$params = @{
    LiteralPath = 'C:\Data[1]\input.txt'
    Destination = 'C:\Output'
    Force       = $true
    ErrorAction = 'Stop'
}
Copy-Item @params
```

- Use `-LiteralPath` for exact paths, except for cmdlets that expose only `-Path` (such as `New-Item`). When uncertain, confirm with `Get-Command <cmdlet> -Syntax`.
- Do not judge cmdlet success by `$LASTEXITCODE`. Use terminating errors instead: `$ErrorActionPreference = 'Stop'`.

## 3. For Complex Commands, Write a Temporary `.ps1` and Run It with `-File`

For multi-line code, nested quoting, JSON or XML, regular expressions, pipelines, redirection, or non-ASCII paths, **do not stack quotes**. Write a temporary script file instead:

```powershell
pwsh.exe -NoLogo -NoProfile -NonInteractive -File script.ps1
```

- Whenever the command grows beyond a short, simple expression, prefer `-File` over `-Command`.
- Avoid deep nesting such as `cmd.exe /c pwsh.exe -Command "..."`.
- Do not add `-ExecutionPolicy Bypass` by default. Add it only when the execution policy actually blocks a trusted script.

## 4. Dollar Expansion When Passing from an Outer to an Inner Layer

An outer PowerShell instance expands `$p`, `$env:`, and similar variables before the inner command sees them. When values must be passed through, **use single quotes in the outer layer**:

```powershell
pwsh.exe -NoLogo -NoProfile -Command '$p = "C:\Data Folder\input.txt"; Test-Path -LiteralPath $p'
```

When a command has to cross several interpreters or wrapper layers, stop stacking quotes and write a `.ps1` file.

## 5. Limitations of `Start-Process`

- For ordinary foreground execution, use `& $exe @argList`. Reserve `Start-Process` for elevation, a new or hidden window, detached startup, and shell behaviors.
- `Start-Process -ArgumentList` **serializes the arguments back into a command-line string**; it is not a reliable structured argument-passing API.
- When exact argument boundaries matter, use `ProcessStartInfo.ArgumentList`:

```powershell
$psi = [System.Diagnostics.ProcessStartInfo]::new()
$psi.FileName = $exe
$psi.UseShellExecute = $false
foreach ($arg in $argList) { $psi.ArgumentList.Add($arg) }

$process = [System.Diagnostics.Process]::Start($psi)
$process.WaitForExit()
if ($process.ExitCode -ne 0) { throw "Process failed: $($process.ExitCode)" }
```

## 6. Confirm an External Program Exists Before Invoking It

`The term 'xxx' is not recognized` means the tool is missing. Do not retry repeatedly:

```powershell
Get-Command rg          # Returns the path and version if present, or raises an error if absent
# If the tool is missing and the task is simple, substitute a native PowerShell command
```

## 7. Verify Version-Specific Parameters

PowerShell 5.1 and 7 may expose different parameters for the same command. Before relying on a version-specific parameter, verify it:

```powershell
$PSVersionTable.PSVersion
Get-Command Format-Hex -Syntax
Get-Command Get-FileHash -Syntax
```

Example: `Format-Hex -Count` exists only in PowerShell 7; under 5.1 use `Format-Hex -LiteralPath f | Select-Object -First 2`.

Probe for a specific parameter rather than assuming, so the check works on either version without an error:

```powershell
# $null means the parameter is absent on this version
$hasCount = Get-Command Format-Hex -ParameterName Count -ErrorAction SilentlyContinue
if ($hasCount) { Format-Hex -LiteralPath f -Count 2 } else { Format-Hex -LiteralPath f | Select-Object -First 2 }
```

When command discovery behaves unexpectedly, inspect `$env:PSModulePath` and run `Get-Module -ListAvailable <name>` before concluding that a cmdlet is missing.

## 8. Decision Order (simplest first)

1. A PowerShell cmdlet can do the job: use the cmdlet.
2. An external program must be called: `& $exe @argList`.
3. The logic is complex: a temporary `.ps1` plus `pwsh -File`.
4. A detached process needs exact argument passing: `ProcessStartInfo.ArgumentList`.
5. Elevation or a new window is required: `Start-Process`.
6. CMD semantics are required: `cmd.exe /c`.
7. `Invoke-Expression`: only as a strictly controlled last resort.

## 9. Strings and Multi-line Code

- Use single quotes for literal strings and paths; use double quotes only when expansion is required.
- Avoid backtick line continuation. Use arrays, hashtables, splatting, parentheses, or script blocks instead.
- Do not use bash heredocs (`python - <<'PY'`), because PowerShell parses `<` differently. Use a temporary script file or a here-string piped to the program.
- Build JSON from objects with `ConvertTo-Json -Depth`; never hand-write the escaping.
- For multi-line literal text, use a single-quoted here-string: nothing may follow the opening `@'`, and the closing `'@` must be on a line of its own.
