# Analyzing Hyperion Part 6: Working Set, RPM detection, and system scans

## Introduction

> [!IMPORTANT]
> The builds analyzed in this article are: version-70a2467227df4077
> for the WS analysis and version-170fa59ac12943d9 for the rest.
> Another small thing I want to mention is that the recent build of Roblox,
> '170fa59ac12943d9',
> renamed the .krampus section to .acedia.
> One can only guess why this was done,
> but probably for the memes,
> considering a P2C named Krampus was recently released.
>
> Hello everyone! .krampus here.
> Recently, Hyperion added a few interesting routines that
> I feel are worth discussing.

As you probably know by now, Hyperion has added a new check which most
people refer to as 'Working Set' check.
This means that people reading unsynchronized from the memory associate
with the engine will get detected.
But not many actually know how this is implemented.
In this article,
we will briefly discuss this alongside a few more interesting topics.

## Credits

- [gogo1000](https://v3rm.net/members/gogo1000.290/)
    [contributed section](#the-new-system-scan-routine-gogo1000)
- 0AVX (can't find)
    [contributed section](#working-set-detections-0avx)
- [0xNemi](https://v3rm.net/members/0xnemi.107/) (fake?)

## Working Set Detections (0AVX)

External cheat providers,
such as ESP/Aimbot,
often access the same Roblox instances,
such as Players, Humanoid, etc.
Therefore, Roblox has taken action against them.

To accomplish this, they identify unauthorized memory access instances.
How?
It's quite straightforward.
Since they execute within the task scheduler worker loop,
they control when Roblox's code is executed.
So, they invalidate particular instances,
and before Roblox's code is executed,
they verify whether these instances have been accessed.

This function is located at: 0x289B710

Let's dive into how this works. First things first,
they acquire the initial function by calling into Hyperion.
How do they call into Hyperion?
They call an address that is invalid and causes an exception,
where Hyperion will catch it, switch on the index,
and execute the respective handler.

```c
sub_289BDC0(&v75) ->
    //Returns a function table.
    return MEMORY[0xFFFFFFFFFFFFFFF7](*(_QWORD *)a1);
```

```c
if ( !HYPERION_DETECTION_TRIPPED && *((_QWORD *)&v71 + 1) )
      HYPERION_DETECTION_TRIPPED = (*((unsigned int (__fastcall **)(_QWORD, __int64, __int64, __int64))&v71 + 1))(
                                     0i64,
                                     5000i64,
                                     15000i64,
                                     1i64) == 7;
```

This function's job is to see if memory has been accessed.
It does this by checking
if the pages supporting the Instances in Roblox
are part of the process's working set using NtQueryVirtualMemory.

Before we proceed,
let's delve into how Windows handles memory,
according to Microsoft's documentation:
"The working set of a process
is the set of pages in the virtual address space
of the process that are currently resident in physical memory.
The working set contains only pageable memory allocations;
non-pageable memory allocations such as Address Windowing Extensions
(AWE) or large page allocations are not included in the working set."

This might sound a bit complex,
so let's break it down.
In any modern operating system, like Windows, memory management
involves two main states: physical memory and paged out memory.
If a chunk of memory isn't frequently used,
it's not efficient to keep it constantly loaded into physical memory.
So, the OS moves it to the page file.
When this memory is accessed again,
it triggers a page fault,
and Windows then fetches it
from the page file back into physical memory
and adds it to the working set.

Hyperion takes advantage of this by extracting memory from the working
set and subsequently checking if it's still there
before the next job iteration.
Interestingly, in the Microsoft documentation,
there's a function that perfectly fits this scenario: VirtualUnlock:
"Calling VirtualUnlock on a range of memory
that isn't locked releases the pages from the process's working set."

Now
that we understand how they're verifying if memory has been accessed,
we can proceed to where they actually invalidate the Instances:

```c
HYPERION_DETECTION_TRIPPED = ((unsigned int (__fastcall *)(_QWORD))v72)(0i64) == 7;
```

If either of these functions returns 7, indicating that the detection is triggered, they set a flag external to the task scheduler loop.
Later on, they then examine whether this flag is set, and if it is, they refrain from calling into Hyperion and instead execute an algorithm that utilizes the TSC to determine whether a crash should occur.
This tactic is employed to confuse individuals since an immediate crash doesn't ensue:

```c
if ( HYPERION_DETECTION_TRIPPED )
{
  v24 = __rdtsc();
  v17 = v24 / 0x2710;
  v18 = v24 % 0x2710;
  if ( v18 < 0x64 )
  {
    // Causes a crash
    sub_5C0960(); -> return MEMORY[0xFFFFFFFFFFFFFFF0]();
    v22 = v19;
    goto LABEL_36;
  }
LABEL_33:
  if ( HYPERION_DETECTION_TRIPPED )
    goto LABEL_36;
}
```

Now, each Instance by Roblox is allocated by the Instance allocator and deallocated with the Instance deallocator, which is called the Deleter.
When Instances are watched and allocated by Hyperion, they also need to be deleted by Hyperion so it can remove them from the cache and not get checked, and their custom deallocator is known as "Deleter2".
You can find all watched Instances by searching for "Deleter2":

Additionally, you could locate all Instance (de)allocation
functions by scanning through Roblox's data section
and identifying pointers that reference Hyperion's module.
Here's an example where they invoke Hyperion
to allocate memory and set the custom deallocator:

```c
HYPERION_ALLOCATE_WACHED_MEMORY = (unsigned int (__fastcall *)(__int64 *, __int64))qword_4B85B78;
if ( !qword_4B85B78 )
{
  ((void (__fastcall *)(__int64))HYPERION_ACQUIRE_PTR)(v5);
  HYPERION_ALLOCATE_WACHED_MEMORY = (unsigned int (__fastcall *)(__int64 *, __int64))qword_4B85B78;
}
v20 = 0i64;
if ( HYPERION_ALLOCATE_WACHED_MEMORY && !HYPERION_ALLOCATE_WACHED_MEMORY(&v20, 0x880i64) )
{
  v8 = sub_196FB90(v20);
  v29 = v28;
  v21[0] = v8;
  v21[1] = (__int64)&v29;
  v22 = 1;
  v9 = sub_293E2E0(0x18ui64);
  v10 = v9;

  *(_DWORD *)(v9 + 8) = 1;
  *(_DWORD *)(v9 + 12) = 1;
  *(_QWORD *)v9 = &std::_Ref_count_resource<RBX::Players *,RBX::Creatable<RBX::Instance>::Deleter2<RBX::Players>>::vftable;
  *(_QWORD *)(v9 + 16) = v8;
  ...
}
```

## The New System Scan Routine (gogo1000)

Recently,
about two weeks ago from the time this article was posted,
Hyperion added a pretty non-standard check.
This check was targeting one of the biggest P2Cs
on the market at the moment.
This check is run from the same routine
that performs system-related scans,
like for illegal processes and drivers.
Internally, it is pretty straightforward;
it's all in a single function.
The way it starts is by constructing an encrypted string
on the stack and decrypting it.

The function RVA in the analyzed build is `0x20C3410`.

![missing image]

This is the assembly code constructing the string.
After putting a breakpoint at the end of it and checking the stack,
the string constructed was '\Device\CRIPXNIGGA'.
This is where the interesting part begins.

After the decrypted string is written to the stack,
the routine proceeds to go to 0x20C9170.
This is where a call to an interesting function is invoked.
The said function is actually a manual syscall stub made by Hyperion,
which calls NtOpenSection.

![missing image]

The arguments passed are essentially trying to open a section handle
to this device object called 'CRIPXNIGGA'.
This is where someone could get confused;
This is a device object,
not a section object. How is this supposed to work?
Well, the secret here is that Hyperion
doesn't really want to open a handle;
they just want to check if this object exists.
If you go a little below the call site,
you can see Hyperion checking
if the result isn't 0 and then comparing it against 0C0000024h.
If we open up the Microsoft documentation,
we can see that this means 'STATUS_OBJECT_TYPE_MISMATCH'.
This essentially tells Hyperion that the object exists
but fails to create a handle, which was exactly their goal.

![missing image]

At this point, if this status code matches,
they set R15 to 0x10 and continue.
This is actually done 2 more times, but with different strings.
Those strings are: '\Driver\NEEEEGRO' and '\Device\test121'.
Now we know that this routine simply checks
if those 2 device objects or the driver object exist,
and if so, they perform quite an interesting action.

![missing image]

At the end of the routine,
R15 is compared,
and if it was set previously,
the routine doesn't return; instead,
they proceed to create a report which will later be sent to the server.
This networking-related part is not important in this article, however.
This is enough to understand what this routine is performing.
Here are the other locations inside
this routine that construct the other strings and perform the check.

---

- `0x20C98C4`: **constructs the 2nd string**
- `0x20CF424`: **call to NpenSection**
- `0x20CFB00`: **constructs the 3rd string**
- `0x20D55B9`: **call to NtOpenSection**
