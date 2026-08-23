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

Given what we observed in the previous post, any model of how memory works must explain this behavior.

## Resources:

- [Operating Systems: Three Easy Pieces (OSTEP), Remzi and Andrea Arpaci-Dusseau](https://pages.cs.wisc.edu/~remzi/OSTEP/): free textbook whose virtual memory chapters cover everything this post touches.  For this post specifically, read chapters [13 (Address Spaces)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-intro.pdf), [15 (Address Translation)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf), [18 (Introduction to Paging)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-paging.pdf), [19 (Translation Lookaside Buffers)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-tlbs.pdf), [20 (Advanced Page Tables)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-smalltables.pdf), and [21 (Swapping: Mechanisms)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys.pdf).