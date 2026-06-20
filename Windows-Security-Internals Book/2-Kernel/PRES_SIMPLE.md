



IMAGE INcase 
- ![[Pasted image 20260129165538.png]]
- Inbrave - https://excalidraw.com/ 
- Draw fro mMullvad
This is 


- Windows NTOS kernel executive, or kernel for short, is the heart of  Windows.
- It provides all the operating system’s privileged functionality, as  
well as interfaces through which the user applications can communicate  
with the hardware. The kernel is split into multiple subsystems, each with a  
dedicated purpose. Each subsystem in the kernel executive exposes APIs for other subsys-  
tems to call.

The Kernel API comds use prefixes to convey which subsystem API you are interogateing: eg
- **Se*** — **Security Reference Monitor (SRM)**
- Nt - System Call INterface
- **Ob*** — **Object Manager**
- **Cm*** — **Configuration Manager (Registry)**

INStall James Forshaws cmd let to work with the Systme Obnjects

```
Install-Module NtObjectManager -Scope CurrentUser -Force
```

```
Import-Module NtObjectManager
```

```
New-NtSecurityDescriptor
```

## The security Reference Monitor

For us here Security Reference Monitor (SRM) is the most important subsystem in the kernel. It implements the security mechanisms that restrict which users can access different resources.

Every process running is assigned an access token when it’s  
created. This access token is managed by the **SRM** and defines the identity  
of the user associated with that process. The SRM can then perform an  
_access check_ which queries a resource’s security  
descriptor, compares it to the process’s access token, and either calculates  
the level of granted access or indicates that access is denied to the caller.

(if enabled) The SRM will also generate audit events 


As an example we can interact with the SRM looking up the SID of the Bulltin users group
```
Get-NtSid -Name "Users"
```


## The Object Manager

IN the windows world , most things are referenced as Objects ( as a ppose to Linux where everything is a file). The object manager is the component of the kernel responsible for managing these resource objects, their memory allocations, and their lifetimes.

Each of these objects can have an assigned security descriptor, which restricts  
which users can access the object and determines the type of access these permited users have (for example, read or write).


WE can see there are many types of object

```
Get-NtType
```

### The Object Manager Namespace ( OMNS)
Underneath Windows operating System's the user interface, there is an additional data structure just for kernel objects.  referred to as the Object Manager Namespace

The OMNS is built out of directory Objects but is different from them as they each have security descruiptors which define whihc usrs can list their contents and create new dubdirectories in them

Thanks to James, his powershell module can list like it were a file system
```
ls NtObject:\ | Sort-Object Name
```

```
ls NtObject:\REGISTRY | Sort-Object Name
```


Example of alternative of using the tool 

Turn on  always elevated
```
reg add HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /t REG_DWORD /d 1 /f
```

Check it wit hhis tool
```
Use-NtObject($k = Get-NtKey -Win32Path "HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer") {
  Get-NtKeyValue -Key $k |
    Where-Object Name -eq "AlwaysInstallElevated" |
    Select-Object Name, Type, DataObject
}
```

**Win32 path** uses `HKLM/HKCU` (user-friendly Win32 registry view); **`NtObject:\`** shows the native `\REGISTRY\...` Object Manager namespace.



Turn it off
```
reg delete HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /f
```


## System Call Interface for System Calls

we can call kernel-mode code in a user-mode application using the system call interface. Not going dee pin the weeds. 

A typical System call would mainly contain 
- **Handle**  - to access Properties on the object, Thnk how a door has a handle or a windows has a handle. They are both endpoints where we the user interface to unlock the window or door
- **DesiredAccess** - the operations the caller wants to perform on the handle.  These are dependent on the previously mentioned SRM access-check result. More on that later
- **ObjectAttributes** - a parameter which would go on to define attributes of the object to open or create. These could be in reference to the handle, the object name, conditionals on opening , the scope of where the handle can be open from ( kernel or user space  ) etc etc 

## STatus Codes
All system calls return a 32-bit NTSTATUS code written in Hex


There are many!!!! 
```
Get-NtStatus
```

Look at 
```
(0xC0000034)
```

```
Get-NtDirectory \THISDOESNOTEXIST
```

### Object Handles
Object handles are secure, numeric references to kernel objects, provisioned by the kernel so a process can interact with those objects safely.
 
When a user-mode process requests access to a kernel object, the kernel creates an entry in that process’s **handle table**, which contains:
- The handle’s numeric value (used as a lookup key)
- The granted access rights (e.g. read, write, wait)
- A pointer to the object in kernel memory
### 🔁 Flow:
1. The user-mode app makes a request via the **System Call Interface (SCI)** passing **DesiredAccess** + **ObjectAttributes** + an **out HANDLE***.
2. The SCI invokes a **kernel API** (e.g., `NtCreateXxx`) - Kernel resolves the name (Object Manager namespace) and finds/creates the kernel object.
3. **SRM access check happens here**: it evaluates the caller’s token vs the object’s security descriptor and produces **granted access** (an ACCESS_MASK).
4. - If allowed, the kernel creates a **handle table entry** in that process containing:
    - handle number
    - granted access mask
    - pointer to the kernel object
 5. Kernel returns the **numeric handle value** to user mode (via that out parameter).
 6. **Point 6 (back in user mode):** the syscall returns **NTSTATUS** and (on success) writes the new **HANDLE value** into your out parameter; your code stores it and uses that handle in later syscalls to operate on the object (or handles the error if the status is failure).

![[Pasted image 20260129104617.png]]

```
Get-NtType | Select-Object Name, GenericMapping
```


```
Get-NtTypeAccess -Type File
```


See the orignial Names  form the Windows SDK 
```
Get-NtTypeAccess -Type File | Select SDKName, Value
```




# The Input/Output Manager

The input/output (I/O) manager provides access to I/O devices through device drivers. The primary purpose of these drivers is to implement a filesystem. For example, when you open a document on your computer, the file is made available through a filesystem driver. The I/O manager supports other kinds of drivers, for devices such as keyboards and video cards, but these other drivers are really just filesystem drivers in disguise.

We can list all the Device Drivers here
```
ls NtObject:\Device
```

we don’t interact with the I/O manager  
using dedicated system calls; rather, we use File object system calls such as `NtCreateFile`.

See the exact location on a device of a particualtr rprogram
```
Use-NtObject($f = Get-NtFile "\SystemRoot\notepad.exe"){$f | Select-Object FullPath, NtTypeName}
```


Get a device Type
```
Get-Item NtObject:\Device\HarddiskVolume3
```


List all the device Drivers loaded into memory

```
Get-NtKernelModule
```


Windows has an I/O manager driver, the `Ancillary Function Driver` (AFD), which provides access to networking services for an application. You don’t need to deal with the driver directly; Win32 provides a BSD sockets-style API, called `WinSock`, to handle access to it. 

In addition to the standard internet protocol suite, such as TCP/IP, AFD also implements other network socket types, such as Unix sockets and bespoke Hyper-V sockets for  
communication with virtual machines.


# Process and Thread Manager

All user-mode code lives in the context of a process, each of which has one or  more threads that control the execution of the code.

Unlike most other kernel objects, you can’t open a process  
or thread by name. Instead, you must open them via a unique, numeric process ID (PID) or thread ID (TID).

```
Get-NtProcess -InfoOnly
```

```
Get-NtThread -InfoOnly
```


```
PS C:\Users\Tester> Get-NtProcess -InfoOnly

PID   PPID Name                        SessionId
---   ---- ----                        ---------
0     0    Idle                        0
4     0    System                      0
108   4    Registry                    0
...
...
```


**However the System process, with PID 4**, is important because it runs entirely in kernel mode. When the kernel or a driver needs to execute  a background thread, the thread is associated with the System process.

```
$proc = Get-NtProcess -ProcessId $pid; $proc.CommandLine
```


---

# The Memory Manager - 

The kernel’s `memory manager` subsystem controls the allocation of every processes address space.


**NtVirtualMemory API related Commands**

PowerShell wraps these system calls using the `Get-`, `Add-,` `Read-`, `Write-`,  `Remove-`, and `Set-NtVirtualMemory` commands. Note that these commands all accept an optional Process parameter that lets you access memory in a different process from the current one.

List all the memory Regions used by the current Process. 
```
Get-NtVirtualMemory
```
includes the address of  the memory region, its size, its protection, and its state. 
## 3 memory States

- `Commit` - Its yours
- `Reservere` - booked but not currently available for use
- Free - Unallocated and so will cause a crash (??) , Dont reley on it

We shall allocate a 1,000-byte read/write region and capture the  
address in a variable. We will pass the address to `Get-NtVirtualMemory` allows  
us to query only that specific virtual memory region. 

You might notice that although we requested a 1,000-byte region, the size of the region returned  
is 4,096 bytes. This is because all virtual memory allocations on Windows have a minimum allocation size;
```
$addr = Add-NtVirtualMemory -Size 1000 -Protection ReadWrite; Get-NtVirtualMemory -Address $addr
```

Read out 4 bytes of the memory region  we created above and  find that the bytes are all zeros
```
Read-NtVirtualMemory -Address $addr -Size 4 | Out-HexDump
```

Now write 1,2,3,4 to it
```
Write-NtVirtualMemory -Address $addr -Data @(1,2,3,4)
```
Read it again
```
Read-NtVirtualMemory -Address $addr -Size 4 | Out-HexDump
```

Change protection, make the allocated memory executable by specifying the protection as ExecuteRead
```
Set-NtVirtualMemory -Address $addr -Protection ExecuteRead -Size 4
```

Show that the protection has changed
```
Get-NtVirtualMemory -Address $addr
```

free the memory using Remove-NtVirtualMemory and verify that  the memory is now in the Free state
```  
Remove-NtVirtualMemory -Address $addr; Get-NtVirtualMemory -Address $addr
```

## Section Objects

A `Section object` is a kernel type that implements memory-mapped files. We  
can be used for two related purposes:  
- Reading or writing a file as if it were all read into memory  
- Sharing memory between processes so that the modification in one  process is reflected in the other

| Section Types | Description                                                               |
| ------------- | ------------------------------------------------------------------------- |
| **Anonymous** | No backing file — pure memory. Think: shared memory, heap, etc.           |
| **Mapped**    | Backed by a file — you can read/write the file like it's RAM.             |
| **Image**     | Backed by an executable file (EXE/DLL) — special loader behavior applies. |

Create a Section object with a size of 4,096 bytes and protection of ReadWrite ( Without a File parameter, it’s anonymous and not backed by any file.)
```
$s = New-NtSection -Size 4096 -Protection ReadWrite  
```


Map the section into memory with required  protection, and query the  address to verify success 
```
$m = Add-NtSection -Section $s -Protection ReadWrite 
Get-NtVirtualMemory;$m.BaseAddress
```

TO remove we call Remove-NtSection to unmap the  section and then verify that it’s now free
```
Remove-NtSection -Mapping $m; Get-NtVirtualMemory -Address 0x1C3DD0E0000  
```

demonstrate that we can’t map a section with different protection than that granted when we created the Section object
```
Add-NtSection -Section $s -Protection **ExecuteRead**
```

The protection you’re allowed to use to map a `Section` object into memory depends on two things.
- The first is the protection specified when the `Section` object was created. For example, if the section was created with  `ReadOnly` protection, you can never map it to be writable.  
- The second dependency is the access granted to the section handle you’re mapping. If you want to map the section as readable, then the handle must have `MapRead` access. 

To map it to be writable, you need both `MapRead` and `MapWrite` access. (And, of course, having just MapWrite access isn’t sufficient to map the section as writable if the original Section object was not specified with a writable protection.) It’s possible to map a section into another process by specifying a process handle to Add-NtSection.


A query to find mapped sections that are  backed by files 
```
Get-NtVirtualMemory -Type Mapped | Where-Object Name -ne ""
```


### Image type 
In addition to the `Anonymous` and `Mapped` types, there is a third section  type, the `Image` type.

Create an Image Section from a File. it's just being loaded into memory in a way that _could_ be executed.
```
$sect = New-NtSectionImage -Win32Path "C:\Windows\notepad.exe"
```

**maps the image into your process’s virtual address space**, with **read-only** protection. 
```
$map = Add-NtSection -Section $sect -Protection ReadOnly
```

Query the Memory Region You Just Mapped
```
Get-NtVirtualMemory -Address $map.BaseAddress
```

Search All Image Mappings for Notepad
```
Get-NtVirtualMemory -Type Image -Name "notepad.exe"
```

Hex Dump the Mapped Image
```
Out-HexDump -Buffer $map -ShowAscii -Length 128
```



# The configuration Manager ( the REGISTRY)
The Configuration Manager, known more commonly as the registry, is an  
important component for configuring the operating system. You can think of the registry as a filesystem or `HIVE` in which keys are like folders  and values are like files. You can access it through the **OMNS**, although you must use registry-specific system calls The root of the registry is the **OMNS**  path `REGISTRY`.

Like so 

```
ls NtObject:\REGISTRY
```


ACT: What do these man ??


```
PS C:\Users\Tester> ls NtObject:\REGISTRY

Name    TypeName
----    --------
A       Key
MACHINE Key
USER    Key
WC      Key

```

Why **`REGISTRY`** ? Because it’s a central *registry* of everything the Windows Operating system needs to know:
- What drivers to load
- What services are installed
- Device settings
- App configurations
- Security policies
- Even your user profile data
Just like a civil registry keeps official records for citizens, the Windows Registry keeps official records for software, hardware, and system behavior.

In the kernel:
- Registry keys are **Object Manager key objects**
- Values are stored in **memory-mapped structures**
- The hive files are loaded and parsed at boot
- Kernel APIs like `NtQueryValueKey` interact with it directly



`PowerShell` already comes with a drive provider that you can use to access the registry. However, this drive provider exposes only  the Win32 view of the registry, which hides the internal details about the registry from view.

List the registry in PowerShell using the NtObject drive
```
ls NtObject:\REGISTRY
```
or 
```
ls NtKey:\
```


```
$key = Get-NtKey \Registry\Machine\SOFTWARE\Microsoft\.NETFramework
```


A hive is a hierarchy of Key objects underneath a single root key.


---


# Code Integrity

Microsoft considers the integrity of code running on Windows to be so important that there is an entire subsystem to deal with it. Almost every executable on a default Windows installation is signed using a mechanism called **Authenticode**.

  Query the signing status of an executable,
```
Get-AuthenticodeSignature "$env:WinDir\system32\notepad.exe" | Format-List
```

We can see the ifo about the signers X.509 Cirt, signature status, `SignatureType` etc 

The most common trust decision the code integrity subsystem makes  
is checking whether a kernel driver can load. Each driver file must have a  
signature that derives its trust from a Microsoft-issued key. If the signature  
is invalid or doesn’t derive from a Microsoft-issued key, then the kernel can  
block loading of the driver to preserve system integrity.
