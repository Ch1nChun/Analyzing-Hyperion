# Analyzing Hyperion - Part 4: Hypervisors and process scans

## Introduction

> [!IMPORTANT]
> Attention: This is part 4 of the series, if you haven't already, make sure you read [Analyzing Hyperion - Part 3: Threads, Memory and Encryption](Part%203.md) first.

In the previous 3 parts we have covered nearly all important aspects of Hyperion, now we are gonna look at some of the things they do to make reversing more difficult. That is the Hypervisor detections and reversing tools.

## Table of Contents

- [Analyzing Hyperion - Part 4: Hypervisors and process scans](#analyzing-hyperion---part-4-hypervisors-and-process-scans)
  - [Introduction](#introduction)
  - [Table of Contents](#table-of-contents)
  - [Hypervisors](#hypervisors)
    - [EIP overflow](#eip-overflow)
    - [Trap flag](#trap-flag)
    - [#UD](#ud)
  - [Process Scans](#process-scans)
  - [Overview](#overview)

## Hypervisors

### EIP overflow

Using hypervisor for analysis can be very powerful if applied correctly, it gives you full control of your system allowing you to intercept anything. Because of that Hyperion uses various method to detect hypervisors. However, unlike kernel level AntiCheats, Hyperion have limited things that they can do from usermode. Even so, they seems to be sufficient in stopping most people.
In general if you have ever written hypervisor from scratch and have followed the SDM correctly then you have nothing to worry about. Sadly most open source hypervisors out there have many emulation mistakes which makes them easily detectable.

Anyone who have written a hypervisor in the past will most likely find their methods within less than a hour. Hyperion uses methods to detect specific hypervisors as well as generic ways. To start analyzing them, I simply log various information for each VMExit Roblox caused. Now you must remember that I have disabled various VMCs controls to limit the exits I get so I can focus on the basics first.

After printing the exit IP as well as some generic information like the CPU mode and the flags the first thing that caught my eye is an exit from 0xFFFFFFFE with reason CPUId.

```json
{
    "Process": "RobloxPlayerBeta.exe",
    "Message":
      "Exit from: 0xFFFFFFFE | Reason: CPUId | RAX: FFFD0033 | RCX: FFFD0000 | Long mode: 0 | Trap flag: 0"
}
```

At first you might be wondering what is so special about that.
Well if you look at the CPU mode,
you can see it's not in long mode anymore.
Now you probably know that 32 bit programs
are executed in compatibility mode on 64 bit systems.
Hyperion takes advantage of this by putting the CPU
in 32 bit mode
and executing CPUId there, which causes
an unconditional VMExit.

The CPUId instruction is 2 bytes long
so in normal operation the EIP will be incremented by 2,
but this will cause it to overflow.
Now many open source hypervisors
like to assume that they are always running
in long mode
and use 64 bit instruction pointer internally.
Which means the behavior of incrementing by 2 will
be different if a hypervisor like that is loaded
than on bare metal.

In any case, an exception will be caused,
for example in normal operation,
the CPU will try fetching the next instruction
at address 0
and get a page fault,
then the OS will invoke the exception handlers with code ACCESS_VIOLATION.

Interestingly enough,
Hyperion doesn't use their IC to handle this,
instead they register a VEH.
In you are wondering how I figured this out,
I simply followed the strategy used in previous parts.
That is to simply use my hypervisor to log accesses
on RIP and
current code segment within the exception record.
After this I got few logs
within the same function and upon looking at it in IDA,
I immediately knew what was going on.

After fixing the control flow and decompiling it, this is what I saw
![image](assets/images/Part%204/Hypervisors-EIP-overflow-decompiled.png)

They are simply checking if the current CS is 32 bit one, if the exception address is 0 and if the reason is ACCESS_VIOLATION. After this, they simply handle the exception and continue. Now you might ask yourself continue where?

![image](assets/images/Part%204/Hypervisors-EIP-overflow-exception-handler-where.png)

This is the function that switches to compatibility mode and jumps to 0xFFFFFFFE.
Basically if the exception happened correctly, they simply set RIP to next instruction after the call, which in this case jumps to the end of the function, which returns normally.

Now fixing this is very simple, just check the current CPU mode and increment the instruction pointer appropriately.

### Trap flag

Now the other generic check is nothing special or unique.
In one of the exits,
I noticed the trap flag being set.
Now if you have followed the manual correctly,
you will know that you are responsible for injecting debug exceptions if the trap flag is set
There are a bunch of side cases with that as well but after looking at the function that caused the exit:

![image](assets/images/Part%204/Hypervisors-trap-flag-setter.png)

You can clearly see that they are setting the trap flag, setting ss which changes interruptability state and finally causing unconditional VMExit.
If you don't handle those correctly, the behavior will change from the one on bare metal.

Here's what the manual says about blocking by ss:
> Execution of a **MOV** to **SS**
> or a **POP** to **SS**
> blocks or interrupts for one instruction after its
> execution.
> In addition,
> certain debug exceptions are inhibited between a **MOV**
> to **SS**
> or a **POP** to **SS** and a subsequent instruction.

So in your VMExit handler,
you need to reset the state
and inject the debug exception,
which in this case will be skipped,
but if you don't reset the interruptability state,
2 instructions will be skipped.

### #UD

Another thing I don't think is worth getting into detail, but is worth mentioning, is that they will also cause various #UD exceptions.

Now some hypervisors disable syscall/ret extension which makes them cause #UD. They then capture the exception in the hypervisor to log syscalls. However some of them assume that all #UDs coming from usermode are syscalls and emulate it. But in Hyperion's case, the cause of the #UD is not real syscall. To fix this you should simply read the instruction and verify it's a syscall, if not simply inject #UD back to guest.

## Process Scans

Now as mentioned before, they also scan running processes for specific reversing tools. Some example of those are:
**Cheat Engine**, **ReClass.NET**, and **IDA**

This makes reversing and testing things much more difficult specially if you rely on those tools. Luckily enough its pretty easy to go around the scans. Hyperion has dedicated thread that runs every second, scanning for those tools.

After finding the thread and isolating syscalls from it, you can see the sequence they are taking. First they query `\\Device\` directory by using NtOpenDirectoryObject and NtQueryDirectoryObject to scan for specific devices, then they also query the `BaseNamedObjects` directory of the current session. This one have interesting reason for. You might not know but IDA actually makes a few named objects.

![image](assets/images/Part%204/Process-scans-IDA-base-named-objects.png)

The reason for those scans is exactly that, they search for them and if they find them, they will crash the client.

The next syscalls invoked are querying `\\REGISTRY\\MACHINE\`,
`KeyValueBasicInformation`.
Getting `SystemPoolTagInformation` and `SystemBigPoolInformation` from `NtQuerySystemInformation`,
as well as `SystemExtendedProcessInformation`.
After that they start firing `NtUserFindWindowEx`
to find every window on your system.
After they find all windows,
they get the owning process and
try to open handle to it with `PROCESS_QUERY_LIMITED_INFORMATION`,
if the handle is successfully opened,
they use `NtQueryInformationProcess` with ProcessImageInformation flag and close it.

Then they wait for a second and repeat the steps.
It's important to note
they also do a different scans every once in a while,
one of which gets all running threads as well.

They also scan for open handles,
so you should avoid having a `FULL_ACCESS` handle opened to Roblox,
especially from unsigned process.

In the end,
if you don't want to bother much,
you can simply hook `NtOpenProcess` from the kernel
and deny access to
whatever program you are trying to hide.
For IDA,
you must also hook `NtQueryDirectoryObject`
and check if they are querying the IDA object,
if so, skip it.

## Overview

In this part,
we learned how Hyperion:
detects hypervisors
and scans for specific processes.

Now, as I have covered
the most **important** concepts of Hyperion.
In the next part,
I will discuss approaches
in decrypting and dumping Roblox,
and in the end, write a working decryptor.
