## The Windows Kernel, explored with `NtObjectManager`

> Personal study notes. Every snippet is from Chapter 2 unless it's marked **[extra]**. **[extra]** marks real-world uses I've added from general practice; they aren't in the book. Prerequisite (from Ch.1): `Install-Module NtObjectManager -Scope CurrentUser` then `Import-Module NtObjectManager`. Many commands show more when you run them in an **elevated** PowerShell.

---

## 0. Mental model in 30 seconds

|Kernel prefix|Subsystem|Why you care|
|---|---|---|
|`Nt` / `Zw`|System call interface|How user mode asks the kernel to do things|
|`Se`|Security Reference Monitor|Does the access checks (token vs security descriptor)|
|`Ob`|Object manager|Everything is an object, and every object can have an ACL|
|`Ps`|Process & thread manager|Processes and threads are opened by ID, not by name|
|`Cm`|Configuration manager|The registry|
|`Mm`|Memory manager|Virtual memory, sections, protections|
|`Io`|I/O manager|Drivers, devices, files|
|`Ci`|Code integrity|Signature checks, especially for drivers|

The PowerShell verbs map to system calls: **`New-Nt<Type>`** = Create, **`Get-Nt<Type>`** = Open.

---

## 1. SIDs (Security Reference Monitor)

```powershell
Get-NtSid -Name "Users"            # name -> SID   (BUILTIN\Users = S-1-5-32-545)
Get-NtSid -Sddl "S-1-5-32-545"     # SID  -> name
```

**When you'd use it**

- **Log triage:** Security events and EDR alerts often show raw SIDs. Resolve them to see _who_ did it.
- **ACL reviews:** decode unfamiliar SIDs in `icacls` output or SDDL strings.
- **[extra]** Spot orphaned SIDs on file and registry ACLs. These are SIDs that no longer resolve, often left behind by deleted accounts. They're a common audit finding.

---

## 2. Object types & generic mappings

```powershell
Get-NtType                                        # all kernel object types (~70+)
Get-NtType | Select-Object Name, GenericMapping   # what GenericRead/Write/Execute/All mean per type
```

**When you'd use it**

- Before you review an ACL, check what "GenericRead" actually grants on that object type. It's different for Files, Tokens, Processes and so on.
- Find out which object types exist on a given Windows build when researching attack surface.

---

## 3. Browsing the Object Manager Namespace (OMNS)

```powershell
ls NtObject:\ | Sort-Object Name                     # root of the kernel "filesystem"
ls NtObject:\Dfs | Select-Object SymbolicLinkTarget  # where a symlink points
Get-Item NtObject:\Device\DfsClient | Format-Table   # inspect a single object
ls NtObject:\Device                                  # device objects (driver entry points)
ls NtObject:\REGISTRY                                # registry root (A, MACHINE, USER, WC)
```

Key directories: `\BaseNamedObjects` (shared named objects), `\Device`, `\GLOBAL??` (drive letters and symlinks), `\KnownDlls`, `\ObjectTypes`, `\Sessions`, `\Windows`, `\RPC Control`.

**When you'd use it**

- **Malware hunting:** many families create a named **mutex** so they don't infect the same host twice. Look in `\BaseNamedObjects` and `\Sessions\<n>\BaseNamedObjects` for known indicator names.
- **Privilege-escalation research:** look for object directories or symlinks a low-privileged user can write to. Object-manager symlink tricks are a classic way to redirect privileged file operations.
- **RPC attack surface:** `\RPC Control` lists local RPC/ALPC endpoints you can then examine.
- **[extra] BYOVD hunting:** `ls NtObject:\Device` shows device objects exposed by third-party drivers. The book's own sample output includes **`DBUtil_2_3`**, the Dell driver behind CVE-2021-21551. Seeing a device like that on an estate is a red flag.

---

## 4. NTSTATUS codes

```powershell
Get-NtStatus                              # list all known status codes
Get-NtStatus 0xC0000034 | Format-List     # decode one: STATUS_OBJECT_NAME_NOT_FOUND, Win32 error 2, etc.
Get-NtDirectory \THISDOESNOTEXIST         # failing calls surface the NTSTATUS as a .NET exception
```

Format: bits 31–30 = severity (0 success … 3 error), 29 = customer bit, 28 = reserved, 27–16 = facility, 15–0 = code. Codes worth knowing: `0xC0000022` access denied · `0xC0000034` name not found · `0xC0000023` buffer too small.

**When you'd use it**

- Decode unexplained hex errors in crash dumps, driver logs, EDR telemetry or failed API calls.
- Translate between NTSTATUS and the Win32 error code that admins actually Google.
- **[extra]** If a log shows a _negative_ number (for example `-1073741790`), that's the same code printed as a signed integer. `Get-NtStatus` shows both forms.

---

## 5. Access masks (decode "GrantedAccess" numbers)

```powershell
Get-NtTypeAccess -Type File                               # named rights for a type
Get-NtTypeAccess -Type File | Select SDKName, Value       # SDK names (FILE_READ_DATA…) for porting C code

Get-NtAccessMask -FileAccess ReadData, ReadAttributes, ReadControl   # names -> number (00020081)
Get-NtAccessMask -FileAccess GenericRead -MapGenericRights           # generic -> type-specific (00120089)
Get-NtAccessMask 0x120089 -AsTypeAccess File                         # number -> names

$mut = New-NtMutant
$mut.GrantedAccess        # enum view
$mut.GrantedAccessMask    # numeric view
```

Mask layout: 31–28 generic · 27–21 special (`AccessSystemSecurity`, `MaximumAllowed`) · 20–16 standard (`Delete`, `ReadControl`, `WriteDac`, `WriteOwner`, `Synchronize`) · 15–0 type-specific.

**When you'd use it**

- **[extra] Detection engineering:** Sysmon Event ID 10 (ProcessAccess) and Windows events 4656/4663 log a raw `GrantedAccess`/`AccessMask` hex value. Decode it to see what was really requested:
    
    ```powershell
    Get-NtAccessMask 0x1010 -AsTypeAccess Process   # VmRead + QueryLimitedInformation, a classic LSASS-dump pattern
    ```
    
- **ACL review:** anything granting `WriteDac` or `WriteOwner` to a low-privileged group effectively hands over full control.

---

## 6. Handles: list, close, duplicate, protect

```powershell
Get-NtHandle -ProcessId $pid                         # handle table of a process

$m = New-NtMutant \BaseNamedObjects\ABC              # create a named object
$m.Close(); $m.IsClosed                              # close manually

Use-NtObject($m = New-NtMutant \BaseNamedObjects\ABC) { $m.FullPath }   # auto-close (like C# using)

$dup = Copy-NtObject $mut                            # duplicate a handle
Compare-NtObject $mut $dup                           # same kernel object? -> True
$mask = Get-NtAccessMask -MutantAccess ModifyState
Copy-NtObject $mut -DesiredAccessMask $mask          # duplicate with reduced rights

$mut.ProtectFromClose = $true
Close-NtObject -SafeHandle $mut.Handle -CurrentProcess   # -> STATUS_HANDLE_NOT_CLOSABLE
```

**When you'd use it**

- **"Which process is locking this file?"** This is the PowerShell equivalent of Sysinternals `handle.exe` (see Worked Example A).
- **Incident response:** find processes holding handles to `lsass.exe` or other sensitive processes. Handle duplication is a known way to dump credentials without opening LSASS directly.
- **Script hygiene:** use `Use-NtObject` so long-running scripts don't leak handles.
- **Defensive design:** reducing rights on a duplicated handle is **not** a security boundary. If the receiving process can reopen the object, it gets full rights back.
- **[extra]** Create the named mutex of a known malware family on a test host to check that your detections fire, or as a "vaccine" against that family.

**Permanent objects:** `MakePermanent()` needs `SeCreatePermanentPrivilege`. `MakeTemporary()` only needs `Delete` access.

---

## 7. Query / Set information on objects

```powershell
Get-NtObjectInformationClass Process          # list query info classes
Get-NtObjectInformationClass Key -Set         # set classes can differ from query classes

$proc = Get-NtProcess -Current
Get-NtObjectInformation $proc ProcessTimes                 # may fail: buffer too small
Get-NtObjectInformation $proc ProcessTimes -Length 32      # raw bytes
Get-NtObjectInformation $proc ProcessTimes -AsObject       # parsed object
$proc | Format-List                                        # all friendly properties
$proc.CreationTime
# Set-NtObjectInformation is the write counterpart
```

**When you'd use it**

- Get data the Win32 APIs don't expose, or that's undocumented.
- **IR timelines:** process creation times, parent PID, session ID.
- **Research:** brute-force buffer lengths when the kernel doesn't report the size it needs.

---

## 8. Files & devices (I/O manager)

```powershell
Use-NtObject($f = Get-NtFile "\SystemRoot\notepad.exe") {
    $f | Select-Object FullPath, NtTypeName      # -> \Device\HarddiskVolume3\Windows\notepad.exe
}
Get-Item NtObject:\Device\HarddiskVolume3        # it's a Device object
Get-NtKernelModule                               # loaded drivers + base addresses
# New-NtFile creates files at the native level
```

**When you'd use it**

- **Log correlation:** EDR and Sysmon often record `\Device\HarddiskVolumeN\...` paths. This maps them back to real files and drives.
- **Driver audits:** `Get-NtKernelModule` lists every loaded driver.
    - **[extra]** Compare that list against the loldrivers.io vulnerable-driver list to hunt for BYOVD (bring-your-own-vulnerable-driver).
    - **[extra]** Investigate any driver you don't recognise when chasing a suspected rootkit.

---

## 9. Processes & threads

```powershell
Get-NtProcess -InfoOnly                  # enumerate without needing to open each process
Get-NtThread  -InfoOnly
$proc = Get-NtProcess -ProcessId $pid
$proc.CommandLine
$proc.Win32ImagePath
Get-NtProcess -Current / Get-NtThread -Current     # pseudo handles (-1 / -2)
Test-NtProcess -ProcessId 1234 -Access DupHandle   # can I open it with this right?
```

**When you'd use it**

- **Low-privileged enumeration:** list processes and owners' sessions as a standard user, for example during a pentest foothold.
- **Triage:** grab command lines to spot LOLBins, encoded PowerShell and odd parents.
- **Access testing:** check which processes your current token can open, and with what rights. This is the first step of many LPE checks.
- Thread security is separate from process security. You can sometimes open a thread even when you can't open its process.

---

## 10. Virtual memory

```powershell
Get-NtVirtualMemory                                            # map of current process memory
$addr = Add-NtVirtualMemory -Size 1000 -Protection ReadWrite   # alloc (rounds up to 4096)
Read-NtVirtualMemory  -Address $addr -Size 4 | Out-HexDump
Write-NtVirtualMemory -Address $addr -Data @(1,2,3,4)
Set-NtVirtualMemory   -Address $addr -Protection ExecuteRead -Size 4   # changes whole page
Remove-NtVirtualMemory -Address $addr
# All accept -Process <handle> to act on another process
```

States: **Commit** (usable) · **Reserve** (address range held, no backing; touching it crashes) · **Free**. Rights needed: query = `QueryLimitedInformation` · read/write = `VmRead`/`VmWrite` · alloc/free/protect = `VmOperation`.

**When you'd use it**

- **Injection hunting:** look for private, executable regions that aren't backed by a file on disk.
- **Lab/research:** understand exactly what alloc → write → protect → execute looks like, so you recognise it in EDR telemetry.
- Know which access rights each operation needs, so you can judge how dangerous a granted handle is.

---

## 11. Sections (shared memory & mapped files)

```powershell
$s = New-NtSection -Size 4096 -Protection ReadWrite     # anonymous section
$m = Add-NtSection -Section $s -Protection ReadWrite    # map it
Remove-NtSection -Mapping $m                            # unmap
# Mapping with incompatible protection -> 0xC000004E

Get-NtVirtualMemory -Type Mapped | Where-Object Name -ne ""   # file-backed mappings

$sect = New-NtSectionImage -Win32Path "C:\Windows\notepad.exe"  # map a PE as an image
$map  = Add-NtSection -Section $sect -Protection ReadOnly
Get-NtVirtualMemory -Type Image -Name "notepad.exe"             # per-section regions
Out-HexDump -Buffer $map -ShowAscii -Length 128                 # MZ header
```

Rules: mapping protection is limited by the protection the section was **created** with, _and_ by the handle's rights (`MapRead`, plus `MapWrite` to map it writable). An image map only needs `Execute` on the file.

**When you'd use it**

- **Static PE inspection:** view a binary's headers and section layout as Windows would load them.
- **IPC attack surface:** shared sections between privileged and unprivileged processes are a known bug class (see Worked Example B).
- **[extra] DLL hijack / sideload triage:** list which file-backed images are mapped into a process.

---

## 12. Code integrity

```powershell
Get-AuthenticodeSignature "$env:WinDir\system32\notepad.exe" | Format-List
# Status, SignerCertificate, SignatureType (Embedded vs Catalog), IsOSBinary
```

**When you'd use it**

- **Triage:** check whether a suspicious binary or driver is validly signed, and by whom.
- **[extra]** Sweep for unsigned executables in `System32` or in startup/persistence locations:
    
    ```powershell
    Get-ChildItem C:\Windows\System32\*.exe | Get-AuthenticodeSignature | Where-Object Status -ne 'Valid'
    ```
    
- A **Catalog** signature means the signature is stored in a separate catalog file, not inside the binary. Don't mistake that for "unsigned".

---

## 13. Registry at the native level (configuration manager)

```powershell
ls NtObject:\REGISTRY        # or the shortcut drive: ls NtKey:\
$key = Get-NtKey \Registry\Machine\SOFTWARE\Microsoft\.NETFramework
Get-NtKeyValue -Key $key
# New-NtKey · Set-NtKeyValue · Remove-NtKey · Remove-NtKeyValue
```

**When you'd use it**

- See the _native_ registry layout rather than the Win32 view (HKLM/HKCU are just aliases).
- **[extra] Malware forensics:** native paths can contain characters, such as embedded NULs, that `regedit` and the Win32 APIs can't show. Some malware hides persistence this way. The native APIs can see and remove those keys.

---

## 14. Worked examples (offensive & defensive recipes)

### A. Find open file handles by name ("who has this file open?")

```powershell
$hs = Get-NtHandle -ObjectType File | Where-Object Name -Match Windows
$hs | Select-Object ProcessId, Handle, Name
```

_Use:_ file-lock troubleshooting, IR (which process has the evidence file or the ransom note open), finding processes touching sensitive files. Run as admin for the best coverage. It's slow because each handle has to be duplicated to read its name.

### B. Find writable Section objects shared between privileged and unprivileged processes

```powershell
$ss = Get-NtHandle -ObjectType Section -GroupByAddress | Where-Object ShareCount -eq 2
$mask = Get-NtAccessMask -SectionAccess MapWrite
$ss = $ss | Where-Object { Test-NtAccessMask $_.AccessIntersection $mask }
foreach($s in $ss) {
    $count = ($s.ProcessIds | Where-Object { Test-NtProcess -ProcessId $_ -Access DupHandle }).Count
    if ($count -eq 1) { $s.Handles | Select ProcessId, ProcessName, Handle }
}
```

_Use:_ vulnerability research. This is the pattern behind **CVE-2019-0943**, where the Font Cache service shared a section that a low-privileged process could write to. Expect false positives; it maps attack surface rather than finding bugs on its own.

### C. Map and fuzz a shared section (lab only)

```powershell
$sect   = $handle.GetObject()                               # duplicate into this process
$map    = Add-NtSection -Section $sect -Protection ReadWrite
$random = Get-RandomByte -Size $map.Length
Write-NtVirtualMemory -Mapping $map -Data $random
Out-HexDump -Buffer $map -Length 16 -ShowAddress -ShowHeader
Show-NtSection -Section $sect                               # GUI hex editor
```

_Use:_ quick fuzzing of shared memory. If the privileged process crashes, investigate whether you can control the crash. ⚠️ Writing to a shared section changes **every** process that has it mapped, so only do this on a disposable VM.

### D. Hunt for RWX (read-write-execute) memory

```powershell
$proc = Get-NtProcess -ProcessId $pid -Access QueryLimitedInformation
Get-NtVirtualMemory -Process $proc | Where-Object { $_.Protect -band "ExecuteReadWrite" }
$proc.Close()
```

_Use:_ triage for shellcode injection. Use `-band`, not `-eq`, because flags such as `Guard` can be combined with the protection value. Expect harmless hits from JIT runtimes (.NET, browsers, Java). An RWX region in something like `notepad.exe` deserves a closer look. **[extra — not tested against your build]** To sweep every process you can open:

```powershell
Get-NtProcess -Access QueryLimitedInformation | ForEach-Object {
    $p = $_
    Get-NtVirtualMemory -Process $p -ErrorAction SilentlyContinue |
        Where-Object { $_.Protect -band "ExecuteReadWrite" -and $_.Type -eq "Private" } |
        Select-Object @{N='PID';E={$p.ProcessId}}, @{N='Name';E={$p.Name}}, Address, Size
    $p.Close()
}
```

---

## Quick "I need to…" index

|I need to…|Command|
|---|---|
|Resolve a SID|`Get-NtSid -Sddl <sid>`|
|Decode a hex error|`Get-NtStatus 0x...`|
|Decode a GrantedAccess value|`Get-NtAccessMask 0x... -AsTypeAccess <Type>`|
|See a process's handles|`Get-NtHandle -ProcessId <pid>`|
|Find who locks a file|`Get-NtHandle -ObjectType File \| ? Name -Match <x>`|
|Browse kernel objects|`ls NtObject:\<path>`|
|List loaded drivers|`Get-NtKernelModule`|
|List processes as low-priv|`Get-NtProcess -InfoOnly`|
|View a process's memory map|`Get-NtVirtualMemory -Process <h>`|
|Find RWX memory|`... \| ? { $_.Protect -band "ExecuteReadWrite" }`|
|Check a signature|`Get-AuthenticodeSignature <file>`|
|Read native registry|`Get-NtKey` / `Get-NtKeyValue`|
|Auto-close a handle|`Use-NtObject($x = ...) { ... }`|