# PowerShell Scripting 101 → Advanced: Hands-On IT Automation

> GitHub path: `scripting/powershell/README.md`  
> Platform: Windows 10/11, Windows Server, PowerShell 7, and Windows PowerShell 5.1  
> Prerequisites: Windows Terminal or PowerShell, test folders, optional Windows Server/AD lab.

## Table of contents

1. [What PowerShell is](#1-what-powershell-is)
2. [Install, discover, and get help](#2-install-discover-and-get-help)
3. [Create and execute a script](#3-create-and-execute-a-script)
4. [Variables, types, and objects](#4-variables-types-and-objects)
5. [Conditions, loops, and arrays](#5-conditions-loops-and-arrays)
6. [Functions and parameters](#6-functions-and-parameters)
7. [Pipelines, filtering, and export](#7-pipelines-filtering-and-export)
8. [Errors, logging, and safe changes](#8-errors-logging-and-safe-changes)
9. [Windows administration](#9-windows-administration)
10. [Active Directory and remote administration](#10-active-directory-and-remote-administration)
11. [Scheduling automation](#11-scheduling-automation)
12. [Labs and capstone](#12-labs-and-capstone)
13. [Troubleshooting and checklist](#13-troubleshooting-and-checklist)


## Lab safety and conventions

Use a disposable VM or test directory. Read a script before executing it, avoid running downloaded scripts blindly, and do not store passwords or API tokens in source files. Commands shown with example paths, usernames, and hostnames must be adapted to your environment. Test with nonproduction data first.

## Learning objectives

By the end, you should be able to read, write, test, debug, and schedule reusable administration scripts; explain exit codes and errors; and document scripts in GitHub.

## 1. What PowerShell is

PowerShell is a command shell and scripting language based on .NET. Unlike Bash's primarily text-based pipelines, PowerShell pipelines pass objects with properties and methods. Cmdlets use a `Verb-Noun` naming pattern such as `Get-Service` or `Set-Content`.

```powershell
$PSVersionTable.PSVersion
$PSVersionTable.PSEdition
Get-Command Get-Service
```

PowerShell 7 uses `pwsh`; Windows PowerShell 5.1 uses `powershell.exe`. Some Windows/AD modules have compatibility constraints: confirm the edition and module support before automation.

## 2. Install, discover, and get help

Open **Windows Terminal → PowerShell**. On supported Windows systems, install PowerShell 7 using the official Microsoft installer or an approved package manager; for example:

```powershell
winget search Microsoft.PowerShell
winget install --id Microsoft.PowerShell --source winget
```

Open a new terminal and verify `pwsh -v`. You can continue using built-in Windows PowerShell for many labs.

```powershell
Get-Command *Service*
Get-Help Get-Service -Examples
Get-Service | Get-Member
```

`Get-Member` reveals object properties and methods. Use `Update-Help` if you need locally installed detailed help (internet and permissions may be required).

## 3. Create and execute a script

```powershell
New-Item -ItemType Directory -Force "$HOME\it-labs\powershell" | Out-Null
notepad "$HOME\it-labs\powershell\hello.ps1"
```

Paste:

```powershell
Write-Output "Hello, $env:USERNAME!"
Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
```

Save, then run:

```powershell
& "$HOME\it-labs\powershell\hello.ps1"
```

If execution policy blocks scripts, inspect it first:

```powershell
Get-ExecutionPolicy -List
```

For a trusted local lab script, a process-only policy can be used where organizational policy allows:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
```

Execution policy is a safety feature, **not a security boundary**; it may be overridden by Group Policy. Do not use blanket bypasses to run untrusted code.

## 4. Variables, types, and objects

```powershell
$name = 'Student'
[int]$count = 3
$servers = @('web01', 'db01', 'dc01')
Write-Output "Hello $name, count=$count"
$info = [pscustomobject]@{
  Computer = $env:COMPUTERNAME
  User = $env:USERNAME
  CheckedAt = Get-Date
}
$info | Format-List
$info.Computer
```

PowerShell is case-insensitive for most identifiers. `$env:NAME` reads environment variables. Objects can be filtered, sorted, and exported without parsing formatted screen output.

## 5. Conditions, loops, and arrays

```powershell
$path = "$HOME\it-labs\powershell"
if (Test-Path -LiteralPath $path -PathType Container) {
  'Lab directory exists'
} else {
  'Lab directory missing'
}

foreach ($server in @('web01', 'db01')) {
  "Checking $server"
}

for ($i = 1; $i -le 3; $i++) {
  "Attempt $i"
}
```

Comparisons include `-eq`, `-ne`, `-gt`, `-lt`, `-like`, and `-match`. Use `-and`, `-or`, `-not` for Boolean expressions. For paths, `-LiteralPath` avoids interpreting wildcard characters.

## 6. Functions and parameters

Save as `Test-LabPath.ps1`:

```powershell
function Test-LabPath {
  [CmdletBinding()]
  param(
    [Parameter(Mandatory)]
    [string]$Path
  )
  if (Test-Path -LiteralPath $Path) {
    Write-Output $true
  } else {
    Write-Output $false
  }
}
Test-LabPath -Path "$HOME\it-labs\powershell"
```

Functions support parameter validation, reusable logic, and consistent error handling. Avoid using `Write-Host` as a substitute for structured output when another command must consume the result.

## 7. Pipelines, filtering, and export

```powershell
Get-Process |
  Where-Object CPU -GT 0 |
  Sort-Object CPU -Descending |
  Select-Object -First 5 Name, Id, CPU

Get-Service |
  Where-Object Status -EQ 'Running' |
  Select-Object Name, DisplayName, Status |
  Export-Csv "$HOME\it-labs\powershell\running-services.csv" -NoTypeInformation
```

Expected: a table of high-CPU processes and a CSV of running services. Avoid `Format-Table` before `Export-Csv`: formatting converts objects into formatting records.

Import and inspect:

```powershell
Import-Csv "$HOME\it-labs\powershell\running-services.csv" |
  Select-Object -First 3
```

JSON examples:

```powershell
[pscustomobject]@{ Host = $env:COMPUTERNAME; Time = (Get-Date) } |
  ConvertTo-Json |
  Set-Content "$HOME\it-labs\powershell\host.json"
Get-Content "$HOME\it-labs\powershell\host.json" -Raw | ConvertFrom-Json
```

## 8. Errors, logging, and safe changes

```powershell
$ErrorActionPreference = 'Stop'
try {
  Get-Item -LiteralPath 'C:\this-path-does-not-exist' -ErrorAction Stop
} catch {
  Write-Warning "Operation failed: $($_.Exception.Message)"
} finally {
  'Check completed'
}
```

Use `-ErrorAction Stop` when a cmdlet emits nonterminating errors that your `catch` must handle. Prefer structured logs with timestamps and avoid logging credentials.

For commands that support it, preview changes with `-WhatIf`:

```powershell
New-Item -Path "$HOME\it-labs\powershell\example.txt" -ItemType File -WhatIf
```

`-WhatIf` support is cmdlet-specific; always check `Get-Help <command> -Full` and verify your own script logic. Do not run destructive operations on production systems while learning.

## 9. Windows administration

Create `Get-LabHealth.ps1`:

```powershell
[CmdletBinding()]
param(
  [string]$OutputPath = "$HOME\it-labs\powershell\health.csv"
)
$ErrorActionPreference = 'Stop'
try {
  $os = Get-CimInstance Win32_OperatingSystem
  $disk = Get-CimInstance Win32_LogicalDisk -Filter "DeviceID='C:'"
  $result = [pscustomobject]@{
    Computer = $env:COMPUTERNAME
    Time = Get-Date
    OS = $os.Caption
    FreeMemoryGB = [math]::Round($os.FreePhysicalMemory / 1MB, 2)
    CFreeGB = if ($disk) { [math]::Round($disk.FreeSpace / 1GB, 2) } else { $null }
  }
  $result | Export-Csv -LiteralPath $OutputPath -NoTypeInformation
  $result
} catch {
  Write-Error "Health collection failed: $($_.Exception.Message)"
  exit 1
}
```

Run:

```powershell
& "$HOME\it-labs\powershell\Get-LabHealth.ps1"
Import-Csv "$HOME\it-labs\powershell\health.csv"
```

The `FreePhysicalMemory` CIM value is in KiB, so dividing by `1MB` (1,048,576) gives GiB (labeled GB for simplicity here). Other examples:

```powershell
Get-Service | Where-Object Status -eq 'Stopped' | Select-Object -First 10
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias,IPAddress
Get-ScheduledTask | Select-Object -First 10 TaskName,State
```

## 10. Active Directory and remote administration

In a Windows Server lab with the **ActiveDirectory** module installed and appropriate permissions:

```powershell
Import-Module ActiveDirectory
Get-ADDomain
Get-ADUser -Filter * -SearchBase 'OU=LabUsers,DC=corp,DC=example,DC=test' |
  Select-Object Name,SamAccountName,Enabled
```

Adjust the distinguished name to your lab. The module is not automatically available on every Windows client; install RSAT as appropriate. Read-only inventory is the safest starting point.

To preview a test account disable operation (authorized lab only):

```powershell
Disable-ADAccount -Identity 'labuser01' -WhatIf
```

For Windows remoting, configure WinRM only on trusted systems and limit network access. Example (after remoting is securely configured):

```powershell
Test-WSMan -ComputerName 'server01.corp.example.test'
Invoke-Command -ComputerName 'server01.corp.example.test' -ScriptBlock { hostname; Get-Service WinRM }
```

Domain authentication and firewall policy must permit access. Prefer Kerberos within a domain; avoid adding broad TrustedHosts entries or disabling certificate verification. PowerShell 7 also supports SSH-based remoting when properly configured.

## 11. Scheduling automation

Use **Task Scheduler** GUI for a lab:

1. Open Start → **Task Scheduler**.
2. Select **Create Task**; give it a descriptive name, e.g., `Lab Health Report`.
3. Under **Triggers**, select **New** → **Daily** and choose a time.
4. Under **Actions**, select **New** → **Start a program**.
5. Program: `powershell.exe` (or the full path to approved `pwsh.exe`).
6. Arguments: `-NoProfile -File "C:\Users\YOURUSER\it-labs\powershell\Get-LabHealth.ps1"` (replace YOURUSER).
7. Save; right-click task → **Run**; verify CSV timestamp and **Last Run Result**.

Use the minimum required account privileges. Prefer an approved service account or managed identity for enterprise jobs; do not embed credentials in task arguments or scripts.

## 12. Labs and capstone

**Lab A — Hello:** write `hello.ps1`, run it, and explain `&`, `$env:USERNAME`, and `Get-Date`.

**Lab B — Objects:** make a `[pscustomobject]` with hostname, IP address, and timestamp. Export to CSV and JSON; import both.

**Lab C — Services:** filter running services, sort by name, and export to CSV. Explain why `Format-Table` should not precede export.

**Lab D — Error handling:** write a function that checks a file path; handle a nonexistent file with `try/catch` and a meaningful message.

**Lab E — AD inventory (optional):** query only a test OU; export account name, status, and last logon metadata; do not change accounts.

**Capstone — Windows fleet inventory:** accept a list of approved hostnames; collect OS version, free disk, and running services via permitted remoting; write structured CSV output; log failures per host; schedule a daily run; document prerequisites and rollback.

## 13. Troubleshooting and checklist

| Symptom | Likely cause | Fix |
|---|---|---|
| `.ps1 cannot be loaded` | Execution policy / trust | `Get-ExecutionPolicy -List`; use approved scope/policy |
| `Access denied` | Insufficient permissions | Use correct authorized account; avoid unnecessary elevation |
| `Command not found` | Module unavailable | `Get-Command`, `Get-Module -ListAvailable` |
| Empty CSV / strange rows | Formatted output piped to export | Export objects before formatting |
| `catch` not triggered | Nonterminating cmdlet error | Add `-ErrorAction Stop` |
| Remote access fails | DNS, WinRM, auth, firewall | `Resolve-DnsName`, `Test-WSMan`, check policies |
| Scheduled task works manually only | Different user, path, environment | Absolute paths, correct account, task history |

**Completion checklist:**

- [ ] Can create and execute `.ps1` scripts.
- [ ] Can use objects, arrays, loops, functions, and parameters.
- [ ] Can filter and export structured data.
- [ ] Can handle errors and preview supported changes with `-WhatIf`.
- [ ] Can generate a Windows health report.
- [ ] Can query a test AD OU read-only (if available).
- [ ] Can schedule and verify a script.

## Further reading

- [Microsoft Learn: PowerShell documentation](https://learn.microsoft.com/powershell/)
- [Microsoft Learn: About scripting](https://learn.microsoft.com/powershell/scripting/learn/ps101/)
- [Microsoft Learn: Active Directory PowerShell](https://learn.microsoft.com/powershell/module/activedirectory/)
