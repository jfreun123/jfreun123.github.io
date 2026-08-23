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

Given what we observed in the previous post, any model of how memory works must explain this behavior.  But which model is a good starting point?  For that, we will do the highly academic, battlement approach of "taking an educated guess."  

## Model #1:  Base and Bounds

Let's take two hardware registers within the CPU:  one for the base register and one for the bounds.  Here, each program assumes it as address zero but, when running, the OS translate the virtual adress using the following formula:


// TODO:  clarify there is one base and bounds pair per process

// todo:  mathify this:  physical address = virtual address + base

Here, the bounds register just helps for protection that we do not go out of bounds. 

While this approach is simple, there is one huge flaw:  internal fragmentation:  That is, the space between the stack and heap is wasted in a huge amount of internal fragmentation-- internal as the wasted space is inside the allocated unit.   Most programs are small (MBs) and this wasted space can waist GBs

// Todo:  show the math of this waste

## Model #2:  Segmentation  

// TODO:  clarify there is one base and bounds and segment tuple per process 

It was nice how simple Base and Bounds was.  Let's try to keep it.  To patch it up, instead of just having one base and bounds pair in our MMU (memory management unit), let's have a base and bounds pair per logical (Specify what logical means here) segment of the address space.  // TODO:  add chapter 16 of ostep to the references.  A segment is just a division of our usual address space that allows the OS to place each base and bound in its own segment and thus avoid filling physical memory with unused virtual address space.  

So, if we have n segments chopping up our previous base and bounds block we will now be n times smaller!  Huge improvement as we can control n.  


## Resources:

- [Operating Systems: Three Easy Pieces (OSTEP), Remzi and Andrea Arpaci-Dusseau](https://pages.cs.wisc.edu/~remzi/OSTEP/): free textbook whose virtual memory chapters cover everything this post touches.  For this post specifically, read chapters [13 (Address Spaces)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-intro.pdf), [15 (Address Translation)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf), [18 (Introduction to Paging)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-paging.pdf), [19 (Translation Lookaside Buffers)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-tlbs.pdf), [20 (Advanced Page Tables)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-smalltables.pdf), and [21 (Swapping: Mechanisms)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys.pdf).