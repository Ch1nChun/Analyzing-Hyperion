# Analyzing Hyperion - Part 6: Working Set, RPM Detection, and System Scans

> [!IMPORTANT]
> Before I get started,
> I want to mention that
> the version of Roblox analyzed in this thread is:
> **0.607.0.41238** | **version-88cfc23f4e7d4e4b**

## Introduction

Some time ago, I started writing posts about Byfron.
I split them into parts from 1 to 4. Link to them here.
Today, I'm adding another part,
focusing on something
new they added after my last post – a check for module integrity.
You can consider this part 5.

1 of Hyperion's goals is to prevent unknown code
from residing in the process address space.
To achieve this,
they ensure that legitimately loaded modules
remain in an unpatched state. Of course,
they cannot simply flag users for having a single patch,
as there are legitimate reasons for modules to have patches.
Due to this, Hyperion's goal isn't to ensure
that every module is completely untouched. Instead,
they set a limit on the number of patches
a module can have before it becomes a problem.

## Getting Started

Let's start with the first goal,
which is to identify the routine that performs those checks.
A quick experiment
you can do is load some unused module and override a big chunk of it.
After a little more than 30 seconds
the process will crash.
To trace how this check is performed,
I placed a trap on one of the pages of a legitimate module and waited
to see if Hyperion will read anything from it. Surprisingly enough,
the trap was never hit, even after waiting a couple of minutes. This
made me think that maybe the check is run only if another condition is
met.

After thinking for a little bit,
I realized that they might be only running
it if a module is patched at least once.
You might be thinking,
how can they know if a module has been patched without ever probing it.
Well, Windows uses a technique called CoW (copy on write),
which means that by default,
every module loaded will point to a shared physical page until a
process writes to it, which will make the kernel copy it into the
process private working set.
Windows also exposes some undocumented APIs to get working set
information, which Hyperion uses to see if any page within a module
has been copied, showing that it's being written to. Before we dive
into how this works, let's write to some page within a module and
check if the trap will be hit.

![missing image]

As expected, we got a read.
Now we can begin the interesting part,
which is to find out how exactly this check is performed.

![missing image]

The highlighted instruction is the one that triggered the trap,
and the function also seems pretty simple.
By the looks of it,
all it does is check if a specific byte matches the original module,
which is acquired from disk, but we will look into this later.
It then increments some value that is probably the amount of patched
bytes within the module.
Tracing the return address gives us a function
whose return value was 1,
which is because I only patched a single byte of the module.

![missing image]

I then tried to patch multiple bytes,
and as expected, I got the correct patch count.

### The Process

For now,
it seems like the routine is pretty straightforward.
To advance things a little, let's find the beginning of the routine
and trace the logic step by step. To do this,
I found the calling thread ID from my breakpoint, and I proceeded
to check its stack to find a common point we can start from.

![missing image]

The only point within Hyperion seems to be waiting for some object.
Going there in the disassembler shows us
that the thread simply waits for 48000ms
before re-doing the check.

![missing image]
Having found the starting point of the routine,
we can run a trace
and look for anything interesting
that will give us insight into how the check is performed.
After a brief look at the trace,
the first interesting call I saw is a call to NtQueryVirtualMemory,
which simply finds
every allocation in the proces
and checks its allocation type for MEM_IMAGE.
If it matches, it then proceeds forward;
otherwise,
it simply goes to the next allocation.
`RobloxPlayerBeta.dll + 0x13AA5E0` represents the RVA
of the call instruction invoking the function,
and shortly below is where the check is performed:

```asm
cmp [rbp+00001A08], 0x1000000
```

After any image has been found, the function proceeds to call
NtQueryVirtualMemory again, but this time with class ID 03.
This call is performed at `RobloxPlayerBeta.dll + 0x13AE00B`.

Looking at unofficial documentation of this class ID,
we can see it's called MemoryRegionInformation.
Hyperion simply uses that to query info about
the underlying image region.
After storing the base and the size,
they proceed to yet another NtQueryVirtualMemory calling site,
located at `RobloxPlayerBeta.dll + 0x13C5FF4`.

This is where the interesting part begins.
It initiates a loop that iterates through every memory region of the
image, checking the working set information for the `shared` flag.
If the bit corresponding to this flag is 0,
they know the page has been modified
and proceed to run a scan to find the patch count.
At the beginning of each memory region,
Hyperion queries the working set with a class id of 4,
which stands for MemoryWorkingSetExInformation
and corresponds to the
`struct MEMORY_WORKING_SET_EX_INFORMATION`

This call is located at `RobloxPlayerBeta.dll + 0x13B1E65`.

[!missing image]

A little below the call site,
we can see Hyperion iterating through the regions and comparing the
flags by the AND'ed result of 8001.
This bitmask represents bit 0 and bit 15,
corresponding to the flags `valid` and `shared` of the structure
discussed previously.
If the valid bit is toggled but the shared bit is not,
they would increment
a counter representing the number of patched pages,
which is later used to decide if the module should be compared or not.
If none of the checked sections are copied into the private working
set of the process,
showing CoW isn't tripped,
they skip the image and proceed to scan the next one.

The next few sections aren't all that interesting,
so I'll simply put their RVA along with a small description.
If you are interested in them, feel free to look deeper.

- `RobloxPlayerBeta.dll + 0x13BAAB2`: opens
    a file handle to the corresponding file backing up the image.

- `RobloxPlayerBeta.dll + 0x13BE967` maps a section
    of the file in memory so they can read it.

After mapping the file, they run basic validity checks and proceed
to call the function we saw in the beginning,
which compares the module in memory to the one from disk,
finding the amount of patches.
This process continues for every read-only section
of the image being scanned,
then the total number of patches is compared against a constant number,
deciding if it exceeds the patch limit.

![missing image]

As we can see,
the patch count is stored at `RBP + 0x11E8`,
which is compared against `0`. If there are any patches,
it is compared again with `0x80001`,
allowing a total of `0x80000` patches before triggering a crash.
If the patch count exceeds this number,
they send telemetry data to Roblox for statistics
and trigger a crash procedure.
Otherwise,
they go all the way back to the initial loop responsible for scanning
memory for images and repeat the process for the next image found.

## Conclusion

In conclusion,
the scan process
is simple while being executed in a relatively smart way,
avoiding the need to scan unpatched modules to keep performance up.
As a side effect, this could easily lead to the misconception
that they don't have any module integrity verification.
This could potentially open a few vectors for hiding patches,
but that's an exercise for the reader to explore.
