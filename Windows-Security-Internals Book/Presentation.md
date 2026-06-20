

The kerne


```
Windows Kernel Executive
│
├─ Security Reference Monitor
│    └─ Enforces access and security
│
├─ Object Manager
│    ├─ Tracks named kernel objects
│    └─ Provides object namespace
│
├─ System Calls
│    └─ Interface to access kernel operations
│
├─ I/O Manager
│    └─ Handles all input and output
│
├─ Process/Thread Manager
│    └─ Manages processes and threads
│
└─ Memory Manager
     └─ Allocates and manages memory
```

![[Windows-KErnal-Executive-modules.png]]







#### Windows Kernel Executive
│
├─ Security Reference Monitor
│    └─ Enforces access and security - PErfoms operations called Access Check on a target resources Security Descryptor. It then provides the access level ( if any ) or a denial. SRM has auditing . INput is a SID . LSASS does the conversation of SID-Name ( and many other things) 
│
├─ Object Manager
│    ├─ Tracks named kernel objects
│    └─ Provides object namespace
│    └─ MAnages resource objects, memory allocations and their lifetimes.
│    └─ Contains the OMNS ( data structure of objects ) 

│
├─ System Calls
│    └─ Interface to access kernel operations
│
├─ I/O Manager
│    └─ Handles all input and output
│
├─ Process/Thread Manager
│    └─ Manages processes and threads
│
└─ Memory Manager
     └─ Allocates and manages memory

