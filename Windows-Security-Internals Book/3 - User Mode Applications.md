
We start with a dive into Win32 APIs and the loading of DLLs. Understanding this topic is important, as it reveals how user-mode applications communicate with the kernel and implement common features. 

We will then have an overview of the Win32 GUI, including a description of the separate system call table used for WIN32K, which is the kernel-mode component of the Win32 subsystem. I introduced the window station and desktop object types and outlined the purpose of the console session, as well as how it corresponds to the desktop you see as a user. 

We will then return to the topic of Win32 APIs by detailing the differences and similarities between a Win32 API (in this case, `CreateMutexEx`) and the underlying system call (`NtCreateMutant`). 
To  deepen understanding of how the Win32 APIs interact with the rest of the operating system. 

We will cover the differences between DOS device paths and native paths as understood by a system call, a topic that is important for understanding how user-mode applications interact with the filesystem. 

We concluded with a discussion of several topics related to Win32 processes and threads, covering the APIs used to create processes directly or through the shell and providing an overview of wellknown system processes. 


# Win32 and the Usermode API
# Win32 GUI
# Comparing Win32 APIs and System Calls
# Win32 Registry Paths
# DOS Device Paths
# Process Creation
# System Processes
# Worked Examples