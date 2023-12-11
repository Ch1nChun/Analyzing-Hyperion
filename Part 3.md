# Analyzing Hyperion - Part 3: Threads, Memory and Encryption

## Introduction

> [!IMPORTANT]
> Attention: This is part 3 of the series, if you haven't already, make sure you read [Analyzing Hyperion - Part 2: Starting Out](Part%202.md) first.

In the last part we covered a lot of the core Hyperion components, in this part we will continue on analyzing the LdrInitializeThunk hook and cover some other concepts.
At the end we will also cover some of the Roblox level protections, that is the ones you will see once you decrypt Roblox and disassemble it.

## Credits

This thread was co-authored by the following users:

- [Dr McDingle](https://v3rmillion.net/member.php?action=profile&uid=618689)
- [AVX](https://v3rmillion.net/member.php?action=profile&uid=3134041)

## Table of Contents

- [Analyzing Hyperion - Part 3: Threads, Memory and Encryption](#analyzing-hyperion---part-3-threads-memory-and-encryption)
  - [Introduction](#introduction)
  - [Credits](#credits)
  - [Table of Contents](#table-of-contents)
  - [Analysis](#analysis)
    - [LdrInitializeThunk](#ldrinitializethunk)
    - [Memory Scans](#memory-scans)
    - [Encryption](#encryption)
    - [Stubbed Branches](#stubbed-branches)
    - [Import Protection](#import-protection)
  - [Overview](#overview)
  
## Analysis

### LdrInitializeThunk

In the last part we analyzed how they block external threads, now we will cover how their hook actually works.
To find the hook address you can either follow their hook jmp inside KiUserExceptionDispatcher or look at the IC, because they still hook the actual function.
A few instruction below where the jmp goes you will see a call instruction. This is the function that we are interested about.

Now there are many ways we can do to trace whats happening inside it, lets think about it for a moment. First based on logic we can assume they will access the start address of the thread that triggered the hook, based on that we can identify the main logic of the function quite easily. So what we will do is use a hypervisor to single step the function but instead doing it manually, we will only break on instructions that access the start address of our thread.

![LdrInitializeThunk Log](assets/images/Part%203/LdrInitializeThunkLog.png)

### Memory Scans

In the last part we also mentioned that Hyperion monitors all Roblox allocations. In short, they whitelist all their internal allocations, they also hook NtProtectVirtualMemory as well as NtAllocateVirtualMemory so they can whitelist memory allocated by internal modules.

Note they only really whitelist/care about executable allocations. Periodically Hyperion scans for all the memory in the Roblox process and if they find executable memory that isn't within their map of whitelisted pages they will remove it's permission and wait in hopes of thread execution there, which will raise exception and Hyperion will find your thread.

There are many ways to attack this, you could try to hide your allocations but that isn't easy considering their inline syscalls. You could also reverse the hooks I mentioned above and follow the actions they take when you attempt to allocate executable memory.
By any means those functions are massive and pretty hard to follow, but one thing you can notice is they all access a map and adds the allocation there if it passes a few checks. Looking at the NtProtectVirtualMemory hook, I noticed one of the function that it calls seemingly adds a value passed to it inside a map.

After breakpointing it, I noticed that the arg is actually the base of the executable page, the next arg is the size and the third arg is seemingly always 1.

The function RVA for the current version is RobloxPlayerBeta.dll + 0x20207D0 and the version of Roblox I'm currently analyzing is version-b9021bd8128442aa

![Whitelist map](assets/images/Part%203/Memory-Scans-WhitelistMap.png)

As you can see, this map contains list off every single whitelisted executable page inside the Roblox process. Once they do their scans, if the allocation they find isn't there, they will target it.
I tried to make executable allocate and add it to the list, and as expected Hyperion didn't change its protection after the scans ran.

### Encryption

In this section, we will delve into the encryption used by Hyperion specifically on Roblox code. As mentioned in part 1, the entire .text section of Roblox is encrypted and only decrypted when necessary. Hyperion accomplishes this by setting the entire encrypted region as NO_ACCESS. Upon execution, this triggers a no access exception, which then invokes their exception routine.
From there, they check the exception type and exception address to determine the appropriate action to take. If the exception occurs within Roblox code, they perform a few checks and initiate the decryption procedure. However, Hyperion has strategically placed traps in certain pages within Roblox that are not actual code but designed to act as traps.
If the execution starts in one of these protected pages, it results in a ban. This tactic prevents someone from decrypting all pages successively.

To avoid race conditions, Roblox employs two separate views of memory. The first view, the executable one, is read-only and contains the encrypted code. The second view is writable, allowing them to write and decrypt the pages without altering the initial decryption and effectively avoiding race conditions.

Moving on, let's analyze their exception hook using the same method applied to the LdrInitializeThunk hook. They must access the exception RIP, so we can log all instructions that access it. This will reveal various compares with hardcoded values, as shown in the picture below:

![Disassembly of the encryption exception hook](assets/images/Part%203/Encryption-exception-hook-disasm.png)

In the image above, you can see that r12 holds the page offset inside the Roblox module, and it is compared against some values. These values are actually the page offsets of the trap pages within Roblox ((Address - Base) / PAGE_SIZE). When decrypting, it is crucial to avoid triggering execution on those specific locations.

Now that we understand this, let's consider how we will begin decryption. The most convenient approach is to do it dynamically. Hyperion decrypts the page when execution occurs. They also decrypt pages if someone attempts to read from them, but they restrict using the same reading location on too many pages, making this approach less ideal.

One possible solution is to spawn a thread on every encrypted page inside Roblox, which will force Hyperion to decrypt it. However, this presents a new issue. When the thread continues execution after Hyperion decrypts the code, it will hit unknown code, leading to a crash.

In the past, Hyperion used NtContinue to continue from the decryption, resuming the thread with the provided context. However, to prevent hooking of NtContinue, they switched to using iret and set the context themselves.
The iret instruction remains in the initial dispatcher, making it vulnerable and easy to patch. Similar to the LdrInitializeThunk hook, you can replace it with a jump and control the context used to continue the execution of the thread.

To avoid any crashes, you can simply set the RIP to a return instruction, which will direct the execution back to the CALL instruction that originally executed the code.

The code is executed by Roblox multiple times, used for each decryption and exception. To avoid returning to actual code and risking a crash, one can set RCX to a magic value. Any register can be used for this purpose, but RCX is a convenient choice as it can easily be passed as an argument to the CreateRemoteThread function.

Here's an example of how this can be achieved:

```cpp
if (Context->Rcx == MAGIC_KEY) {
 //
 // This simulates a return instruction and sets the RIP to the return address, which is located on the stack for us.
 // As mentioned above, we can simply pass a MAGIC_KEY as the argument to CreateRemoteThread, which will be in RCX for us.
 //
 Context->Rip = *(uintptr_t*)Context->Rsp;
 Context->Rsp += 8;
}
```

With this approach, we should have sufficient understanding of the encryption process. In the next part, we will begin writing a decryptor using the methods discussed above. Subsequent sections will delve into some of the 'Roblox level' protections, such as the stubbed branches scattered throughout the code and the imports.

### Stubbed Branches

If you have ever disassembled Roblox, you have noticed the INT3 breakpoints scattered throughout its binary code.
![stubbed branches](assets/images/Part%203/stubbed-branches-int3-breakpoints.png)

Those are always located where the first branching instruction was. This essentially means the first branch of every function is replaced by the INT3s.

Before analyzing how this works, lets first delve into how we stumbled upon this discovery. While we were making our binary re-builder, we can across the INT3's.

Initially, we thought they are replaced call instructions, but to confirm that, we found the same function in Roblox Studio and determined that they, in fact, constituted branching instructions.
With this newfound knowledge, we set a breakpoint on their exception hook and logged every access to the RFlags member in the CONTEXT structure. This approach was motivated by the realization that, in order to effectively emulate these instructions, Hyperion would need to access the flags to figure the appropriate branch to follow.

![image](assets/images/Part%203/stubbed-branches-int3-handler.png)

Each variation of the emulation handlers was associated with a corresponding entry in a jump table, which is used to invoke the desired handler.

![image](assets/images/Part%203/stubbed-branches-int3-handler.png)

During my observance of accesses to the CONTEXT structure, I detected an instruction responsible for writing to RIP after the emulation process had finished.

![image](assets/images/Part%203/stubbed-branches-int3-handler-RIP-writer.png)

After tracking back, I noticed that Hyperion was reading from an massive array in the .data section. This array contains structure used for describing what each int3 stub replaces, it contains the RVA of the stub, the type of the branch, and even the real instructions that were replaced. This is done so Hyperion can restore them on functions that are called often.

![image](assets/images/Part%203/stubbed-branches-data-array.png)

In the image above, the first 4 bytes are the type, the next byte is the size of the real instruction replaced, followed by the actual instruction, and at the end, the RVA to the specific stub is stored.

So in short, Hyperion emulates the real branch instruction every time the INT3's are hit, if you want to restore them with the real instructions, you can iterate the array and get the RVA of each entry, and write the original instruction on that place, which as I said is also stored in the array

### Import Protection

Just like before, Hyperion also protects the binary through their dynamically protected imports.

To kickstart the process, I set a breakpoint on a commonly used import and waited until Roblox triggered it.
Upon further examination, I noticed that the caller was dereferencing the key in a readonly section (.rdata) and jumping to it.

To debug the code, I simply set a conditional breakpoint on their hook and traced if the ExceptionCode was an access violation. Jumping to the key is an invalid address and triggers a page fault.
Once the import exception handler was hit, Hyperion first checks if the exception being ACCESS_VIOLATION is caused by a protected import. If so, it continues through the handler logic. The handler will first decrypt the 'key', which will then become an index inside a specific handler. As mentioned earlier, handlers have a hardcoded list for each possible module that Roblox imports something from.

![image](assets/images/Part%203/import-protection.png)

Each handler has a hash lookup table containing the hash for a specific import. When the correct handler is identified, the RVA to the hash list in Roblox is stored in the structure. The hash algo used is Fnv1a-32. After the import key is decrypted, the code scans through each handler and checks the imports it handles. For example, if the NtDLL handler handles keys from 0 to 20 and your key is 10, then the NtDLL handler will be triggered, and 10 will be used as an index in the hash lookup table. Similarly, if the kernelbase handler handles keys from 50 to 60 and your key is 55, then lookup index 5 in the handler-specific lookup table will be used. After that, they now have the Fnv1a hash, and they will call the GetExport function with it, which will return the specific export once it's found.

![import protection lookup](assets/images/Part%203/import-protection-lookup.png)

If the module isn't loaded (Roblox uses imports from some modules that aren't loaded when the process loads), then Hyperion will manually load it and try using both flags on LoadLibraryExA.
After locating the import, Hyperion then recursively follows every JMP/CALL to locate the lowest point in the chain.
Once this process is completed, they set the RIP to the respective import and return back to their exception dispatcher.

## Overview

In this part we figured out a way to make external threads, analyzed how Hyperion scans for executable memory, figured out how they encrypt the Roblox code and a way to decrypt it and looked at some of the Roblox level protections.

In the last 3 parts we have covered all the basics and figured out how to successfully make threads, decrypt Roblox and prevent our allocations from being found. In the next part we will analyze how Hyperion detects malicious processes and also how they detect hypervisors. And finally we will use all this knowledge to write a example decryptor and injector.
