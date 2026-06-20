
The kernel consists of many separate subsystems, such as:
- the Security Reference Monitor, 
- the object manager, 
- the configuration manager (or  registry), 
- the I/O manager, 
- and the Process and Thread manager.  

We look at 
- how the `Object Manager` manages kernel resources  
and types, 
- how to access kernel resources through system calls,
- how handles are allocated with specific access rights. 
- You also accessed object manager resources through the NtObject drive provider as well as through  individual commands.  

I then  the basics of process and thread creation and demonstrated the use of commands such as `Get-NtProcess` to query for process  information on the system. I explained how to inspect the virtual memory of a process, as well as some of the individual memory types.


# The Windows Kernel Executive
The `Windows NTOS kernel executive`, or **kernel** for short, is the heart of
Windows.

The Windows Kernel Executive
The Windows NTOS kernel executive, or kernel for short, is the heart of
Windows.

![[Windows-KErnal-Executive-modules.png]]

Each subsystem in the kernel executive exposes APIs for other subsystems to call. IF you are looking at kernel code, you can quickly determine what subsystem each API belongs to using its two-character prefix.

**API Prefix-to-Subsystem Mappinhg**

| **Prefix** |       **Subsystem**        |        **Example**        |
| :--------: | :------------------------: | :-----------------------: |
|  Nt or Zw  |   System call interface    |  NtOpenFile / ZwOpenFile  |
|     Se     | Security Reference Monitor |       SeAccessCheck       |
|     Ob     |       Object manager       | ObReferenceObjectByHandle |
|     Ps     | Process and thread manager |    PsGetCurrentProcess    |
|     Cm     |   Configuration manager    |    CmRegisterCallback     |
|     Mm     |       Memory manager       |       MmMapIoSpace        |
|     Io     |    Input/output manager    |       IoCreateFile        |
|     Ci     |       Code integrity       |   CiValidateFileObject    |


# The Security Reference Monitor
`Se` hence why all the Privs start with Se; like `SeImpersonate`

`SIDs` represent users and groups and that we can represent them as strings in `SDDL` form.

Every process running on the system is assigned an access token when it’s
created. This access token is managed by the SRM and defines the identity
of the user associated with that process.

The task of `name–to-SID`conversion is handled by the Local Security Authority Subsystem (LSASS), which runs inside a privileged process independent from any logged-in users.

It’s infeasible to represent every possible SID as a name, so Microsoft
defines the Security Descriptor Definition Language (SDDL) format to represent
a SID as a string. SDDL can represent the entire security descriptor of a
resource, but for now we’ll just use it to represent the SID. 
We can use PowerShell to look up the Users group name using the `Get-NtSid` com-
mand; this should retrieve the SDDL string for the SID.

```
Get-NtSid -Name "Users"

Name          Sid
----          ---
BUILTIN\Users S-1-5-32-545
```

This returns the fully qualified name, with the local domain `BUILTIN` attached. The
`BUILTIN\Users` SID is always the same between different Windows systems.

The output also contains the SID in SDDL format, which can be broken
down into the following dash-separated parts:
• The `S` **character prefix**. This indicates that what follows is an SDDL SID.
• The **version of the SID structure** in decimal. This has a fixed value of `1`.
• The **security authority**. Authority `5` indicates the built-in NT authority.
• Two **relative identifiers** (**RIDs**), in decimal. The RIDs (here, `32` and `545`)
represent the NT authority group.


We can do the reverse whti hthe `Get-NtSid` cmdlt
```
Get-NtSid -Sddl "S-1-5-32-545"
```


## The Object Manager
On Unix-like operating systems, everything is a file. On Windows, every-
thing is an object, meaning that every file, process, and thread is repre-
sented in kernel memory as an object structure. **The object manager is the component of the kernel responsible for managing these resource objects, their memory allocations, and their lifetimes.**

**Security Descriptor** - each of these objects can have an assigned `security descriptor`, which restricts which users can access the object and determines the type of access they have (for example, read or write).  The **object manager** is the component of the kernel responsible for man-
aging these resource objects, their memory allocations, and their lifetimes.

**Object Types** -  The kernel maintains a list of all the types of objects it supports. This is necessary, as each object type has different supported operations and security properties. 
```
Get-NtType
```
**Noteworthy entries**
- Type
- PRocess
- Thread

### The Object Manager Namespace

Underneath the user interface is a whole additional filesystem just for kernel objects. Access to this filesystem, referred to as the **Object Manager Namespace (OMNS)**, isn’t very well documented or exposed to most developers, which makes it even more interesting. The **OMNS** is built out of **Directory objects.** 

The objects act ***as if*** they were in a filesystem, so each directory contains other objects, which you can consider to be files. However, they are distinct from the file directories you’re used to. Each directory is configured with a security descriptor that determines which users can list its contents and which users can create new subdirectories and objects inside it with :

The Object Manager Namespace is like an invisible folder system in the kernel, but instead of files, it holds memory sections, handles, and symbolic links — all critical to the runtime of Windows.

```
ls NtObject:\ | Sort-Object Name
```


We can also see another important type, `SymbolicLink`. You can use symbolic links to redirect one OMNS path to another. A `SymbolicLink` object contains a `SymbolicLinkTarget` property, which itself contains the target that the link should open.
```
ls NtObject:\Dfs | Select-Object SymbolicLinkTarget
```

Below, we list the \Dfs OMNS path, then extract the `SymbolicLinkTarget`
property to get the real target. Next, we check the target path, `Device\ DfsClient`,
to show it’s a Device type, which is what the symbolic link can be
used to access.
```
Get-Item NtObject:\Device\DfsClient | Format-Table
```

We can see all objects related to the `\Dfs` Directory File System

```
ls NtObject:\Dfs | Select-Object *
```

Even though the Dfs stands for Directory File System, it is not a file system, It is a OMNS listing for the OBject representation of the OMNS itself. A bit confuseing but it but **Object Manager Namespace (OMNS)** isn’t arranged like a tree because it _needs_ to be...  
It’s arranged that way so **we can understand it** — reason through it — **and find things**.

### Well-Known Object Directories and Descriptions

| Path              | Description    -                                                   |                                                     |
| ----------------- | ------------------------------------------------------------------ | --------------------------------------------------- |
| \BaseNamedObjects | Global directory for user objects                                  |                                                     |
| \Device Directory | containing devices such as mounted filesystems                     |                                                     |
| \GLOBAL??         | Global directory for symbolic links, including drive mappings      |                                                     |
| \KnownDlls        | Directory containing special, known DLL mappings                   |                                                     |
| \ObjectTypes      | Directory containing named object types                            |                                                     |
| \Sessions         | Directory for separate console sessions                            |                                                     |
| \Windows          | Directory for objects related to the Window Manager                |                                                     |
| \RPC              | Control Directory for remote procedure call endpoints              |                                                     |
|                   |                                                                    |                                                     |



## System Calls

How can we access the named objects in the **OMNS** from a user-mode  
application? If we’re in a user-mode application, we need the kernel to  
access the objects, and we can call kernel-mode code in a user-mode  
application using the system call interface. Most system calls perform  
some operation on a specific type of kernel object exposed by the object  
manager.

As an example of a an object created to access the KErnel space, A Mutant object is a kernel synchronization object that enforces mutual exclusion across threads and processes, with ownership tracked by the kernel.

Ses Script in Notes


**Object Attribute Flags and Descriptions**

| PowerShell Name               | Description                                                                               |
| ----------------------------- | ----------------------------------------------------------------------------------------- |
| `Inherit`                     | Marks the handle as inheritable                                                           |
| `Permanent`                   | Marks the handle as permanent                                                             |
| `Exclusive`                   | Marks the handle as exclusive if creating a new object; only the same process can open it |
| `CaseInsensitive`             | Looks up the object name in a case-insensitive manner                                     |
| `OpenIf`                      | If using a Create call, opens a handle to an existing object if available                 |
| `OpenLink`                    | Opens the object if it’s a link; otherwise, follows the link (used by the config manager) |
| `KernelHandle`                | Opens the handle as a kernel handle in kernel mode; user-mode apps can't access it        |
| `ForceAccessCheck`            | In kernel mode, forces all access checks even with `Zw*` calls                            |
| `IgnoreImpersonatedDeviceMap` | Disables the device map when impersonating                                                |
| `DontReparse`                 | Indicates not to follow paths that contain a symbolic link                                |
|                               |                                                                                           |


### NTSTATUS Codes


```
Get-NtStatus
```

```
Get-NtStatus 0xC0000034 | Format-List
```



### Object Handles
> Object handles are secure, numeric references to kernel objects, provisioned by the kernel so a process can interact with those objects safely.
> 
When a user-mode process requests access to a kernel object, the kernel creates an entry in that process’s **handle table**, which contains:
- The handle’s numeric value (used as a lookup key)
- The granted access rights (e.g. read, write, wait)
- A pointer to the object in kernel memory
    
### 🔁 Flow:
1. The user-mode app makes a request via the **System Call Interface (SCI)**
2. The SCI invokes a **kernel API** (e.g., `NtCreateXxx`) to process the request
3. The kernel **validates** the access request via the **Security Reference Monitor**
4. If approved, the kernel **creates or opens** the object
5. A handle to an `Access Mask` (a number) is returned and stored in the process’s handle table. but the access mask its self is stored in the kernel. The process know knows where to find it when it needs it . 

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

The input/output (I/O) manager provides access to I/O devices through device drivers. The primary purpose of these drivers is to implement a filesystem. For example, when you open a document on your computer, the file is made available through a filesystem driver. The I/O manager supports other kinds of drivers, for devices such as keyboards and video cards, but 7these other drivers are really just filesystem drivers in disguise.


```
ls NtObject:\Device
```

we don’t interact with the I/O manager  
using dedicated system calls; rather, we use File object system calls such  
as NtCreateFile.

See the exact location on a device of a particualtr rprogram
```
Use-NtObject($f = Get-NtFile "\SystemRoot\notepad.exe"){$f | Select-Object FullPath, NtTypeName}
```


Get a device Type
```
  
Get-Item NtObject:\Device\HarddiskVolume3
```


List all the device Drivers loaded into memopry

```
Get-NtKernelModule
```


Windows has an I/O manager driver, the Ancillary Function Driver  
(AFD), which provides access to networking services for an application. You  
don’t need to deal with the driver directly; Win32 provides a BSD sockets-  
style API, called WinSock, to handle access to it. In addition to the standard  
internet protocol suite, such as TCP/IP, AFD also implements other net-  
work socket types, such as Unix sockets and bespoke Hyper-V sockets for  
communication with virtual machines.


# Process and Thread Manager

All user-mode code lives in the context of a process, each of which has one or  more threads that control the execution of the code.
Unlike most other kernel objects, you can’t open a process  
or thread by name. Instead, you must open them via a unique, numeric pro-  
cess ID (PID) or thread ID (TID).

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

The  Idle process, with PID 0, contains threads that execute when  
the operating system is idle, hence its name. Not to important for this. **However the System process, with PID 4**, is important because it runs entirely in kernel mode. When the kernel or a driver needs to execute  a background thread, the thread is associated with the System process.


```
$proc = Get-NtProcess -ProcessId $pid; $proc.CommandLine
```


# The Memory Manager

The kernel’s `memory manager` subsystem controls the allocation  
of this address space.


**NtVirtualMemory API related Commands**
PowerShell wraps these system calls using the `Get-`, `Add-,` `Read-`, `Write-`,  `Remove-`, and `Set-NtVirtualMemory` commands. Note that these commands allaccept an optional Process parameter that lets you access memory in a different process from the current one.

## 3 memory States
- Commited - ITsa yours
- Reservere - booked but not currently availible for use
- Free - Unallocated and so will cause a crash (??)


List all the memoryt Regions used by the current Process. 
```
Get-NtVirtualMemory
```
includes the address of  the memory region, its size, its protection, and its state. There are three  possible state values:
- Commitr


Allocate a 1,000-byte read/write region and capture the  
address in a variable. Passing the address to Get-NtVirtualMemory allows  
us to query only that specific virtual memory region. You might notice that  
although we requested a 1,000-byte region, the size of the region returned  
is 4,096 bytes. This is because all virtual memory allocations on Windows  
have a minimum allocation size;
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


# Advanced Local Procedure Call (ALPC)
ALPC provides the underlying  transport for local remote procedure call APIs implemented in Windows. The OMNS is where the ALPC lives. It is used for implements local, crossprocess communication. 


# The configuration Manager ( the REGISTRY)
The Configuration Manager, known more commonly as the registry, is an  
important component for configuring the operating system. You can think of the registry as a filesystem or `HIVE` in which keys are like folders  and values are like files. You can access it through the **OMNS**, although you must use registry-specific system calls The root of the registry is the **OMNS**  
path `REGISTRY`.


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


# Worked Examples ( Test Cases)

### Finding Open Handles by Name
This is **enumerating the _current_ per-process handle tables** (a snapshot), and a handle table entry only exists while that handle is still open so all the out put is of open handles

```
$hs = Get-NtHandle -ObjectType File | Where-Object Name -Match Windows;$hs | Select-Object ProcessId, Handle, Name
```

This script queries for all File object handles and filters them to only  
the ones with the string Windows in the Name property, which represents the  
filepath.

### Finding Shared Objects

Where an object is shared between two processes with different privileges. The lower-privileged process might be able to modify the properties of the object to bypass security checks in the higher-privileged process, enabling it to gain additional privileges. eg : CVE-2019-0943 - a privileged process, the `Windows Font Cache`, that shared section handles with a low-privileged process. This effectively allowed the low-privileged process to modify arbitrary mem-  
ory in the privileged process, resulting in privileged code execution.


##### Steps
First get the handles, specifying the `GroupByAddress` parameter. This returns a list of groups organized based on the kernel object address,  instead of a list of handles.  Here, we filter to include only  handles that are shared between two processes.

```
$ss = Get-NtHandle -ObjectType Section -GroupByAddress | Where-Object ShareCount -eq 2
```


We want to find Section objects that can be mapped as writable.  We first check that all the handles have `MapWrite` access. We use the `AccessIntersection` property, which contains the granted access rights shared among all the handles.
```
$mask = Get-NtAccessMask -SectionAccess MapWrite
```

Next, we want to find Section objects that can be mapped as writable.
```
$ss = $ss | Where-Object { Test-NtAccessMask $_.AccessIntersection $mask }
```

With potential candidates for shared sections, we check  which meet the criterion that we can access only one of the processes containing the section handle.
```
	foreach($s in $ss) {  
	$count = ($s.ProcessIds | Where-Object {  
		Test-NtProcess -ProcessId $_ -Access DupHandle  
	}).Count  
	if ($count -eq 1) {  
		$s.Handles | Select ProcessId, ProcessName, Handle  
	}
}
```


We might get output looking like :
```
ProcessId ProcessName Handle  
--------- ----------- ------  
9100 Chrome.exe 4400  
4072 audiodg.exe 2560
```

Here i a section shared between Chrome and the Audio Device Graph process. The shared section is used to play audio from the browser, and it’s probably not a security issue.

Modifying a Mapped Section


### Modifying a Mapped Section
The simplest approach modify a discovered and interesting mapped memory from the cli is with  the `Write-NtVirtualMemory` command, which supports passing a mapped section and an array of bytes to write. Below demonstrates this technique by assuming you have a handle of interest in the `$handle` variable.


Below the hope is that by modifying the memory, the privileged process will mishandle the contents of the memory region. If the privileged process crashes, we should investigate to determine whether we can control the crash using a more targeted modification of the shared memory.

First call the `GetObject` method on the handle to duplicate it into the current process and return a `Section` object. Note: the process  in which we’re running this command must be able to access the process with the handle.
```
$sect = $handle.GetObject()
```

Map the handle as ReadWrite into the current process’s memory.
```
$map = Add-NtSection -Section $sect -Protection ReadWrite
```

Create a random array of bytes up to the size of the mapped section ...
```
$random = Get-RandomByte -Size $map.Length
```

... and write them to the memory region
```
Write-NtVirtualMemory -Mapping $map -Data $random
```


```
Out-HexDump -Buffer $map -Length 16 -ShowAddress -ShowHeader
```

If the privileged process crashes, we should investi-  
gate to determine whether we can control the crash using a more targeted  
modification of the shared memory.

A useful features of this command over the built-in `Format-Hex` is that it’ll print the  
address in memory based on the mapped file, whereas `Format-Hex` just prints  
an offset starting at 0.



Bonus : You can also create a GUI hex editor with the Show-NtSection command,  
specifying a Section object to edit.
```
Show-NtSection -Section $sect
```

### Finding Writable and Executable Memory

In Windows, for a process to execute instructions, the memory must be marked as executable. However, it’s also possible to map the memory as both writable and executable. Malware sometimes uses this combination of permissions to inject shell code into a process and run malicious code using the host process’s identity.

Start by opening a process for `QueryLimitedInformation` access, which is  
all we need to enumerate the virtual memory regions. Here, we’re opening the current PowerShell process; as PowerShell is .NET, we know it will have some writable and executable memory regions,
```
$proc = Get-NtProcess -ProcessId $pid -Access QueryLimitedInformation
```

Then enumerate all the memory regions using `Get-NtVirtualMemory` and filter on the `ExecuteReadWrite` protection type.
```
Get-NtVirtualMemory -Process $proc | Where-Object {  
$_.Protect -band "ExecuteReadWrite"  
}
```

```
$proc.Close()
```