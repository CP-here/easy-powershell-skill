# PowerShell Syntax Traps

Read this file when a command fails or when the syntax is uncertain. Every entry is a pitfall that agents hit frequently.

## 1. Parenthesize Cmdlet Calls Around Logical Operators

A cmdlet call on either side of `-and` or `-or` must be wrapped in parentheses, otherwise PowerShell reports `parameter 'or'`:

```powershell
# [!] Fails
if (Test-Path "a" -or Test-Path "b") { ... }
if (Get-Item $x -and $y -eq 5) { ... }

# [OK]
if ((Test-Path "a") -or (Test-Path "b")) { ... }
if ((Get-Item $x) -and ($y -eq 5)) { ... }
```

## 2. A `foreach` Statement Cannot Be Piped Directly

Appending `|` to a statement block (`foreach (...) { ... }`) raises `An empty pipe element is not allowed`:

```powershell
# [!] ParserError
$rows = foreach ($x in $items) { [pscustomobject]@{name=$x} } | Format-Table

# [OK] Assign first, then pipe
$rows = foreach ($x in $items) { [pscustomobject]@{name=$x} }
$rows | Format-Table
```

## 3. `ConvertTo-Json` Requires an Explicit `-Depth`

The default depth is **2**, and anything deeper is replaced with the type name rather than raising an error. Measured with `{"a":{"b":{"c":{"d":1}}}}`:

```
-Depth 1  ->  {"a":{"b":"@{c=}"}}
-Depth 2  ->  {"a":{"b":{"c":"@{d=1}"}}}     <- the default
-Depth 3  ->  {"a":{"b":{"c":{"d":1}}}}      <- complete
```

Note where the truncation lands: at depth 1 the *value* of `b` is destroyed, at depth 2 the value of `c` is. The depth must be **at least** the number of nested levels, so a deeply nested object may need a large value — but match the data rather than reaching for an arbitrary `10`:

```powershell
$data | ConvertTo-Json -Depth 3      # enough for 3 levels of nesting
# Reading back
Get-Content f.json -Raw | ConvertFrom-Json
```

**The two versions report this differently, and the difference is a trap:**

| | Behaviour at insufficient depth |
|---|---|
| PowerShell 5.1 | **completely silent** — truncated output, no message |
| PowerShell 7 | emits a warning: *the JSON has been truncated, because serialization exceeded the set depth* |

Because 5.1 says nothing, a truncated payload can reach a consumer undetected. In 7 the message goes to the **warning** stream, so `-ErrorAction Stop` does **not** promote it to a terminating error, and `-WarningAction SilentlyContinue` or `3>$null` silences it:

```powershell
# 7 only: make the truncation loud instead of suppressing it
$json = $data | ConvertTo-Json -Depth 10 -WarningAction Stop
```

Verify the round trip when the schema nests more than two levels, and do it by **inspecting the parsed result**, not by trusting that no error was raised:

```powershell
$roundTrip = ($data | ConvertTo-Json -Depth 10) | ConvertFrom-Json
# Compare against $data; a '@{' value marks a truncation
```

## 4. Test for Null Before Accessing a Property

An empty array has `.Count` of `0` and is safe to read; a genuine `$null` is the hazard. Under 5.1, `$null.Count` silently yields `0` rather than raising, so a guard such as `.Count -gt 0` fails quietly instead of failing loudly:

```powershell
# [!] $null slips through: .Count is 0, so the guard reads as "valid, but empty"
if ($array.Count -gt 0) { ... }

# [OK] Guard the object before touching the property
if ($array -and $array.Count -gt 0) { ... }
if ($text) { $text.Length }
```

`-gt 0` on a `$null` value and on an empty collection produce the **same** `$false`, so the guard cannot tell "missing" from "empty". Decide which one you need: `if ($array -and $array.Count -gt 0)` rejects both, whereas `if ($null -ne $array)` accepts an empty array. Test the object itself, not a property of it, and the same applies to `.Length` and to indexing.

## 5. Use ASCII Symbols in Scripts and Output

Unicode symbols can trigger parsing or encoding problems. Use text markers instead:

| Purpose | Symbols to avoid | Markers to use |
|---|---|---|
| Success | U+2705, U+2713 | `[OK]` `[+]` |
| Failure | U+274C, U+2717, U+1F534 | `[!]` `[X]` |
| Warning | U+26A0, U+1F7E1 | `[WARN]` `[*]` |
| Information | U+2139, U+1F535 | `[INFO]` `[i]` |

## 6. Exit Codes Are Two Separate Systems

- `$?` is a Boolean and reflects only whether the previous cmdlet succeeded. Never write `$? -eq 1`.
- The exit code of an external `.exe` is held in `$LASTEXITCODE`, which cmdlets do not modify.
- To detect failure, use `if (-not $?)` for cmdlets and `if ($LASTEXITCODE -ne 0)` for external programs.

## 7. Use `-LiteralPath` for Exact Paths

Square brackets are wildcards under `-Path`, so `log[1].txt` silently matches nothing:

```powershell
Get-Content -LiteralPath 'C:\Data[1]\log[1].txt'
# Build paths with Join-Path, never by string concatenation
$p = Join-Path $env:USERPROFILE 'file.txt'
```

## 8. Compact Arguments Are Split by Native Programs

PowerShell re-parses arguments in the `-flagvalue` form. In testing, `-h127.0.0.1` arrived at the program as the two arguments `-h127` and `.0.0.1`:

```powershell
# [!] May be split
tool.exe -h127.0.0.1

# [OK] Reliable
tool.exe -h 127.0.0.1
tool.exe --host=127.0.0.1
tool.exe '-h127.0.0.1'
```

## 9. `touch` Has No Safe Equivalent

`New-Item -ItemType File -Force` **truncates an existing file**. To update the timestamp only:

```powershell
if (Test-Path $f) { (Get-Item $f).LastWriteTime = Get-Date } else { New-Item -ItemType File $f }
```

## 10. There Is No Input Redirection (`<`)

`prog < query.sql` raises a ParserError, because the syntax is reserved but unimplemented. Feed stdin through a pipeline instead:

```powershell
Get-Content query.sql | prog
```

## 11. Comparison and Assignment Syntax

- Comparison uses `-eq`, `-ne`, `-lt`, and `-gt` (not `==` or `<`); assignment uses `=`.
- Put `$null` on the **left** when comparing: `if ($null -eq $x)`. On the right, a collection comparison misleads.
- Strings compare lexicographically, so cast before comparing numerically: `[int]$a -lt [int]$b`.

## 12. `Write-Output` Versus `Write-Host`

- `Write-Output` (or emitting an object directly) writes to the pipeline and returns data.
- `Write-Host` writes to the console display only and never enters the pipeline.

## 13. `Select-String` Returns Objects

`Select-String` emits `MatchInfo` objects rather than plain text. To obtain the line text only:

```powershell
Get-ChildItem -Recurse -File | Select-String -Pattern 'error' | Select-Object -ExpandProperty Line
```

## 14. Assign Complex Interpolations to a Variable First

```powershell
# [!] Multi-level property interpolation is hard to read and easy to get wrong
"Value: $($obj.prop.sub)"

# [OK]
$value = $obj.prop.sub
"Value: $value"
```
