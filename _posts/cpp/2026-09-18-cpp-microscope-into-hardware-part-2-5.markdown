---
layout: post
title:  "C++ as a Microscope Into Hardware, Part 2.5 (Virtual Memory)"
date:   2026-09-18 00:00:00 -0400
categories: cpp
series: "C++ as a Microscope Into Hardware"
part: 2.5
---

*WORK IN PROGRESS*

*The following was written by me, but reviewed by Claude for grammar and general mistakes.*

*This is Part 2.5 of a series on Linus Boehm's C++Now 2025 talk, [C++ as a Microscope Into Hardware](https://www.youtube.com/watch?v=KFe6LCcDjL8).*

Let's list out what we know from [part 2](/cpp/2026/08/22/cpp-microscope-into-hardware-part-2.html):

- The memory we see in our program is virtual:  two programs can have the same virtual memory addresses that map to different physical memory addresses.
- Given we fail after using 131071 GiB of virtual memory, we'd expect the translation to somehow use *at least* 47 bits of address.  Addresses name bytes (not GiB), so:

  ```
  1 GiB = 1024^3 bytes = 2^30 bytes

  ceil(log2(131071 * 2^30)) = ceil(log2(131071) + log2(2^30))
                            = ceil(log2(131071)) + 30
                            = 17 + 30            (since 2^17 = 131072)
                            = 47 bits
  ```
- The first time reading/writing to memory takes substantially longer than the second time.
- Bytes per fault:  4095 bytes/fault (though we currently do not know what this means).
- On the read-only run, we allocated 3072 MiB of virtual memory but only 1 MiB of physical memory.

Given what we observed in the previous post, any model of how memory works must explain this behavior.  But which model is a good starting point?  For that, we will use the highly academic, battle-tested approach of "taking an educated guess."

## Model #1:  Base and Bounds

Let's take two hardware registers within the CPU:  one for the base and one for the bounds.  Here, each program assumes it starts at address zero but, when running, the hardware translates the virtual address using the following formula:

```
physical address = virtual address + base
```

Note that there is one base and bounds pair per process:  the OS saves and restores each process's pair on a context switch.

Here, the bounds register just provides protection, checking that we do not go out of bounds.

While this approach is simple, there is one huge flaw:  internal fragmentation.  That is, the space between the stack and heap is wasted — internal, as the wasted space is inside the allocated unit.  Most programs are small (MBs) and this gap can waste TiBs:

```
address space (base to bounds):    2^47 bytes = 128 TiB   (what we measured in part 2)
code + heap (bottom):            ~ 4 MiB                  (hello world is tens of KiB of code)
stack (top):                     ~ 8 MiB                  (default Linux stack limit: ulimit -s = 8192 KiB)

wasted gap in the middle:          128 TiB - 12 MiB ≈ 128 TiB   (~99.99999% of the allocation)
```

In fact, on a real base and bounds machine our part 2 experiment could never have gotten this far:  handing out address space *is* handing out physical memory under this model, so the mmap loop would have failed at roughly the size of our RAM (tens of GiB at best) — not at 131071 GiB.

Moreover, for large programs, it becomes challenging to run a program when the entire address space does not easily fit into memory.  We need a new model.

## Model #2:  Segmentation

It was nice how simple Base and Bounds was.  Let's try to keep it.  To patch it up, instead of just having one base and bounds pair in our MMU (memory management unit), let's have a base and bounds pair per logical segment of the address space — logical meaning the natural divisions of the address space we already know:  code, heap, and stack ([OSTEP chapter 16](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf)).  A segment is just a division of our usual address space (one segment each for code, stack, and heap); giving each segment its own base and bounds allows the OS to place each segment independently in physical memory and thus avoid filling physical memory with unused virtual address space, as happened when we mandated that code, stack, and heap be placed in one single unit.  As before, this state is per process:  each process gets its own set of (segment, base, bounds) tuples, saved and restored on a context switch.

Here is an example of what this translation could look like:

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
2.  Malloc may grow the heap.  The OS may end up having to copy the entire heap to a bigger free hole somewhere else.  This can also lead to immense external fragmentation.
3.  Segmentation is still not fully flexible enough to support a generalized, sparse address space.  For instance, if we make a `std::vector<int> a(n)` for some huge `n` and only access `a[0]` and `a[n-1]`, then we still need a large heap.  In other words, if we have a large but sparsely-used heap all in one logical segment, the entire heap must still reside in memory in order to be accessed.

## Scorecard

Before guessing again, let's check our models against what we measured in part 2:

| Observation from part 2 | Base and Bounds | Segmentation |
|---|---|---|
| Same virtual address in two programs maps to different physical addresses | Explained:  each process gets its own base | Explained:  each process gets its own segment bases |
| 47-bit (128 TiB) address space | Fails:  needs 128 TiB of contiguous physical RAM | Partial:  the heap–stack gap costs nothing, but a large sparse heap still needs full backing |
| First touch of memory is much slower than the second | Unexplained:  everything is mapped up front, so both touches should cost the same | Unexplained:  same story |
| 4095 bytes/fault | Unexplained:  nothing in this model works in ~4 KiB units | Unexplained:  same story |
| 3072 MiB virtual but only 1 MiB physical on the read-only run | Fails:  everything from base to bounds is physically backed | Fails:  the whole heap segment must be resident to be accessed |

So, we need yet another model.  For our final guess...

## Model #3:  Paging

As we will see, this will be the model we need to explain everything.  Instead of fragmenting memory three ways, let's further chop it up into fixed-size pieces:  pages.  Now, the 64-bit virtual address has two components:  a p-bit virtual page offset (VPO) and a (64-p)-bit virtual page number (VPN).  (In practice, today's x86-64 hardware translates only the low 48 of those 64 bits, and user space gets the lower half — hence the 47-bit address space we measured in part 2.)  The memory management unit (MMU) uses the VPN to select the appropriate page table entry (PTE).  The final physical address is just the concatenation of the physical page number (PPN) from the page table entry and the VPO from the virtual address.

Similar to the first two models, there is a page table base register (PTBR) that points to the current per-process page table.  When a given process is running, the address of its page table sits inside that special register (CR3 on x86-64) and is swapped in and out on context switches.

Moreover, we can now explain why first touch of memory is so slow:  page faults.  When we have a virtual address whose page is *not* yet present in physical memory, we have a page fault when we first access the memory — the `PAGE_FAULT` branch in the code below — and the OS must page in our page before the instruction can finish.

Also, we can bookkeep which virtual addresses are valid without backing them with any physical memory until they are first touched; in the earlier models, every virtual address had to be immediately backed.  Because of this lazy ability, we get the feeling of far more memory than we would otherwise have:  in models #1 and #2, everything we allocated had to fit in our physical memory (often ~32 GB), while paging gives us 2^47 bytes = 131072 GiB (= 128 TiB) of virtual memory!

In fact, these page faults explain what the 4095 bytes/fault means:  each fault maps in one page, and a page on Linux is 4096 bytes (2^12 = 4 KiB).  Check the numbers from part 2:  we touched 3072 MiB and the read loop took 786432 page faults — exactly 3072 MiB / 4096 bytes.  The extra 161 faults the rest of the process took drag the printed average down to 4095.

Still, even for small programs we'd need the full linear page table.  This can be expensive.  For instance, a 32-bit address space (2^32 = 4 GB) with 4 KiB pages and a 20-bit VPN implies there are 2^20 ≈ 1 million virtual to physical address translations the OS would need to manage.  Assuming we'd need 4 bytes per page table entry, that means 2^20 * 4 bytes = 4 MB of memory needed for each page table per running process.  For a quick sanity check:  those 2^20 entries each map a 4 KiB page, and 2^20 * 2^12 = 2^32 bytes = 4 GB — the whole address space.  Moreover, for the 64-bit example with the 47-bit address space we measured in part 2, a flat table would need 2^47 / 4 KiB = 2^35 entries, or 128 GiB, per process!  Going back to our 32-bit example, while 4 MB may not sound like a lot, it is important to note that a machine typically runs hundreds of processes at once (`ps -e | wc -l` on an idle Linux desktop easily shows 200+), so 250 processes * 4 MB ≈ 1 GB of RAM used just for the tables... and RAM is already expensive as is.  On a 64-bit system, a single flat page table would not even fit on most computers.

Instead, to solve that issue, we introduce multi-level page tables:  instead of one large linear array, we have layers of them.  Conceptually, a multi-level page table resembles a tree.  Let's consider a simple two-level setup:  instead of using the 20-bit VPN as one index into one giant array, we split it up such that the first 10 bits are for the first level and the last 10 bits are for the next level.  Why 10?  Because a 4 KiB page holding 4-byte entries fits 4 KiB / 4 bytes = 2^10 of them — each table is exactly one page.  Then, each of the first level's PTEs points to one level-2 table:  a single 4 KiB page holding 1,024 PTEs, each mapping one page.  In general, we have 1,024 level-1 entries * 1,024 PTEs per level-2 table * 4 KiB per page = 4 GB, so the two-level tree covers our entire virtual address space — we lose nothing relative to the flat table.  However, this dramatically saves on memory:  if some PTE in level 1 is null, we simply never allocate the level-2 table it would point to — that table's 4 MB slice of the address space just isn't mapped.

For example, if a program only uses a single 4 MB chunk of its address space (which needs 4 MB / 4 KiB = 1,024 pages — exactly one level-2 table), this is a huge saving.  That is, with a single linear map for a 32-bit system we needed 4 MB of space no matter what; with two levels, such a program needs the 4 KiB level-1 table plus one 4 KiB level-2 table — just 8 KiB, a 512x improvement on how much memory page tables consume (and the always-resident minimum is just the 4 KiB level-1 table, a full 1000x less).  Moreover, only the level 1 table needs to be in main memory at all times:  the level 2 page tables can be created and paged in and out by the VM system itself, which greatly reduces pressure on main memory.

It is worth noting that this multi-level process generalizes:  instead of two levels we could have k levels.  For instance, in our 64-bit example (with the 47-bit address space we measured, so a 47 - 12 = 35-bit VPN), we have 2^35 ≈ 34 billion pages.  This means that for a program that only uses a single 4 MB chunk of its address space we'd only need one 4 KiB table per level along the path:  if we keep our 10-bit levels, that is k = ceil(35 / 10) = 4 levels, so 4 * 4 KiB = 16 KiB of page tables — compared to 128 GiB this is an obvious win (a factor of about 8 million).  (Real x86-64 lands in the same place:  four levels, just with 9-bit indices and 8-byte PTEs — 4 KiB / 8 bytes = 2^9.)  While it may seem expensive to dereference memory k times for a single address translation, it is important to remember (and thank) the TLB, introduced below:  on a hit we skip the walk entirely.  Still, there are reasons to avoid using a large number of levels, which will be further discussed in the next post.

Now that we understand paging, we can explain where the 47-bit address space comes from:  4 levels * 9 bits per level + 12 offset bits = 48 translatable bits, and user space gets half of that, so it effectively only gets 47 bits.  Our earlier failure at 131071 GiB (≈ 2^47 bytes) is now fully understood.

Unfortunately, this entire process is incredibly slow.  That's where more caching comes in, in the form of the translation lookaside buffer (TLB).  A TLB has a high degree of associativity (often fully associative):  a translation can live in any TLB slot and the hardware compares against every slot in parallel, so we rarely miss just because two pages fought over the same slot.

Now, with a TLB, we first try to get the translation from the TLB, skipping the expensive page-table walk out to main memory on a hit.  This is the difference between roughly a cycle (a hit is overlapped with the L1 cache lookup) and tens to hundreds of cycles for a miss (a page-table walk on x86-64 is up to four dependent memory accesses because each level in a multi-level setup requires a memory access).

Now, for the full pseudocode for this system:

```
VPN    = (VirtualAddress & VPN_MASK) >> SHIFT
Offset = VirtualAddress & OFFSET_MASK

(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success == True)    // TLB Hit
    if (CanAccess(TlbEntry.ProtectBits) == True)
        PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
        Register = AccessMemory(PhysAddr)
    else
        RaiseException(PROTECTION_FAULT)
else                    // TLB Miss
    PDIndex = (VPN & PD_MASK) >> PD_SHIFT
    PDEAddr = PTBR + (PDIndex * sizeof(PDE))
    PDE     = AccessMemory(PDEAddr)
    if (PDE.Valid == False)
        RaiseException(SEGMENTATION_FAULT)
    else if (PDE.Present == False)
        RaiseException(PAGE_FAULT)

    PTIndex = VPN & PT_MASK
    PTEAddr = (PDE.PFN << SHIFT) + (PTIndex * sizeof(PTE))
    PTE     = AccessMemory(PTEAddr)
    if (PTE.Valid == False)
        RaiseException(SEGMENTATION_FAULT)
    else if (PTE.Present == False)
        RaiseException(PAGE_FAULT)
    else if (CanAccess(PTE.ProtectBits) == False)
        RaiseException(PROTECTION_FAULT)
    else
        TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
        RetryInstruction()
```

*(Pseudocode adapted from [OSTEP chapter 19](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-tlbs.pdf), Figure 19.1, extended with the multi-level walk of [chapter 20](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-smalltables.pdf) and the page-fault path of [chapter 21](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys.pdf).  Note the terminology:  OSTEP says PFN — page frame number — for what CS:APP and the text above call the PPN, and OSTEP's `Offset` is our VPO.)*

Why is the exception called a `SEGMENTATION_FAULT`?  This is just a relic from the past — the days of Model #2!  Naming is hard...

There is still a ton to learn when it comes to paging.  For instance:

- How does general caching fit into this (does the cache work on virtual or physical addresses)?
- How does the TLB handle context switches?
- What are big pages?
- How does Linux represent all of this?

For those questions, we will need another part.

## Final Scorecard

So, how did our final guess do?  Let's grade all three models against what we measured in part 2:

| Observation from part 2 | Base and Bounds | Segmentation | Paging |
|---|---|---|---|
| Same virtual address in two programs maps to different physical addresses | Explained:  each process gets its own base | Explained:  each process gets its own segment bases | Explained:  each process gets its own page table — a context switch just swaps the PTBR |
| 47-bit (128 TiB) address space | Fails:  needs 128 TiB of contiguous physical RAM | Partial:  the heap–stack gap costs nothing, but a large sparse heap still needs full backing | Explained:  multi-level tables map only what is used, so a huge, sparse address space costs KiBs of tables — not 128 TiB of RAM |
| First touch of memory is much slower than the second | Unexplained:  everything is mapped up front, so both touches should cost the same | Unexplained:  same story | Explained:  first touch page-faults, and the OS must map the page in before retrying; later touches skip all of that (and usually hit the TLB) |
| 4095 bytes/fault | Unexplained:  nothing in this model works in ~4 KiB units | Unexplained:  same story | Explained:  each fault maps exactly one 4096-byte page; the handful of unrelated faults drag the printed average to 4095 |
| 3072 MiB virtual but only 1 MiB physical on the read-only run | Fails:  everything from base to bounds is physically backed | Fails:  the whole heap segment must be resident to be accessed | Explained:  every untouched page maps to the same shared, read-only zero page; a real frame appears only when we write |

Paging goes five for five.

## Resources:

- [Computer Systems: A Programmer's Perspective (CS:APP), 3rd Edition, Randal E. Bryant and David R. O'Hallaron](https://csapp.cs.cmu.edu/): the classic systems textbook from the programmer's point of view.  Chapter 9 (Virtual Memory) covers address translation, TLBs, and multi-level page tables end to end.
- [Low Latency Optimization: Understanding Huge Pages (Part 1), Hudson River Trading](https://www.hudsonrivertrading.com/hrtbeat/low-latency-optimization-part-1/): a practitioner's take on everything above — how TLB misses and page-table walks cost real latency, and why huge pages exist to reduce both.
- [Operating Systems: Three Easy Pieces (OSTEP), Remzi and Andrea Arpaci-Dusseau](https://pages.cs.wisc.edu/~remzi/OSTEP/): free textbook whose virtual memory chapters cover everything this post touches.  For this post specifically, read chapters [13 (Address Spaces)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-intro.pdf), [15 (Address Translation)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf), [16 (Segmentation)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf), [18 (Introduction to Paging)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-paging.pdf), [19 (Translation Lookaside Buffers)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-tlbs.pdf), [20 (Advanced Page Tables)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-smalltables.pdf), and [21 (Swapping: Mechanisms)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys.pdf).