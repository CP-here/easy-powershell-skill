# Windows Administration Quick Reference

Read this file when performing administration tasks: registry, services, network, users, scheduled tasks, event logs, packages, and similar. Operations marked `[WARN]` are high-impact; follow the confirmation process in safety.md first.

## 1. System Information

```powershell
[Environment]::OSVersion.Platform      # Unix / Win32NT - confirm this is Windows first
$PSVersionTable.PSVersion              # PowerShell version
Get-CimInstance Win32_OperatingSystem  # Operating system (prefer CIM over the deprecated Get-WmiObject)
Get-CimInstance Win32_Processor        # CPU
Get-CimInstance Win32_ComputerSystem   # Total memory
Get-PSDrive -PSProvider FileSystem     # Disk volumes
```

## 2. Additional File Operations

```powershell
Get-ChildItem -Recurse -File | Sort-Object Length -Descending | Select-Object -First 20   # Large files
Get-Content big.log -ReadCount 1000    # Stream a large file to avoid exhausting memory
Get-ChildItem | Rename-Item -NewName { $_.Name -replace 'old','new' }   # Batch rename
```

## 3. Service Management [WARN] — stopping or modifying requires confirmation

```powershell
Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' }
Get-Service -Name Spooler -DependentServices          # Dependencies
Restart-Service -Name Spooler -WhatIf                 # Preview first
Set-Service -Name Spooler -StartupType Automatic      # [WARN] Changes the startup type
```

## 4. Registry [WARN] — writing or deleting requires confirmation

```powershell
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' -Name ReleaseId
New-Item -Path 'HKCU:\Software\MyApp' -Force                            # [WARN]
Set-ItemProperty -Path 'HKCU:\Software\MyApp' -Name 'K' -Value 'V'      # [WARN]
Remove-ItemProperty -Path 'HKCU:\Software\MyApp' -Name 'K'              # [WARN]
```

Frequently used paths: `...\CurrentVersion\Run` (startup entries), `...\CurrentVersion\Uninstall` (installed programs), and `HKLM:\SYSTEM\CurrentControlSet\Services` (service configuration).

## 5. Network [WARN] — changing configuration requires confirmation

```powershell
Get-NetIPConfiguration
Get-NetAdapter | Where-Object Status -eq 'Up'
Test-NetConnection -ComputerName example.com -Port 443
Resolve-DnsName -Name example.com
Get-NetTCPConnection -State Listen | Sort-Object LocalPort
Get-NetRoute -AddressFamily IPv4
Get-NetFirewallRule -Enabled True | Select-Object -First 10   # Read-only
```

## 6. Users and Privileges [WARN] — creating, deleting, or modifying requires confirmation

```powershell
whoami                                     # Current user
Get-LocalUser                              # Local user accounts
Get-LocalGroupMember -Group Administrators # Members of the Administrators group
# [WARN] New-LocalUser / Disable-LocalUser / Remove-LocalUser
# Administrator privilege check:
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole('Administrators')
```

## 7. Process Management [WARN] — stopping a critical process requires confirmation

```powershell
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-Process -Name notepad
Stop-Process -Name notepad -WhatIf       # [WARN] Preview first
Start-Process -FilePath notepad           # Only for elevation, a new window, or detached startup
```

## 8. Package Management (Winget / Chocolatey)

```powershell
winget search <name>
winget list --upgrade-available
winget install <id>                       # [WARN] Installing software requires confirmation
choco list                                # Chocolatey (install it first)
choco install <name> -y                   # [WARN]
```

## 9. Scheduled Tasks

```powershell
Get-ScheduledTask | Where-Object State -eq 'Ready'
Get-ScheduledTaskInfo -TaskName 'MyTask'
New-ScheduledTaskAction -Execute 'powershell.exe' -Argument '-File C:\x.ps1'   # Build the action
Register-ScheduledTask -TaskName 'MyTask' -Action $action -Trigger $trigger    # [WARN] Create
Disable-ScheduledTask / Unregister-ScheduledTask                                # [WARN]
Start-ScheduledTask -TaskName 'MyTask'
```

## 10. Disks and Cleanup

```powershell
Get-Volume
Get-PSDrive -PSProvider FileSystem | Select-Object Name, @{N='FreeGB';E={[math]::Round($_.Free/1GB,1)}}
Get-ChildItem $env:TEMP -Recurse -File | Measure-Object Length -Sum   # Space used by temporary files
# [WARN] Preview before cleaning up: Remove-Item $env:TEMP\* -Recurse -Force -WhatIf
```

## 11. Event Logs

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2; StartTime=(Get-Date).AddHours(-24)}
Get-WinEvent -FilterHashtable @{LogName='Application'; Level=2} |
    Select-Object TimeCreated, Id, ProviderName, Message | Select-Object -First 20
# The Security log requires administrator privileges
```

## 12. System Updates [WARN] — installing requires confirmation

```powershell
# PSWindowsUpdate module (run Install-Module PSWindowsUpdate first)
Get-WUList
Get-WUInstall -WhatIf        # [WARN] Preview before installing
```

## 13. File Permissions (ACL)

```powershell
Get-Acl -Path 'C:\data' | Format-List
# Modifying an ACL is high-impact: print the current ACL and the planned change, then confirm before executing
```
