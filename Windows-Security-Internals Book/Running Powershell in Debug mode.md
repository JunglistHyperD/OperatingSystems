Powershell ISE ( INtergrated Scripting Env)


We have a simple script called `demo.ps1`
```
$sid = Get-NtSid -Sddl "S-1-5-32-545"
$sid.Name
```

IN powershell We set up the debug for line 2
```
Set-PSBreakpoint -Script demo.ps1 -Line 2
```

Run the script
```
.\demo.ps1
```

You’ll get a debug prompt like: `[DBG]: PS C:\Path>>`
4. Type:
    - `s` — step into
    - `v` — view variables
    - `c` — continue
    - `q` — quit debug


### Mutant object 
A Mutant object is a kernel synchronization object that enforces mutual exclusion across threads and processes, with ownership tracked by the kernel.


`.\MutantLifecycle.ps1`

```powershell

# Import NtObjectManager
Import-Module NtObjectManager

# Create a named Mutant object
$mutant = New-NtMutant -Name "\BaseNamedObjects\MyDebugMutant"

# Add custom attributes (non-functional, just for tracking in debug)
$mutant | Add-Member -MemberType NoteProperty -Name "Label" -Value "SessionLock"
$mutant | Add-Member -MemberType NoteProperty -Name "IsCritical" -Value $true

# Output info to debug console
Write-Debug "Created mutant: $($mutant.Name)"
Write-Debug "Label: $($mutant.Label), IsCritical: $($mutant.IsCritical)"

# Pause execution so you can step through (you'll hit this in debugger)
$null = Read-Host "Press Enter to continue and delete the mutant"

# Close handle — object will be deleted if no other references exist
$mutant.Close()

Write-Debug "Mutant closed."

# End of script
```

Debug
```
Set-PSBreakpoint -Script .\MutantLifecycle.ps1 -Line 4
```
