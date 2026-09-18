# Encoding and Version Differences

Read this file when CJK output is mojibake, when the encoding is uncertain, or when behavior differs between PowerShell 5.1 and 7.

## 1. Session-Level UTF-8 Output

Before running an external command that emits CJK text, force the console decoding to UTF-8:

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

## 2. `$OutputEncoding` Versus `Console.OutputEncoding`

Two names that look alike and point in opposite directions:

- **`$OutputEncoding`** — a PowerShell **variable**. It controls how text is encoded **on the way out to** a native program. It affects the argument and pipeline-input direction, not the output you read back.
- **`[Console]::OutputEncoding`** — a **.NET property**. It controls how bytes **returned by** a native program are decoded.
- Do not set `Console.InputEncoding` or `Console.OutputEncoding` by default. Set them only when the terminal and the native program are known to disagree.

**Version note:** the variable is named `$OutputEncoding` in both 5.1 and 7, but its **default value differs**. Under 5.1 it defaults to ASCII (`WebName` = `us-ascii`), which silently replaces CJK characters when they are handed to a native program. Under 7 it defaults to UTF-8. Whenever CJK text will cross into a native program, set it explicitly rather than trusting the default:

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
```

## 3. Specify the Encoding Explicitly When Reading and Writing Text Files

```powershell
Get-Content f -Encoding UTF8
'text' | Set-Content f -Encoding UTF8
'text' | Out-File f -Encoding UTF8
```

- New files: specify the encoding explicitly, according to the requirements of the consumer (usually `utf8`).
- **PowerShell 5.1 defaults are inconsistent — always pass `-Encoding` explicitly.** Measured on 5.1 (code page 936), writing `中文内容` four different ways:

  | Write operation | Default result |
  |---|---|
  | `Set-Content` (no `-Encoding`) | UTF-8 **without** BOM (`E4 B8 AD …`) |
  | `Add-Content` (no `-Encoding`) | UTF-8 **without** BOM |
  | `Out-File` (no `-Encoding`) | **UTF-16LE with BOM** (`FF FE …`) |
  | `>` redirection | **UTF-16LE with BOM** |

  The two families disagree, and neither announces itself. `Set-Content` and `Out-File` writing the same string to the same kind of file produce bytes that are not interchangeable — so a value round-tripped through one and read as the other is mojibake. `>` is shorthand for `Out-File`, which is why it follows the UTF-16 group. Never rely on these defaults; state the encoding.
- **Before modifying an existing file, identify and preserve its current encoding.** Never silently convert a GBK, UTF-16, or BOM-sensitive file. If the encoding is unknown, inspect it or ask.
- Do not apply text encoding options to binary files; use the byte APIs instead.

## 4. Script File Encoding Across Versions (important)

The rule, stated precisely — the two versions fail in **different places**:

| Script file bytes | PowerShell 5.1 | PowerShell 7 |
|---|---|---|
| UTF-8 **with** BOM | correct | correct |
| UTF-8 **without** BOM | **mojibake** | correct |

- **PowerShell 5.1** decodes a BOM-less `.ps1` using the system ANSI code page, so a CJK literal is corrupted before execution even begins. Saving as **UTF-8 with BOM** fixes it — 5.1 writes the BOM automatically when `Set-Content -Encoding UTF8` is used.
- **PowerShell 7** reads BOM-less UTF-8 natively, so it is immune to this trap either way.

Verified on a 936 system with two files holding byte-identical UTF-8 content (`Write-Output "汉字"`), differing only in a BOM:

```
                      5.1               7
nobom.ps1   ->  姹夊瓧  (broken)   汉字  (correct)
bom.ps1     ->  汉字   (correct)   汉字  (correct)
```

Note that `Set-Content -Encoding UTF8` under **7** writes **without** a BOM, whereas under **5.1** it writes **with** one — so a script generated on 7 and then run on 5.1 can regress. When a script must run on both, either add the BOM explicitly or write the file with a byte-oriented API:

```powershell
$bytes = [System.Text.Encoding]::UTF8.GetPreamble() + [System.Text.Encoding]::UTF8.GetBytes($source)
[System.IO.File]::WriteAllBytes($path, $bytes)
```

Do not conclude "mojibake means the script is broken" — the source bytes may be perfectly valid UTF-8 and only the reader at fault. Read the bytes (§6) before assigning blame.

## 5. CJK Output from Child Processes (Python and others)

On Windows, non-ASCII output from Python may raise `UnicodeEncodeError` or produce mojibake:

```powershell
$env:PYTHONIOENCODING = 'utf-8'
$env:PYTHONUTF8 = '1'
python -c "print('中文输出')"
```

## 6. Troubleshooting Order for Mojibake

External `.exe` output is decoded using the console code page, whereas strings inside a PowerShell 7 pipeline are UTF-8. **Suspect the encoding boundary before suspecting the data.**

Start by establishing the baseline, because everything downstream depends on it:

```powershell
[Console]::OutputEncoding.CodePage   # the decisive number
chcp                                 # the process code page
$OutputEncoding.WebName              # the outbound default
```

Measured defaults on a 936 system, which differ more than expected:

| Setting | PowerShell 5.1 | PowerShell 7 |
|---|---|---|
| `$OutputEncoding.WebName` | `us-ascii` (20127) | `utf-8` (65001) |
| `[Console]::OutputEncoding` | `gb2312` (936) | `gb2312` (936) |

The code page is inherited from the machine in **both** versions — installing 7 does not change it. What 7 changes is the outbound default, which is why a pipeline into a native program behaves differently between them.

Then fix in this order:

1. **The code page.** On a 936 system a child process emitting UTF-8 bytes is decoded as GB2312 first, and **setting `[Console]::OutputEncoding` afterwards cannot recover the text** — the bytes are already mis-decoded. Correct the code page (`chcp 65001`) or move to pwsh 7.
2. **`[Console]::OutputEncoding`.** Set it only when the terminal and the native program are known to disagree, and only after the code page is correct.
3. **`$OutputEncoding`.** Governs only the direction *into* a native program. Under 5.1 it defaults to ASCII and silently substitutes `?` for CJK, so set it explicitly before piping CJK into a native program.
4. **The data source.** Only now suspect the file or the program itself.

Diagnostic discipline: read the **raw bytes** before trusting any rendered string. On a mis-set code page, a mojibake string tells you nothing about what the child process actually emitted:

```powershell
$p = Start-Process -FilePath python -ArgumentList $script `
     -RedirectStandardOutput out.bin -NoNewWindow -Wait -PassThru
$b = [System.IO.File]::ReadAllBytes('out.bin')
[System.Text.Encoding]::UTF8.GetString($b)      # try UTF-8
[System.Text.Encoding]::GetEncoding(936).GetString($b)   # try the code page
```

If the bytes decode cleanly as UTF-8, the child process was correct all along and only the reading side — the code page — needs fixing.

## 7. Other Verified 5.1 / 7 Differences

Syntax and behaviour that exists in only one of the two. Each was measured directly:

| Feature | 5.1 | 7 |
|---|---|---|
| `&&` / `||` pipeline chain | `ParserError`: not a valid statement separator | works |
| Ternary `$a ? $b : $c` | `ParserError`: unexpected token `?` | works |
| Null-coalescing `$a ?? $b` | `ParserError`: unexpected token `??` | works |
| `ConvertTo-Json` truncation | **silent** | emits a **warning** |
| `Format-Hex -Count` | parameter absent | present |
| `Set-Content -Encoding UTF8` | writes **with** BOM | writes **without** BOM |
| `Out-File` with no `-Encoding` | UTF-16LE with BOM | UTF-8 without BOM |

Notes:

- Guard for a version when a script must run on both: `if ($PSVersionTable.PSVersion.Major -ge 7) { ... }`. The syntax differences above are **parser** errors, so they cannot be caught with `try/catch` — the file will not load at all.
- The JSON truncation difference is the most dangerous of these, because 5.1 fails **silently**. See pitfalls.md §3. In 7 the warning goes to the warning stream, so `3>$null` suppresses it and `-ErrorAction Stop` does **not** turn it into a terminating error.
