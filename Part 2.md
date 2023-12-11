# Analyzing Hyperion - Part 2: Core Components

## Introduction

> ![IMPORTANT]
> Attention: This is part 2 of the series, if you haven't already, make sure you read [Analyzing Hyperion - Part 1: Starting Out](Part%201.md) first.

Previously, we discussed what Hyperion is and how it differs from traditional anti-cheats.
In this part, we'll analyze their module, learn about the obfuscation techniques they use, examine their initialization routine, and discover how they block the creation of external threads.

## Credits

This thread was co-authored by the following user(s):
    - [gogo1000](gogo1000)

## Table of Contents

- [Analyzing Hyperion - Part 2: Core Components](#analyzing-hyperion---part-2-core-components)
  - [Introduction](#introduction)
  - [Credits](#credits)
  - [Table of Contents](#table-of-contents)
  - [Analysis](#analysis)
    - [Main obfuscations](#main-obfuscations)
    - [Other Obfuscations](#other-obfuscations)
      - [Initialization Routine](#initialization-routine)
      - [Instrumentation Callback](#instrumentation-callback)
      - [LdrInitializeThunk](#ldrinitializethunk)
  - [Overview](#overview)

## Analysis

### Main obfuscations

In this section, we'll start our initial analysis of Hyperion's binary. We'll look at the obfuscation techniques they utilize, and how to circumvent them when reversing.
First, we'll open the binary in IDA, initially you'll see that all of the executable code is in a .byfron section; as you may have already noticed, Hyperion is not packed, it also doesn't employ any sort of virtualization. So what do they do?

Let's start out by just clicking on a couple of functions. No matter which function you've chosen, as long as it has control flow, you'll see that IDA will likely fail to disassemble it:
Hyperion implements a few variations of this across all of their functions. This is the most common form of obfuscation that you will see throughout your analysis, and it's easily recognizable.
The seemingly "fake" instruction sequences are also not arbitrary, but follow specific sequences that repeat throughout the binary.
Along with this are actual fake instructions, as in dead code, which is present throughout the binary.

![image](assets/images/Part%202/disasm-fail-control-flow.png)

Some functions have a jump to a register, where the register is a decrypted address being the instruction proceeding the jump. This is done to break control flow analysis by creating an unconditional jump to an "unknown" location (the instruction after the jump as shown below):

![image](assets/images/Part%202/disasm-fail-control-flow-mem.png)

In the above image, the "jmp rcx" jumps to the proceeding instruction.
Above it, rax is a jump table, rcx is the hard-coded offset for the next instruction's relative address, and 4 is the size of each entry in the table.

Some functions unnecessarily store excessive amounts of opaque values on the stack, this is present even in the smaller functions and makes their stack frame massive:
![image](assets/images/Part%202/massive-stack-frame.png)

### Other Obfuscations

Now that we've covered some of the main obfuscation techniques that Hyperion uses, let's look at some others.

First, you should know that rather than calling exported functions, Hyperion is mainly prone to invoking inlined syscalls, this means many of the exported functions you may have been trying to debug won't be called because they're explicitly invoking syscalls rather than directly calling them.

Now, that isn't to say that they don't call any imports at all, but those that they do call are encrypted then decrypted at runtime, which you will see examples of ahead. Similarly to these encrypted imports, many of the important pointers that Hyperion stores are also encrypted then decrypted at runtime, that'll become evident once you start analyzing the hook routines placed on thread initialization.

Let's go back to the inlined syscalls, for now, you can ignore the location of the syscall shown here, this is what it looks like:
![image](assets/images/Part%202/inlined-syscalls.png)

As you can see in the image above, rbx holds the address of their stub, I'll cover how they calculate the stub address in the following section.
Here's what that stub looks like:
![image](assets/images/Part%202/inlined-syscalls-2.png)

After you follow the call, you can see that the syscall invoker stub is written to an RX allocation, or should I say: a view?

Let's check out how the syscall invoker stub is acquired, remember how I said it's a view? Let's go back to the loop that's located right before the "call rbx."

![image](assets/images/Part%202/inlined-syscalls-3.png)

The first thing you should notice here is 2 writes that they perform on [rsi + rcx].
As you can see, rsi holds the base address, and rcx holds the offset. You might be wondering, "what is rsi?"
Well, upon looking at it in a debugger, you can see that rsi is actually the pointer to the syscall invoker stub itself...
That's weird, it looks like it's writing over the RX allocation for the syscall invoker, but that allocation didn't have any write permission, so how do they do it?

If you look closely, you can see that while the code appears to be the same, they aren't actually the same address, that's what I mean by view.
They have 2 separate allocations that are both mapped to the same physical address, but have 2 separate permissions.
The allocation currently being written to is RW, and the one used in the calls is RX. Here, we can see a structure containing both of these views:
![image](assets/images/Part%202/address-view-1.png)

So, going back to the syscall invocation. Before they perform the call to the stub, they move the RX view into rbx, it looks like this: mov rbx, [r12 + 08].
It should also be known that they only use one stub, and prior to calls they update the syscall ID, that's what the loop does.

Here's the writes that occurred during their LdrInitializeThunk hook, which I'll discuss later:

![image](assets/images/Part%202/LdrInitializeThunk-occurances.png)

That mainly covers the dynamic obfuscation techniques that are used. There are more, but they don't deserve their own separate discussion, they'll be covered in reversals for the next parts.

#### Initialization Routine

In this section I'll be covering the first routines that Hyperion invokes, these are called prior to Roblox's entry point.
I should note that there are a lot of things that happen during initialization, I won't be covering all of them, just the important parts.

As stated in [part 1](Part%201.md), one of the first things that Hyperion does is load the modules that Roblox imports functions from, and while that might be pretty obvious considering that Roblox only has one import (Hyperion's dll), something else happens in this stage.

If you look at the memory regions of the imported modules, you might notice that some of them aren't mapped normally.

![image](assets/images//Part%202/init-routine-map-user32.dll.png)
The picture above shows the memory regions for where user32.dll is mapped. As you can see, the allocation type is mostly private, and part of the executable code is mapped.
Normally, loaded modules aren't copied into the private working set of the process, instead they're shared until a process writes into them, which invokes CoW (Copy on Write), but that isn't the topic of this thread.
Basically, this is one of the modules that Hyperion "protects," the idea is that you can't patch its code, and if you try to, you will get an error while trying to change its protection to writable.

"Why is this, and how is it possible" is a question that you may be asking yourself.
Well, analyzing the function responsible for protecting the module is no easy task, it's massive and has a lot of completely "unrelated" code, which makes manual static analysis nearly impossible.

So, for this I'll be using my hypervisor and exit on each syscall fired by Roblox during the execution of this function.
To do this, I set a trap at the beginning to enable my logger, and a trap at the end to stop tracing.
These are the logs captured by my hypervisor during the execution of this function:

```log
07\07\2023-20:43:23:823 {
    "Process": "RobloxPlayerBeta",
    "Message": "Syscall: NtOpenSection (55) args: [ (PHANDLE) 0xad8b1fc938, ([SECTION_ACCESS_MASK]) SECTION_QUERY, (POBJECT_ATTRIBUTES) path(46): \"\\KnownDlls\\kernel32.dll\" ]"
}
07\07\2023-20:43:23:823 {
    "Process": "RobloxPlayerBeta",
    "Message":
        "Syscall:NtQuerySection (81) args: [ (HANDLE) 0x130, (SECTION_INFORMATION_CLASS) SectionImageInformation, (PVOID) 0xad8b1fd130, (SIZE_T) 0x40, (PSIZE_T) 0x0 ]"
}
07\07\2023-20:43:23:823 {
    "Process": "RobloxPlayerBeta",
    "Message": "Syscall: NtQueryVirtualMemory (35) args: [ ([ProcessHandle]) GetCurrentProcess(), (PVOID) 0x7ffaa1e176b0, (MEMORY_INFORMATION_CLASS) MemoryBasicInformation, (PVOID) 0xad8b1fd1a0, (SIZE_T) 0x30, (PSIZE_T) 0x0 ]"
}
07\07\2023-20:43:23:823 {
    "Process": "RobloxPlayerBeta",
    "Message": "Syscall: NtQueryVirtualMemory (35) args: [ ([ProcessHandle]) GetCurrentProcess(), (PVOID) 0x7ffaa1e00000, (MEMORY_INFORMATION_CLASS) MemoryMappedFilenameInformation, (PVOID) 0x26de54821d0, (SIZE_T) 0x218, (PSIZE_T) 0xad8b1fb900 ]"
}
07\07\2023-20:43:23:823 {
    "Process": "RobloxPlayerBeta",
    "Message": "Syscall: NtOpenFile (51) args: [ (PHANDLE) 0xad8b1fb868, ([FILE_ACCESS_MASK]) FILE_READ_DATA/FILE_LIST_DIRECTORY | FILE_GENERIC_READ | FILE_GENERIC_WRITE | FILE_GENERIC_EXECUTE, (POBJECT_ATTRIBUTES) path(106): \"\\Device\\HarddiskVolume7\\Windows\\System32\\kernel32.dll\", (PIO_STATUS_BLOCK) 0xad8b1fb780, ([FILE_SHARE_MODE]) FILE_SHARE_READ, ([NtCreateOptions]) FILE_SYNCHRONOUS_IO_NONALERT | FILE_NON_DIRECTORY_FILE ]"
}
07\07\2023-20:43:23:824 {
    "Process": "RobloxPlayerBeta",
    "Message": "Syscall: NtCreateSection (74) args: [ (PHANDLE) 0xad8b1fb8a8, ([SECTION_ACCESS_MASK]) SECTION_MAP_READ, (POBJECT_ATTRIBUTES) 0x0, (PLARGE_INTEGER) 0x0, ([NtProtectionFlags]) PAGE_READONLY, ([SECTION_ALLOCATION]) SEC_COMMIT, (HANDLE) 0x128 ]"
}
```

What you'll see first is Hyperion opening a handle to the dll under the knownDlls OB directory.
Then, it queries various bits of information about the dll. After that, it opens a handle to the file itself and creates a section object backed by the file.
Next, you will notice that it creates a view of it with only read permissions and closes the handle to the section object. This is where things get interesting.
Then, they create 2 sections with RWX permissions, I should note that the number of sections and the size of them varies per each protected module and these sections are not file backed.
Once the sections are made, it creates views of the sections with RW permissions, and copies the original dll data to them.
Interestingly, it then un-maps the real module allocations that the Windows loader created, and maps the same sections again at the exact same addresses that the Windows loader allocated at, but this time without write permissions.

It should be noted that physical address they translate to is the same as the RW sections that were initially created.
Now that Hyperion has replaced all of the Windows loader allocations with its mapped sections, they allocate memory normally for the rest of the module, this part isn't protected.
Finally, they un-map the first file-backed read-only section, as well as the 2 writable sections that they used to write the original module's code. They then close all section handles.
Another important thing to note is that Hyperion mapped their views with the MEM_PHYSICAL flag, which prevents you from enabling CoW from usermode.

Some of the other modules they protect are: NtDLL, kernel32, kernelbase, and other Windows dlls.

Now that we've gotten past the imported module protection, we can discuss what else their initialization routine does.
It can now register an instrumentation callback, which from now on I will refer to as "IC," being an acronym.
Wondering what an IC is? It's an undocumented feature within Windows that allows you to capture transitions from kernel mode to user mode.
For example, before a syscall returns to the process, Windows checks if there is an IC registered for the process, if so, it sets rip to the address of the callback instead of the intended return address.
ICs allow you to log APCs, exceptions, and thread creations, making them very powerful.

Before I move on to the next section, it should be known that Hyperion hooks various functions within NtDLL, here are some:
![image](assets/images/Part%202/hyperion-hooked-functions.png)

Some of these hooked functions are also redirected in Hyperion's IC, meaning if you were to place a breakpoint on one of these functions, it will never trigger because the IC will manually redirect it to their hook directly.

#### Instrumentation Callback

Now that we know what an IC is, where they can be applied, and some elementary knowledge of their use in Hyperion, we can now go more in-depth.
In this section, I'll talk about the events their IC hooks, and the reason for hooking them.

Note that in the current version, their IC ignores all syscall events.

Below is the disassembly of their IC:
![image](assets/images/Part%202/Instrumental%20Callback%20(IC).png)

The function in the middle is responsible for setting the return address. Basically, the argument it receives is the original return address and it compares it against various target functions, if the addresses match up, then it returns their hook, otherwise it returns the argument passed.
Finally, the IC jumps into the address that was returned.
The functions that are currently targeted are: `KiUserExceptionDispatcher`, `LdrInitializeThunk`, and `KiUserApcDispatcher`.

Also, they set a flag in TEB, they do this to prevent recursion within the thread, if they didn't do this then if their hook triggered an event that is targeted, it'd infinitely recur. They prevent this by not redirecting in their IC if this flag is set (initialization of this flag can be seen below):

![image](assets/images/Part%202/IC%20prevent%20recursion.png)

Here's a decompilation of Hyperion's IC after fixing the control flow, this is not from the latest version, but everything is the same.
![image](assets/images/Part%202/IC-decompilation.png)

#### LdrInitializeThunk

In the previous section, we uncovered that one of the events Hyperion's IC redirects is LdrInitializeThunk.
So, what's so special about this function? Well, when the Windows kernel creates a new thread, it jumps to LdrInitializeThunk, it performs various datum initialization before jumping to the actual thread start address.

This means that Hyperion has full control over any new thread as it's created before its start address is ran, and can decide whether or not to let the thread continue, or to terminate it.
If you try to create a thread externally, you will realize that it never actually jumps to your thread's start address, this is because Hyperion does not let non-whitelisted threads be created.

You might be wondering "what determines if a thread is whitelisted" and that's a good question.
Earlier we saw that they hook some NtDLL imports, well among those is NtCreateThread(Ex).
This hook exists so that internally created threads by Roblox code or loaded modules will be whitelisted after a few checks, the hook creates a stub that decrypts the start address, then jumps to it. Then, they spoof the start address of the thread to the stub.
By the time the new thread spawns and hits the LdrInitializeThunk hook, Hyperion knows if it's legit or not.

Here's the disassembly of this stub, you can see it backs up registers, decrypts the real start address, then later it restores them and jumps to the start address:

![image](assets/images/Part%202/LdrInitializeThunk-disasm.png)

Before the stub returns, it pushes rax, which holds the decrypted start address, and when it hits the return it continues to the actual thread entry.

## Overview

In this part we started reversing the Hyperion binary, discovered the common obfuscation techniques that are employed, found out how they invoke syscalls, looked at the initialization routine, figured out how their protected modules work, uncovered instrumentation callbacks and how/why Hyperion uses them, analyzed the instrumentation callbacks and discovered which events it hooks, and shed light on how Hyperion prevents/detects external thread creation.

In the next part we will do a detailed analysis of Hyperion's LdrInitializeThunk hook, and figure out how we can successfully create threads externally.
Then, we will figure out how they monitor the working set of the process and also analyze how they encrypt/decrypt Roblox's code in realtime. I'll also explain how they detect various reverse engineering tools and how their hypervisor detections work.
Finally, we'll get started on writing a decryptor and injector and detail the process behind it.
