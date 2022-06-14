# Linux rootkits

## Rootkits
> From Wikipedia:
> A rootkit is a collection of computer software, typically malicious, 
> designed to enable access to a computer or an area of its software that is not otherwise allowed 
> (for example, to an unauthorised user) and often masks its existence or the existence of other software.

## Hooking syscall(s)

Most of the rootkits used in the malware attacks have almost the same behaviour (hiding and hooking) as a normal process running in the system.

### Some ways:
	1. [Syscal table hijacking - a good old way.](#Syscall-table-hijacking)
	2. [Sys_close - Brute force method.](#Sys-close)
	3. [VFS hooking.](#VFS-hooking)
	4. [The ftrace helper method.](#Thee-ftrace-helper-method)

#### Syscall table hijacking
** [LKM](https://en.wikipedia.org/wiki/Loadable_kernel_module) - Loadable Kernel Module **
Before digging deep, I would like to describe some basic concepts. LKM is an object file that can be inserted into a running kernel. This is largely used for expanding the Kernel's functionality (device drivers, filesystem, etc.).
Another use could be creating a rootkit that will operate inside the kernel.
In other words, if we want to add code to a Linux Kernel, the most basic way to do that is to add some source files to the kernel source tree and recompile the kernel.
But we can also add code to the Linux Kernel while it's running, and it's called Loadable Kernel Modules.

An LKM can do:
	1) device drivers.
	2) filesystem drivers.
	3) system calls.

LKMs in linux are loaded/unloaded by the [modprobe](https://en.wikipedia.org/wiki/Modprobe) command. They are located in `/lib/modules` or `/usr/lib/modules` and have the extension `.ko` ("kernel object") since version 2.6.
The `[lsmod](https://en.wikipedia.org/wiki/Lsmod)` can be used to list the loaded LKMs.

** [Protection rings](https://en.wikipedia.org/wiki/Protection_ring) **
An OS has two modes, `user mode` and `kernel mode` which are defined by the protection rings.
Protection rings is the hierarchy architecture of privileges in a system. There are typically four rings (ring-0, ring-1, etc) and more we go down, the more privilege we are.
In the most modern OSs, there is actually two modes of those rings, ring-0 (kernel mode) and ring-3 (user mode).
A process that runs in the kernel-mode, has access to all the system resources, including the most “sensitive” parts, which may cause a system crash when kernel-mode process access to the wrong resource or if it is just crashes from one reason or another.
However, user-mode processes have very limited resources access, so it won’t access any sensitive resource.
that was designed that way to protect (hence the name, protection rings) the user from making big mistakes.

When talking about rootkits, we should differ it into two different modes, kernel-mode and user-mode.
Kernel-mode rootkits run with kernel privileges, and user-mode rootkits run with user privileges.
A user-mode rootkit can change binaries like ls, ss, cat etc.
A user-mode rootkit may also hook dynamically linked libraries to change the behaviour of certain functions.
A kernel-mode rootkit, however, can do much more thanks to the privileges it has, things like changing kernel level function pointers, changing kernel code, manipulate important data structures and most important, hooking system calls.

** System calls **

The kernel acts as a translator between users and the machine, each time we want our machine to do something we talk to the kernel and the kernel would translate our demand to the machine, these communication is possible thanks to the system calls.
A system call is a function in the kernel that is also visible to the user.
When a user needs a service from the kernel, it asks the kernel to execute a system call.

`open(), read(), write(), close(), etc` are some examples of system calls.

** Anyway what's the matter **
Imagine we could change the `read()` system call in a way that every time a user want to read a stream of bytes, it will only read the bytes we want it to read.
That way we could hide secrets data in every file and the user won't even know about it.

** The system calls table **
Basically the system calls table is an array in the kernel which holds a pointer to all the system calls the OS has to offer.

```c
void	*sys_call_table[NR_syscalls] = {
	[0 ... NR_syscalls-1] = sys_ni_syscall,
	#include <asm/unistd.h>
}
```
[More info](http://faculty.nps.edu/cseagle/assembly/sys_call.html)

The `NR_syscalls` is a macro that defined in the kernel which holds the maximum number of allowed syscalls.
 Also, all the elements in the syscall table are initialise to `sys_ni_syscall`.

