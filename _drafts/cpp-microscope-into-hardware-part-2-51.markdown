---
layout: post
title:  "C++ as a Microscope Into Hardware, Part 2.51 (Virtual Memory, Continued)"
date:   TODO
categories: cpp
series: "C++ as a Microscope Into Hardware"
part: 2.51
---

*WORK IN PROGRESS*

*The following was written by me, but reviewed by Claude for grammar and general mistakes.*

*This is Part 2.51 of a series on Linus Boehm's C++Now 2025 talk, [C++ as a Microscope Into Hardware](https://www.youtube.com/watch?v=KFe6LCcDjL8).*

// TODO:  intro — recap [part 2.5](/cpp/2026/09/18/cpp-microscope-into-hardware-part-2-5.html) ("why paging") and set up what this post covers

// TODO:  drawbacks of multi-level page tables and why big pages exist in the first place (from part 2.5's appendix TODO)

// TODO:  how the TLB works with context switches (from part 2.5's appendix TODO)

// TODO:  how does general caching fit in — do caches work on virtual or physical addresses?  (from part 2.5's closing list)

// TODO:  how does Linux represent all of this?  (from part 2.5's closing list)

// TODO:  where do page tables themselves live?  (cut from 2.5:  the kernel reaches its page tables through its own virtual addresses, while the hardware's page-table walker follows physical addresses, so the recursion bottoms out)

## Resources:

- [Computer Systems: A Programmer's Perspective (CS:APP), 3rd Edition, Randal E. Bryant and David R. O'Hallaron](https://csapp.cs.cmu.edu/): the classic systems textbook from the programmer's point of view.  Chapter 9 (Virtual Memory) covers address translation, TLBs, and multi-level page tables end to end.
- [Low Latency Optimization: Understanding Huge Pages (Part 1), Hudson River Trading](https://www.hudsonrivertrading.com/hrtbeat/low-latency-optimization-part-1/): a practitioner's take on how TLB misses and page-table walks cost real latency, and why huge pages exist to reduce both.
- [Operating Systems: Three Easy Pieces (OSTEP), Remzi and Andrea Arpaci-Dusseau](https://pages.cs.wisc.edu/~remzi/OSTEP/): free textbook whose virtual memory chapters cover everything this post touches.  For this post specifically, read chapters [19 (Translation Lookaside Buffers)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-tlbs.pdf), [20 (Advanced Page Tables)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-smalltables.pdf), and [21 (Swapping: Mechanisms)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys.pdf).
