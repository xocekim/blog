---
title: Learning Powershell
date: 2025-08-28
categories:
- windows
tags:
- powershell
---

# PowerShell Scripting – What I’ve Learned So Far  

*This post distills the most useful take‑aways from a Udemy PowerShell course. The audience is expected to be comfortable with basic programming concepts, so only the PowerShell‑specific nuances are highlighted.*


# PowerShell 7 (Core)

| Feature | PowerShell 5.x | PowerShell 7 (Core) |
|---------|----------------|----------------------|
| **Platform** | Windows‑only | Windows, macOS, Linux |
| **License** | Closed‑source | Open‑source (GitHub) |
| **Runtime** | .NET Framework | .NET 6/7 – better performance |
| **Future** | No new major releases | Quarterly updates, active community |

---  

# VS Code

* Install **Visual Studio Code** → *Extensions* → **PowerShell**.  
* The extension underlines **aliases** and suggests the full cmdlet name – a great habit‑forming reminder.  
* **Integrated terminal**:  
  * Set the default shell to PowerShell Core (`"terminal.integrated.shell.windows": "C:\\Program Files\\PowerShell\\7\\pwsh.exe"`).  
  * Highlight code and press **F8** to send it to the **interactive console**; variables become immediately available.  


# Cmdlets & Tab Completion

| Goal | PowerShell command | What you see |
|------|-------------------|--------------|
| Find a cmdlet by **verb‑noun** | `Get-Command -Verb Get -Noun Service` | All matching commands, module name, module version |
| View command history | `Get-History` | List of previously executed lines |
| Built‑in documentation | `Get-Help about*` | All help topics (aliases, pipelines, etc.) |
| Update offline help | `Update-Help` | Downloads the latest help files |
| See examples for a cmdlet | `Get-Help -Name Get-Service -Examples` | Ready‑to‑run usage snippets |

*Tab‑completion works for cmdlet names, parameter names, and object members (`$obj.|Tab`).*  

> **Tip:** Enable **strict mode** early (`Set-StrictMode -Version Latest`) to catch undefined variables and other subtle bugs.


# Core Language Features  

Variables & Automatic Variables  

```powershell
Set-StrictMode -Version Latest
Set-Variable -Name filePath -Value 'C:\MyPath'
Get-Variable -Name filePath
$LASTEXITCODE                # Exit code of the last native command
```

* `$null` is the “no value” placeholder.  
* Read‑only automatic variables (e.g., `$PSEdition`).  
* Constants: `Set-Variable -Name Color -Value 'Green' -Option Constant`.

Objects, Properties, and Methods  

```powershell
$color = 'red'
$color.Length                # Property access
Get-Member -InputObject $color   # List members
$color.Remove(1,1)           # Invoke a method
```

Strings  

* Single quotes → literal (`'my $color'`).  
* Double quotes → variable expansion (`"my $color"`).  
* Format operator (`-f`) for safe interpolation:  

```powershell
'my {0}' -f $color
```

Numbers & Casting  

```powershell
[int]$i = 1
[float]$i = $i               # Explicit cast when needed
```

ScriptBlocks  

```powershell
$sb = { Test-Path -Path 'C:\MyPath' }
$sb          # Shows the block object
& $sb         # Executes the block
```

Collections  

| Collection | Creation | Typical use |
|------------|----------|-------------|
| **Array** | `@('red','blue')` | Fixed‑size, easy indexing |
| **Generic List** | `[System.Collections.Generic.List[string]]@('blue','white')` | Fast inserts/removals for large data |
| **Hashtable** | `@{user1='Mike'; user2='Dave'}` | Key/value look‑ups |

```powershell
$users['user1']   # or $users.user1
$users.ContainsKey('user1')
$users.Remove('user2')
```


# Pipelines

```powershell
$service = 'wuauserv'
Get-Service -Name $service | Stop-Service            # One‑liner
Get-Content -Path c:\services.txt | Get-Service     # Bulk query from a file
```

To verify whether a cmdlet accepts pipeline input:

```powershell
Get-Help Get-Service -Full | Where-Object {
    $_.Name -eq 'InputObject' -and $_.Description -match 'Accept pipeline input'
}
```

*Why pipelines matter:* they let you stream data between commands without materialising intermediate collections, which reduces memory usage and yields expressive, readable one‑liners.


# Control Flow

```powershell
# Equality test
1 -eq 1       # → $true
```

Simple `if/else`

```powershell
if (Test-Connection -ComputerName 1.1.1.1 -Quiet) {
    Write-Host "Host reachable"
} else {
    Write-Host "Host unreachable"
}
```

Negation and logical operators  

```powershell
if (-not (Test-Path -Path 'C:\Temp')) {
    Write-Host "Temp folder missing"
} elseif (-not $true -and (Get-Process -Name notepad)) {
    Write-Host "Notepad is running and condition true"
}
```

`switch` with pipeline variable `$_`

```powershell
$day = 'Tuesday'
switch ($day) {
    'Monday'    { Write-Host "Start of week" }
    'Tuesday'   { Write-Host "Second day" }
    default     { Write-Host "Another day" }
}
```

# Looping

| Loop type | Sample syntax |
|----------|----------------|
| **`foreach` (statement)** | `foreach ($srv in $servers) { Get-Service -Name $srv }` |
| **`ForEach-Object` (pipeline)** | `$servers | ForEach-Object { Get-Service -Name $_ }` |
| **`.foreach()` (method)** | `$servers.foreach({ Get-Service -Name $_ })` |
| **`for` (index)** | `for ($i = 0; $i -lt $servers.Length; $i++) { Get-Service -Name $servers[$i] }` |
| **`while`** | `while ($counter -lt 10) { Write-Host $counter; Start-Sleep -Seconds 1; $counter++ }` |


# Error Handling

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory)]
    [int]$Days
)

function Remove-OldFiles {
    param (
        [Parameter(Mandatory)]
        [string]$RootPath,
        [int]$Days
    )

    Write-Verbose "Scanning $RootPath for files older than $Days days"

    try {
        Get-ChildItem -Path $RootPath -Recurse -File -ErrorAction Stop |
            Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-$Days) } |
            ForEach-Object {
                Remove-Item -Path $_.FullName -Force -ErrorAction Stop
                Write-Verbose "Deleted $($_.FullName)"
            }
    }
    catch {
        Write-Warning "Failed to delete files: $($_.Exception.Message)"
        $Error[0]   # most recent error object
    }
}
```

*Key concepts*  

* **Terminating vs. non‑terminating errors** – `-ErrorAction Stop` converts a non‑terminating error into a terminating one, making it catchable.  
* **`Write‑Verbose` / `Write‑Warning`** – controlled output that can be toggled with `-Verbose`.  
* **`$Error`** – automatic array of recent errors; `$Error[0]` is the latest.  
* To discover the exception type you need to catch:  

```powershell
try { Get-Item 'C:\nonexistent' -ErrorAction Stop } catch { $_.GetType().FullName }
```

# Modularity

Inspecting a function’s definition  

```powershell
Get-Command -Name Get-Process | Select-Object -ExpandProperty Definition
```

Finding the module path  

```powershell
(Get-Module -ListAvailable -Name Microsoft.PowerShell.Management).Path
```

Comment‑based help template  

```powershell
function Write-Log {
<#
.SYNOPSIS
    Appends a timestamped message to a log file.

.DESCRIPTION
    Uses `Add-Content` to write a line that begins with the current
    date/time and the supplied message.

.PARAMETER LogMessage
    The text to log.

.PARAMETER LogFilePath
    Destination file. Defaults to `$env:Temp\log.txt`. Must resolve to an existing
    folder (validated with `ValidateScript`).

.EXAMPLE
    Write-Log -LogMessage 'Backup completed' -LogFilePath 'C:\Logs\backup.log'

#>
    [CmdletBinding()]
    param (
        [Parameter(Mandatory)]
        [string]$LogMessage,

        [Parameter(Mandatory)]
        [ValidateScript({ Test-Path (Split-Path $_ -Parent) })]
        [string]$LogFilePath = "$env:Temp\log.txt"
    )

    $entry = "$(Get-Date -Format o) - $LogMessage"
    Add-Content -Path $LogFilePath -Value $entry -Encoding UTF8
}
```

*`[CmdletBinding()]`* turns an advanced function into a cmdlet‑like entity, giving you common parameters (`-Verbose`, `-ErrorAction`, etc.) and enabling support for `Validate*` attributes.

A more complex function using pipeline blocks  

```powershell
function Install-Software {
<#
.SYNOPSIS
    Installs a software package on one or more remote computers.

.PARAMETER Version
    Software version to install. Valid values are 1 or 2.

.PARAMETER ComputerName
    Target computer(s). Supports pipeline input by property name.
#>
    [CmdletBinding()]
    param (
        [Parameter(Mandatory, ValueFromPipelineByPropertyName)]
        [ValidateSet('1','2')]
        [string]$Version,

        [Parameter(Mandatory, ValueFromPipelineByPropertyName)]
        [string]$ComputerName
    )

    begin {
        Write-Verbose "Starting installation (Version $Version)"
    }
    process {
        Write-Verbose "Installing on $ComputerName"
        $args = @("-Version", $Version, "-ComputerName", $ComputerName)
        Start-Process -FilePath "installer.exe" -ArgumentList $args -Wait -NoNewWindow
        Write-Log -LogMessage "Installed version $Version on $ComputerName" -LogFilePath "C:\Logs\install.log"
    }
    end {
        Write-Verbose "Installation routine finished."
    }
}
```

*Key points*  

* `-ArgumentList` passes an array of arguments to the external executable.  
* `-Wait` makes PowerShell pause until the process exits.  
* `-NoNewWindow` keeps the installer hidden in the current console.  
* `Begin/Process/End` blocks give fine‑grained control over pipeline processing.  


# Practical Tips & Best Practices  

- **Enable strict mode** (`Set-StrictMode -Version Latest`).  
- **Prefer Verb‑Noun cmdlets** for discoverability.  
- **Document every function** with comment‑based help (`.SYNOPSIS`, `.PARAMETER`, `.EXAMPLE`).  
- **Use `CmdletBinding()`** on advanced functions to inherit common parameters.  
- **Validate inputs** (`ValidateSet`, `ValidatePattern`, `ValidateScript`).  
- **Write verbose output** (`Write‑Verbose`) and expose it via `-Verbose`.  
- **Never ignore errors** – either set `-ErrorAction Stop` or handle them in `catch`.  
- **Leverage VS Code**: highlight code → **F8** to send to the interactive terminal, and use the built‑in IntelliSense for cmdlet discovery.  
- **Keep pipelines short and focused**; pipe objects, not raw strings, whenever possible.

