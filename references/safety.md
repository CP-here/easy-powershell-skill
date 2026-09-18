# Safety, Destructive Operations, and Retry Discipline

Read this file before operations such as deletion, overwriting, stopping services, modifying the registry, or modifying user accounts, or when a command keeps failing.

## 1. High-Impact Operation Confirmation Table

The following operations have a wide blast radius. **State the risk and wait for user confirmation before executing them:**

| Operation | Impact |
|---------|---------|
| Stopping or deleting a system service | Critical functionality may become unavailable |
| Modifying the registry | May compromise system stability and security |
| Deleting system files or formatting | Permanent, unrecoverable data loss |
| Disabling the firewall | Exposes the system to network threats |
| Creating or modifying user accounts | Affects system security and access control |
| Stopping a critical process | May destabilize the system |
| Modifying system environment variables | Changes application behavior |

Process: ① list the users, systems, and services the operation will affect → ② check it against the table above → ③ for a high-impact operation, state the risk and wait for confirmation → ④ execute ordinary operations directly.

## 2. Preflight for Recursive Delete and Overwrite

Before a recursive delete, move, or overwrite, **resolve and verify the target path**:

1. Resolve the absolute target path and print it.
2. Verify that the target lies within the expected root directory, so that wildcard or variable expansion cannot drift.
3. Reject empty paths, root-level paths, and unexpected paths.
4. Perform file-system changes inside PowerShell; never pass a path across shells.

```powershell
# Preview first
Remove-Item -Recurse -Force $target -WhatIf
# After confirmation; use -LiteralPath when the target comes from a variable
Remove-Item -LiteralPath (Resolve-Path $target) -Recurse -Force
```

## 3. Failure Classification and Retry Discipline

**Never retry a failure unchanged:**

1. Read the complete error message.
2. Classify it: parser error, quoting error, path error, encoding error, missing tool, or other.
3. Retry with a command form that addresses the class, for example switching to `-LiteralPath` for a path error, or confirming the tool with `Get-Command` for a missing-tool error.
4. **Stop after two failing forms.** Report the blocker rather than continuing to guess.

## 4. Execution Policy

- On `running scripts is disabled on this system`, bypass the policy for that single invocation only:
  ```powershell
  pwsh -ExecutionPolicy Bypass -File .\script.ps1
  ```
- `Set-ExecutionPolicy Unrestricted` is not recommended; the persistent ceiling is `RemoteSigned -Scope CurrentUser`.
- Do not add `-ExecutionPolicy Bypass` to every invocation by default.

## 5. Error-Handling Patterns

Add `-ErrorAction Stop` or a `try/catch` around critical steps that change state, and never swallow errors silently:

```powershell
$ErrorActionPreference = 'Stop'
try {
    Get-Content -Path .\missing.txt -ErrorAction Stop
} catch {
    Write-Error "Read failed: $_"
    exit 1
}
```

- Do not leave a `catch` block empty, and capture `$_` early if it will be reused.
- `Set-StrictMode -Version Latest` at the top of a script surfaces undefined variables and similar errors early.

## 6. Credentials and Sensitive Information

- Never hard-code passwords or tokens, and never write them to logs. Use `PSCredential`, SecretManagement, or environment variables.
- Never interpolate user input into a command string. Pass arguments to external programs as an argument array (see native-invocation.md).

## 7. Mapped Drives Are Per-User and Per-Session

A mapped drive letter (`X:\`) is scoped to **one user in one session**. When `Test-Path X:\...` fails under an automation or sandbox account but works interactively:

1. Confirm the current identity with `whoami`.
2. List the visible drives with `Get-PSDrive`.
3. If the drive is not visible, ask the user for a UNC path, create the mapping for that account, or switch to running as the current user.

## 8. Timeouts and Long-Running Commands

- Set an explicit timeout for commands that may hang or produce a large volume of output.
- Capture stdout, stderr, the exit code, and the elapsed time, and classify "timed out" separately from "failed".
- Redirect long output to a file and inspect it with a tool, rather than flooding and truncating the console.
