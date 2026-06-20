
Below is “talk-track + on-screen bullets” you can drop next to each command in your presentation. I’m explicitly basing the explanations on **(a)** the book text you provided (cited) and **(b)** Microsoft / PowerShell official documentation (cited). I’m not inventing behaviors beyond what those sources support.

---

## 0) Setup: install the tooling

```
Install-Module NtObjectManager -Scope CurrentUser -Force
```

- **What this does (PowerShell level):**
    - Downloads and installs the **NtObjectManager** module for _your_ user profile (no admin install needed).
    - `-Force` overwrites/reinstalls if it already exists.
- **Why it matters:**
    - I think this this module was written by James ForShaw  “adds a provider and cmdlets to access the NT object manager namespace” (that hidden kernel-object namespace you’re exploring).      

---

## 1) Kernel “Executive” = Set of subsystems that expose APIs

_(This is concept framing you’ll say before commands.)_
- Windows kernel resources are managed as **objects**, and subsystems (Object Manager, SRM, Memory Manager, etc.) expose APIs for other components to call.
- Microsoft’s driver documentation describes the **Object Manager** as responsible for creation/destruction of objects, keeping a namespace, tracking per-process resources, and tracking access rights.


The KErnele API comds use prefixes to convey which subsystem you are interogateing: eg
- **Se*** — **Security Reference Monitor (SRM)**
- **Ob*** — **Object Manager**
- **Cm*** — **Configuration Manager (Registry)**


---

## 2) Security Reference Monitor basics: SIDs and SDDL

### 2.1 Look up a SID by name

```
Get-NtSid -Name "Users"
```

- **What this does:**
    - Asks Windows for the SID corresponding to the _Users_ group and prints it in **SDDL (Security Descriptor Definition Language) SID string form** (example: `S-1-5-32-545`). ( )
- **What to explain simply:**
    
    - A **SID** is Windows’ numeric identity for a user or group. Names can change; SIDs are what the kernel uses for access checks.
        
- **What to point out on screen:**
    
    - The book breaks `S-1-5-32-545` into parts: `S` prefix, revision `1`, authority `5` (NT authority), and RIDs `32` + `545` (BUILTIN domain + Users).
        

### 2.2 Reverse lookup: SID → name

`Get-NtSid -Sddl "S-1-5-32-545"`
- **What this does:**
    - Converts an SDDL SID string back to the friendly name.
- **Why you care (security angle):**
    - In logs, memory dumps, ACLs, etc., you often see **SIDs**, not names. Being able to translate both ways is essential.    

---
NEXT 
## 3) Object Manager: “everything is an object”

### 3.1 Enumerate kernel object types

```
Get-NtType
```

- **What this does:**
    - Lists kernel **object types** (e.g., `Process`, `Thread`, `Token`, `Section`, etc.).
- **What to explain simply:**
    - Windows doesn’t treat “files” as the universal abstraction (like Unix). Instead, Windows represents many resources as **kernel objects** with types and security descriptors.
- **Why types matter:**
    - Type influences what operations are legal and how access checks are interpreted.

### 3.2 Show per-type access mapping (handles + rights)

```
Get-NtType | Select-Object Name, GenericMapping
```

Those Read Write Execute , All   -  `R:00020000 W:00020000 E:00020000 A:000F0001` values are **hex access masks** (bitmasks) inside a **GENERIC_MAPPING**.

**overall those hex values are “permission bitmasks” (access masks) that the Windows kernel understands and enforces.** More precisely:

- They are **ACCESS_MASK** values: a 32-bit bitfield that represents what operations are allowed on a handle (read/write/delete/synchronize + type-specific ops).
    
- The kernel stores the **granted access mask** in the process’s **handle table entry** and checks it when you use the handle in a system call. If you request an operation your handle wasn’t granted, the kernel fails it (e.g., `STATUS_ACCESS_DENIED`).
    
- The **GenericMapping** values you saw (`R:… W:… E:… A:…`) are the table the SRM uses to convert **generic rights** (`GenericRead/Write/Execute/All`) into the **type-specific + standard rights** for that object type.

- **What this does (conceptually):**
    - Shows how **generic rights** (like “GenericRead”) map onto type-specific rights.
- **Why it matters:**
    - Access checks are fundamentally about an **ACCESS_MASK** bitfield (requested vs granted). The book ties this to what’s stored in the handle table.

### 3.3 Display access rights for a specific type (File)

```
Get-NtTypeAccess -Type File 
```

```
Get-NtTypeAccess -Type File | Select SDKName, Value
```

- **What this does:**
    - Lists the file object’s access rights; second command prints the original “SDK-style” names with numeric values.
- **What to say:**
    - “When you see rights like `ReadData`, `WriteData`, `AppendData`, those aren’t abstract—they’re individual bits in an access mask.”

---

## 4) Object Manager Namespace (OMNS): the “hidden filesystem” of kernel objects

### 4.1 List the OMNS root

`ls NtObject:\ | Sort-Object Name`

- **What this does:**
    
    - Enumerates the **Object Manager Namespace**, presented like a filesystem via a PowerShell drive provider.
        
- **What to explain simply:**
    
    - The OMNS is built from **Directory objects**; it behaves like folders/paths, but it contains kernel objects (not regular files).
        
- **Security relevance:**
    
    - Each directory/object can have a **security descriptor** controlling who can list, create, or open objects there.
        

### 4.2 Demonstrate OMNS symbolic links

`ls NtObject:\Dfs | Select-Object SymbolicLinkTarget Get-Item NtObject:\Device\DfsClient | Format-Table`

- **What this does:**
    
    - Shows a `SymbolicLink` object and the target it points to, then inspects the target object type.
        
- **What to say:**
    
    - “This is like shortcuts—except they live in the kernel namespace and often bridge to device objects.”
        

### 4.3 Quick “well-known directories” framing

- The book highlights common OMNS directories like `\Device`, `\KnownDlls`, `\Sessions`, and `\RPC Control`.
    
- Microsoft separately documents _kernel object namespaces_ concepts (global vs session) for named objects.
    

---

## 5) System calls and NTSTATUS: how user-mode asks the kernel

### 5.1 List NTSTATUS codes

`Get-NtStatus`

- **What this does:**
    
    - Prints known NTSTATUS values and human-readable meanings.
        
- **What to explain:**
    
    - “System calls return **NTSTATUS**, not Win32 `GetLastError` codes. PowerShell surfaces failing NTSTATUS as exceptions.”
        

### 5.2 Decode a specific NTSTATUS

`Get-NtStatus 0xC0000034 | Format-List`

- **What this does:**
    
    - Decodes `STATUS_OBJECT_NAME_NOT_FOUND` and shows fields like Facility and Severity.
        
- **What to say:**
    
    - “This is the kernel’s native error vocabulary; Win32 errors often get _mapped_ from it.”
        

### 5.3 Tie-in: Nt vs Zw naming

- Microsoft documents that most native services have both **NtXxx** and **ZwXxx** names and are serviced by the same underlying kernel routine.
    
- The book adds: in **user mode** they’re equivalent; in **kernel mode** the `Zw` prefix can affect the access-checking process.
    

---

## 6) Object handles: what a handle _is_ and why output means “open”

### 6.1 The key concept (say this once, then demo)

- The book: each process has a **handle table** entry containing:
    
    - handle number
        
    - granted access mask
        
    - pointer to the kernel object
        
- Microsoft: a handle is a process-specific reference used to access objects safely; Windows manages access through the object’s security and the handle’s granted rights.
    

### 6.2 Finding open file handles by name (your script)

`$hs = Get-NtHandle -ObjectType File | Where-Object Name -Match Windows $hs | Select-Object ProcessId, Handle, Name`

- **What this does (step-by-step):**
    
    - `Get-NtHandle -ObjectType File` takes a snapshot of **current handle tables** and returns handles that refer to **File** objects.
        
    - `Where-Object Name -Match Windows` filters to handles whose resolved object **Name** contains “Windows”.
        
    - `Select-Object ...` prints only PID, handle value, and resolved path.
        
- **Why this output demonstrates handles are open:**
    
    - You only get a row if a handle table entry exists _right now_. Handle table entries exist only while the process still has that handle open; closing the handle removes the entry. (That’s the whole point of enumerating handles.)
        
- **Important nuance to mention on screen:**
    
    - Resolving `Name` is “expensive”: the module often must open the target process with **DupHandle**, duplicate the handle into your PowerShell process, then query properties. So you’ll miss processes you can’t open.
        
- **Practical tip (for your demo VM):**
    
    - Run PowerShell as admin for richer results, because you can open more processes for duplication.
        

---

## 7) Worked example: Finding shared Section objects (your “Should I be worried?” output)

### 7.1 The exact script you ran (explained line-by-line)

`$ss = Get-NtHandle -ObjectType Section -GroupByAddress | Where-Object ShareCount -eq 2 $mask = Get-NtAccessMask -SectionAccess MapWrite $ss = $ss | Where-Object { Test-NtAccessMask $_.AccessIntersection $mask }  foreach($s in $ss) {   $count = ($s.ProcessIds | Where-Object {     Test-NtProcess -ProcessId $_ -Access DupHandle   }).Count   if ($count -eq 1) {     $s.Handles | Select ProcessId, ProcessName, Handle   } }`

- **Line 1 (group shared objects):**
    
    - `-GroupByAddress` groups handles by the **kernel object address** (same underlying object, different per-process handle numbers).
        
    - `ShareCount -eq 2` keeps sections referenced by exactly 2 processes.
        
- **Lines 2–3 (find “writable-capable” shared sections):**
    
    - `MapWrite` is the permission needed to map a section writable.
        
    - `AccessIntersection` means “rights common to all handles in this group”; you’re checking whether **every participant** has MapWrite.
        
    - The book warns this is a **proxy**: you can’t directly query the original section protection, and you can get false positives.
        
- **Loop (privilege-boundary heuristic):**
    
    - `Test-NtProcess -Access DupHandle` asks “Can I open this process with rights that let me duplicate its handles?”
        
    - If you can do that for **only one** of the two processes, the book uses this as a heuristic that one process may be “more privileged” than the other (because you can’t open it).
        
- **Why this matters (security story):**
    
    - Shared writable sections can become an attack surface if a privileged process trusts data that a less-privileged process can modify. The book explicitly ties this pattern to real vulns (example: CVE-2019-0943) and also warns about benign cases / false positives.
        
- **Should you be worried about your output (NVIDIA, VMware, etc.)?**
    
    - **From the book’s guidance:** output like this is _only_ a list of **candidates**. It explicitly says you can get false positives and you must inspect the handles / sharing context to decide if it’s a real issue.
        
    - So: **don’t treat “found a shared writable section” as “compromised.”** Treat it as “something to understand.”
        

> If you want a _safe_ follow-up demo for your audience: focus on explaining the heuristic + why it can be benign, rather than trying to “fuzz” privileged processes.

### 7.2 (Optional) What “duplicate handle” means (official doc angle)

- Microsoft: `DuplicateHandle` creates a new handle in another process that refers to the **same underlying object**; changes can be reflected through either handle.
    
- That’s the mechanism this module relies on when it “brings a foreign handle into your PowerShell” to query properties like Name/SD.
    

---

## 8) Processes and threads: IDs vs handles

### 8.1 List processes/threads without opening them

`Get-NtProcess -InfoOnly Get-NtThread -InfoOnly`

- **What this does:**
    
    - Uses system information queries to list processes/threads _without_ needing handles to each process object.
        
- **What to explain simply:**
    
    - A **PID** uniquely identifies a process while it exists.
        
    - A process is an executing program; threads run inside it and share its address space.
        
- **Nice on-screen callout:**
    
    - Book note: PID 0 = Idle, PID 4 = System (kernel-mode background threads often attach here).
        

### 8.2 Open the current process by PID and show command line

`$proc = Get-NtProcess -ProcessId $pid $proc.CommandLine $proc.Win32ImagePath`

- **What this does:**
    
    - Opens a **process handle** to your current PowerShell process and queries its properties.
        
- **Security relevance:**
    
    - Process objects are securable; access is checked against the process security descriptor, and privileges like `SeDebugPrivilege` can affect what access you can get.
        

---

## 9) I/O Manager: devices look like paths too

### 9.1 List device objects in the OMNS

`ls NtObject:\Device`

- **What this does:**
    
    - Shows kernel **Device** objects exposed in the namespace.
        
- **Key teaching point:**
    
    - There isn’t a special “Device syscall”; file APIs/system calls are used to access device paths. The book calls out that we use file object syscalls (e.g., `NtCreateFile`).
        

### 9.2 Open a file via `\SystemRoot` and show the underlying device path

`Use-NtObject($f = Get-NtFile "\SystemRoot\notepad.exe") {    $f | Select-Object FullPath, NtTypeName  } Get-Item NtObject:\Device\HarddiskVolume3`

- **What this does:**
    
    - Opens a handle to `notepad.exe`, then shows it lives at a device path like `\Device\HarddiskVolumeX\Windows\notepad.exe`.
        
- **What to say:**
    
    - “The Object Manager resolves `\SystemRoot` (a symbolic link), then hands off the rest to the I/O manager + filesystem driver.”
        

### 9.3 Enumerate loaded kernel modules (drivers)

`Get-NtKernelModule`

- **What this does:**
    
    - Lists loaded kernel images and their base addresses/sizes.
        
- **Why security folks care:**
    
    - Loaded drivers expand kernel attack surface; visibility here helps triage and forensics. (General principle; command output is from the book.)
        

---

## 10) Memory Manager: regions, protections, and why page size shows up

### 10.1 List virtual memory regions for the current process

`Get-NtVirtualMemory`

- **What this does:**
    
    - Enumerates memory regions: base address, size, state, and protection.
        
- **What to explain simply:**
    
    - Every process has its own virtual address space; protections like `ReadWrite` vs `ExecuteReadWrite` control what you can do with a page.
        

### 10.2 Allocate memory and see it round up to a page

`$addr = Add-NtVirtualMemory -Size 1000 -Protection ReadWrite Get-NtVirtualMemory -Address $addr`

- **What this does:**
    
    - Requests 1000 bytes; Windows allocates at page granularity (commonly 4096 bytes), so you’ll see a larger region. (Your notes describe this behavior; Windows allocation APIs operate in pages.)
        
- **Underlying official concept:**
    
    - This maps to the idea of reserving/committing pages via virtual memory APIs (e.g., VirtualAlloc/VirtualProtect in Win32 terms).
        

### 10.3 Read / write bytes

`Read-NtVirtualMemory -Address $addr -Size 4 | Out-HexDump Write-NtVirtualMemory -Address $addr -Data @(1,2,3,4) Read-NtVirtualMemory -Address $addr -Size 4 | Out-HexDump`

- **What this does:**
    
    - Reads raw bytes from the region, writes new bytes, reads again.
        
- **What to say:**
    
    - “This is the same primitive used by debuggers and memory scanners—read and write virtual memory.”
        

### 10.4 Flip protection to executable (and why that’s notable)

`Set-NtVirtualMemory -Address $addr -Protection ExecuteRead -Size 4 Get-NtVirtualMemory -Address $addr`

- **What this does:**
    
    - Changes page protection so the region can now be executed (but not written).
        
- **Security relevance:**
    
    - W+X (`ExecuteReadWrite`) is a common red flag because it supports self-modifying code and some injection patterns. Microsoft documents page protection constants; the book discusses executable protections as required for code execution.
        

### 10.5 Free it

`Remove-NtVirtualMemory -Address $addr Get-NtVirtualMemory -Address $addr`

- **What this does:**
    
    - Frees the region; querying again should show it as free/unallocated.
        

---

## 11) Section objects: memory-mapped files / shared memory

### 11.1 Create an anonymous (no-file) section and map it

`$s = New-NtSection -Size 4096 -Protection ReadWrite $m = Add-NtSection -Section $s -Protection ReadWrite Get-NtVirtualMemory $m.BaseAddress`

- **What this does:**
    
    - Creates a **Section** kernel object (a mapping source), then maps a view into your process address space.
        
- **What to explain simply:**
    
    - A Section is the kernel primitive behind **memory-mapped files** and **shared memory**.
        
    - Win32 equivalents are `CreateFileMapping` + `MapViewOfFile`.
        

### 11.2 Unmap

`Remove-NtSection -Mapping $m Get-NtVirtualMemory -Address $m.BaseAddress`

- **What this does:**
    
    - Removes the view from your address space; the mapping disappears from the process memory map.
        

### 11.3 Demonstrate protection constraints

`Add-NtSection -Section $s -Protection ExecuteRead`

- **What to say:**
    
    - “You can’t magically map a section with more permissive protections than the section + handle allow.”
        
- **Official kernel-side anchor:**
    
    - Microsoft’s native routine `ZwCreateSection` creates a section object, and mapping a view is done with `ZwMapViewOfSection`.
        

---

## 12) Image sections: how EXEs/DLLs get mapped

`$sect = New-NtSectionImage -Win32Path "C:\Windows\notepad.exe" $map  = Add-NtSection -Section $sect -Protection ReadOnly Get-NtVirtualMemory -Address $map.BaseAddress Get-NtVirtualMemory -Type Image -Name "notepad.exe" Out-HexDump -Buffer $map -ShowAscii -Length 128`

- **What this does:**
    
    - Creates a section backed by an executable image and maps it into memory (read-only here), then shows it among `Image` mappings and dumps bytes.
        
- **What to explain:**
    
    - “Loading a program is largely ‘mapping sections’—the loader maps executable images into memory rather than reading the whole file manually.” (This is consistent with Windows image mapping behavior; the command demonstrates it.)
        

---

## 13) Code Integrity and Authenticode: “is this file signed?”

`Get-AuthenticodeSignature "$env:WinDir\system32\notepad.exe" | Format-List`

- **What this does:**
    
    - Verifies Authenticode signature status and returns signer/cert chain info.
        
- **Official doc anchor:**
    
    - `Get-AuthenticodeSignature` is a documented PowerShell cmdlet for signature verification.
        
- **Kernel security angle (official):**
    
    - Microsoft documents Code Integrity checks during driver loading and that CI validates signatures for kernel-mode drivers.
        

---

## 14) Registry as kernel objects via the OMNS

### 14.1 List registry root in OMNS form

`ls NtObject:\REGISTRY # or ls NtKey:\`

- **What this does:**
    
    - Shows the root registry keys as kernel **Key** objects exposed under the OMNS path `REGISTRY`.
        
- **What to explain simply:**
    
    - “Registry keys are kernel objects; hives attach under these roots.”
        

### 14.2 Open a key and list values

`$key = Get-NtKey \Registry\Machine\SOFTWARE\Microsoft\.NETFramework Get-NtKeyValue -Key $key`

- **What this does:**
    
    - Opens a Key object, then enumerates its values (name, type, data).
        
- **Important clarity point:**
    
    - PowerShell’s built-in `HKLM:\` provider shows a Win32 view; `NtObject:\REGISTRY` / `NtKey:\` shows the **native** object view (more “internals”).
        

---

## 15) A safe way to present “Writable + Executable memory” (defender mindset)

`$proc = Get-NtProcess -ProcessId $pid -Access QueryLimitedInformation Get-NtVirtualMemory -Process $proc | Where-Object { $_.Protect -band "ExecuteReadWrite" } $proc.Close()`

- **What this does:**
    
    - Opens a process with minimal rights needed to enumerate memory regions, then filters for `ExecuteReadWrite`.
        
- **What to explain carefully:**
    
    - W+X can be used by malware, _but is often benign_, e.g., .NET JIT. The book explicitly makes that point.
        
- **Official doc anchor:**
    
    - `PROCESS_QUERY_LIMITED_INFORMATION` is a documented process access right.
        

---

## 16) (Important) How to present the “Modifying a mapped section” example safely

Your notes include:

`$sect = $handle.GetObject() $map = Add-NtSection -Section $sect -Protection ReadWrite $random = Get-RandomByte -Size $map.Length Write-NtVirtualMemory -Mapping $map -Data $random Out-HexDump -Buffer $map -Length 16 -ShowAddress -ShowHeader Show-NtSection -Section $sect`

- **What the book says this is:** a “quick and dirty fuzzer for shared memory” intended to see whether another process mishandles shared data.
    
- **How I recommend you frame it in a talk (to avoid accidental harm):**
    
    - Present it as: **“why shared writable memory is risky in privileged design”** and stop at **inspection** (who shares it, what the rights are).
        
    - If you demo _writing_, do it only against a section you created yourself (same privilege), not third-party or system processes.
        

---

# Quick recap you can say out loud

- “The OMNS is a kernel-object namespace exposed as a PowerShell drive.”
    
- “A handle is an entry in a per-process handle table: value + granted rights + pointer to the kernel object.”
    
- “So enumerating handles shows _currently open_ references.”
    
- “Shared writable sections can be benign IPC—or an attack surface—so results are candidates, not proof of compromise.”
    

---

If you want, I can convert this into a **scene-by-scene mini script** (intro sentence + 3 bullets + command + “what you should see”) for a 8–12 minute video, using only the commands that are lightweight on a Windows 10 Education VM.