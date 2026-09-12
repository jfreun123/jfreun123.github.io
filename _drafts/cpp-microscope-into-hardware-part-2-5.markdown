---
layout: post
title:  "C++ as a Microscope Into Hardware, Part 2.5 (Virtual Memory)"
date:   TODO
categories: cpp
series: "C++ as a Microscope Into Hardware"
part: 2.5
---


*The following was written by me, but reviewed by Claude for grammar and general mistakes.*

*This is Part 2.5 of a series on Linus Boehm's C++Now 2025 talk, [C++ as a Microscope Into Hardware](https://www.youtube.com/watch?v=KFe6LCcDjL8).*

Let's list out what we know from [part 2](/cpp/2026/08/22/cpp-microscope-into-hardware-part-2.html):

- the memory we see in our program is virtual:  two programs can have the same virtual memory addresses that map to different physical memory addresses
- Given we fail after using 131071 GiB of virtual memory, we'd expect the translation to somehow use *at least* 47 bits of address.  Addresses name bytes (not GiB), so:

  ```
  1 GiB = 1024^3 bytes = 2^30 bytes

  ceil(log2(131071 * 2^30)) = ceil(log2(131071) + log2(2^30))
                            = ceil(log2(131071)) + 30
                            = 17 + 30            (since 2^17 = 131072)
                            = 47 bits
  ```
- The first time reading/writing to memory takes substantially longer than the second time.
- bytes per fault: 4095 bytes/fault (though we currently do not know what this means)

Given what we observed in the previous post, any model of how memory works must explain this behavior.  But which model is a good starting point?  For that, we will use the highly academic, battle-tested approach of "taking an educated guess."  

## Model #1:  Base and Bounds

Let's take two hardware registers within the CPU:  one for the base and one for the bounds.  Here, each program assumes it starts at address zero but, when running, the hardware translates the virtual address using the following formula:

```
physical address = virtual address + base
```

Note that there is one base and bounds pair per process:  the OS saves and restores each process's pair on a context switch.

Here, the bounds register just provides protection, checking that we do not go out of bounds.

While this approach is simple, there is one huge flaw:  internal fragmentation.  That is, the space between the stack and heap is wasted in a huge amount of internal fragmentation — internal as the wasted space is inside the allocated unit.  Most programs are small (MBs) and this wasted space can waste GBs:

```
address space (base to bounds):    4 GiB // TODO:  why 4 GiB
code + heap (bottom):            ~ 4 MiB // TODO:  why 4 MiB
stack (top):                     ~ 4 MiB

wasted gap in the middle:          4 GiB - 8 MiB ≈ 4088 MiB   (~99.8% of the allocation)
```

Moreover, for large programs, it becomes challenging to run a program when the entire address space does not easily fit into memory.  We need a new model.

## Model #2:  Segmentation  

It was nice how simple Base and Bounds was.  Let's try to keep it.  To patch it up, instead of just having one base and bounds pair in our MMU (memory management unit), let's have a base and bounds pair per logical segment of the address space — logical meaning the natural divisions of the address space we already know:  code, heap, and stack ([OSTEP chapter 16](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf)).  A segment is just a division of our usual address space (one segment each for code, stack, and heap); giving each segment its own base and bounds allows the OS to place each segment independently in physical memory and thus avoid filling physical memory with unused virtual address space, as happened when we mandated that code, stack, and heap be placed in one single unit.  As before, this state is per process:  each process gets its own set of (segment, base, bounds) tuples, saved and restored on a context switch.

For an example of what this translation could look like:
```
// get top 2 bits of 14-bit VA
Segment = (VirtualAddress & SEG_MASK) >> SEG_SHIFT
// now get offset
Offset  = VirtualAddress & OFFSET_MASK
if (Offset >= Bounds[Segment])
    RaiseException(PROTECTION_FAULT)
else
    PhysAddr = Base[Segment] + Offset
    Register = AccessMemory(PhysAddr)
```

*(Pseudocode from [OSTEP chapter 16](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf), Figure 16.2.)*

So, if we have n segments chopping up our previous base and bounds block, we will now be n times smaller!  Huge improvement, as we can control n.  Moreover, now that code, stack, and heap are ripped apart, it becomes trivial to share:  for instance, we can introduce a protection bit that marks the code segment read-only and trivially share code across processes.

However, there are still some issues with this model:

1.  Context switching becomes a bit more awkward:  we need to save more bases and bounds instead of just one.
2.  Malloc may grow the heap.  The OS may end having to copy the entire heap to a bigger free hole somewhere else.  This can also lead to immense external fragmentation.
4.  Segmentation is still not fully flexible enough to support a generalized, sparse address space.  For instance, if we make a `std::vector<int> a(n)` for some huge `n` and only access `a[0]` and `a[n-1]`, then we still need a large heap.  In other words, if we have a large but sparsely-used heap all in one logical segment, the entire heap must still reside in memory in order to be accessed.

So, we need yet another model.  For our final guess...

## Model #3:  Paging


## Resources:

- [Computer Systems: A Programmer's Perspective (CS:APP), 3rd Edition, Randal E. Bryant and David R. O'Hallaron](https://csapp.cs.cmu.edu/): the classic systems textbook from the programmer's point of view.  Chapter 9 (Virtual Memory) covers address translation, TLBs, and multi-level page tables end to end.
- [Operating Systems: Three Easy Pieces (OSTEP), Remzi and Andrea Arpaci-Dusseau](https://pages.cs.wisc.edu/~remzi/OSTEP/): free textbook whose virtual memory chapters cover everything this post touches.  For this post specifically, read chapters [13 (Address Spaces)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-intro.pdf), [15 (Address Translation)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf), [16 (Segmentation)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf), [18 (Introduction to Paging)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-paging.pdf), [19 (Translation Lookaside Buffers)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-tlbs.pdf), [20 (Advanced Page Tables)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-smalltables.pdf), and [21 (Swapping: Mechanisms)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys.pdf).