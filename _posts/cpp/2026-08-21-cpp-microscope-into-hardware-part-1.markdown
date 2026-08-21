---
layout: post
title:  "C++ as a Microscope Into Hardware, Part 1"
date:   2026-08-21 00:00:00 -0400
categories: cpp
series: "C++ as a Microscope Into Hardware"
part: 1
---

*The following was written by me, but reviewed by Claude for grammar and general mistakes.*

*This is Part 1 of a series on Linus Boehm's C++Now 2025 talk, [C++ as a Microscope Into Hardware](https://www.youtube.com/watch?v=KFe6LCcDjL8).*

This is one of the best C++ talks available on YouTube-- even with the awful audio and microphone sensitivity.  Almost everything mentioned here is worth a full talk or at least a deeper exploration-- a deeper exploration I'd like to explore in a series of blogs.

To start, we use his first function

```cpp
int return_zero() {
    return 0;
}
```

([return_zero.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/return_zero.cpp) in my [cpp_study](https://github.com/jfreun123/cpp_study) repo)

For this blog, we will be using gcc on x86-64 with linux.

```bash
g++ --version
g++ (GCC) 15.3.0
Copyright (C) 2025 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

If you aren't on linux, you should be able to recreate this with docker like such

```
docker run --rm -v "$PWD":/work -w /work gcc:15 bash -c 'g++ --version' 
```

Now, compiling with

```bash
g++ -g -c -O1 return_zero.cpp
less return_zero.o
```

```
^?ELF^B^A^A^@^@^@^@^@^@^@^@^@^A^@>^@^A^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@
^@^@p^E^@^@^@^@^@^@^@^@^@^@@^@^@^@^@^@@^@^T^@^S^@M-8^@^@^@^@M-CY^@^@^@^E
^@^A^H^@^@^@^@^A^@^@^@^@!^DM-g^S^C^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^F^@
^@^@^@^@^@^@^@^@^@^@^B^@^@^@^@^A^A^E^@^@^@^@U^@^@^@^@^@^@^@^@^@^@^@^F^@^
@^@^@^@^@^@^AM-^\^C^D^Eint^@^@^A^Q^A%^N^S^KM-^P^A^KM-^Q^A^F^C^_^[^_^Q^A^
R^G^P^W^@^@^B.^@?^Y^C^N:^K;^K9^Kn^NI^S^Q^A^R^G@^Xz^Y^@^@^C$^@^K^K>^K^C^H
^@^@^@,^@^@^@^B^@^@^@^@^@^H^@^@^@^@^@^@^@^@^@^@^@^@^@^F^@^@^@^@^@^@^@^@^
@^@^@^@^@^@^@^@^@^@^@^@^@^@^@L^@^@^@^E^@^H^@*^@^@^@^A^A^AM-{^N^M^@^A^A^A
^A^@^@^@^A^@^@^A^A^A^_^A^@^@^@^@^B^A^_^B^O^B^@^@^@^@^@^@^@^@^@^@^E^S^@	
^B^@^@^@^@^@^@^@^@^A^E^E^S^E^A^F^S^B^F^@^A^Areturn_zero^@GNU C++17 15.3.
0 -mtune=generic -march=x86-64 -g -O1^@_Z11return_zerov^@return_zero.cpp
^@/work^@/work^@return_zero.cpp^@return_zero.cpp^@^@GCC: (GNU) 15.3.0^@^
@^@^@^@^T^@^@^@^@^@^@^@^AzR^@^Ax^P^A^[^L^G^HM-^P^A^@^@^T^@^@^@^\^@^@^@^@
^@^@^@^F^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@
...
```

We can then interpret that as binary as this

```bash
xxd -b return_zero.o
```

```
00000000: 01111111 01000101 01001100 01000110 00000010 00000001  .ELF..
00000006: 00000001 00000000 00000000 00000000 00000000 00000000  ......
0000000c: 00000000 00000000 00000000 00000000 00000001 00000000  ......
00000012: 00111110 00000000 00000001 00000000 00000000 00000000  >.....
00000018: 00000000 00000000 00000000 00000000 00000000 00000000  ......
0000001e: 00000000 00000000 00000000 00000000 00000000 00000000  ......
00000024: 00000000 00000000 00000000 00000000 01110000 00000101  ....p.
0000002a: 00000000 00000000 00000000 00000000 00000000 00000000  ......
00000030: 00000000 00000000 00000000 00000000 01000000 00000000  ....@.
00000036: 00000000 00000000 00000000 00000000 01000000 00000000  ....@.
0000003c: 00010100 00000000 00010011 00000000 10111000 00000000  ......
00000042: 00000000 00000000 00000000 11000011 01011001 00000000  ....Y.
... (434 more lines)
```

As a SWE, we'd try to open the file, interpret all the 1's and 0's, and then call the function that emulates that behavior.  In hardware, we do the more official fetch -> decode -> execute.

Focusing on the decode stage, we can disassemble the binary into assembly instructions (as it is just a mechanical 1:1 decoding of each instruction).

```bash
objdump -d -C -M intel return_zero.o
```

```
return_zero.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <return_zero()>:
   0:	b8 00 00 00 00       	mov    eax,0x0
   5:	c3                   	ret
```

This may look familiar if you've ever used compiler explorer.  In fact

<iframe width="100%" height="300px" src="https://godbolt.org/e?hideEditorToolbars=true#compiler:g152,filters:'demangle,commentOnly,libraryCode,trim,intel,directives,labels',options:'-O1',source:'int+return_zero()+%7B%0A++++return+0%3B%0A%7D'"></iframe>

([open in Compiler Explorer](https://godbolt.org/z/PTbj1aE7T))

Let's focus on this

```
   0:	b8 00 00 00 00       	mov    eax,0x0
```

Here, mov is defined as `mov <destination> <value>` and is actually a copy.  So, we are copying the hex value 0x0 (which is just 0) into register `eax` which is just the 32-bit return register.  What about `b8 00 00 00 00`?  These five bytes (one hex is 4 bits and there are 10 hex numbers here) detail the instruction and the argument!  For instance, `b8` in hex translates to `10111000` which directly appears in the previous raw binary dump.  Moreover, if we compiled with gcc at `-O2` we would've seen `xor eax, eax` which has the same effect but is only 2 bytes instead of 5 bytes-- that optimization relieves pressure on the instruction cache.


Now, what if we called this function?  Let's add a simple main
```cpp
int return_zero();

int main() {
    return return_zero();
}
```

([main.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/main.cpp)-- it lives in its own file, so it needs the declaration; if both functions shared a file, gcc would inline the call away even at `-O1`)

Once we compile and dump with `-O0` (as can be confirmed by adjusting the compiler explorer view above to `-O0` instead of `-O1`), we will see:

```bash
g++ -g -O0 return_zero.cpp main.cpp -o return_zero
objdump -d -C -M intel return_zero
```

```
0000000000401106 <return_zero()>:
  401106:       55                      push   rbp
  401107:       48 89 e5                mov    rbp,rsp
  40110a:       b8 00 00 00 00          mov    eax,0x0
  40110f:       5d                      pop    rbp
  401110:       c3                      ret

0000000000401111 <main>:
  401111:       55                      push   rbp
  401112:       48 89 e5                mov    rbp,rsp
  401115:       e8 ec ff ff ff          call   401106 <return_zero()>
  40111a:       90                      nop
  40111b:       5d                      pop    rbp
  40111c:       c3                      ret
  ```

There are a few interesting things happening here.  Step by step:

1. When `main` is entered, before we ever reach our call, we push (and save) the caller's base pointer `rbp` (the register that anchors the current function's variables on the stack) and then move our stack pointer `rsp` into `rbp`.  `rbp` now holds `main`'s frame.

   After: `rip=0x401115` (our `call`), `rsp=0x7fffffffea00`, `rbp=0x7fffffffea00`, stack top holds the caller's saved `rbp`

2. When we `call return_zero()`, the CPU pushes the return address onto the stack (the address of the next instruction in `main`, here `40111a`) and jumps to `401106`.

   After: `rip=0x401106`, `rsp=0x7fffffffe9f8`, `rbp=0x7fffffffea00` (unchanged!), stack top holds `0x40111a` (the return address)

3. `return_zero()` runs a similar prologue: it pushes `rbp` (saving `main`'s base pointer) and moves `rsp` into `rbp` to anchor its own frame.

   After: `rip=0x40110a`, `rsp=0x7fffffffe9f0`, `rbp=0x7fffffffe9f0`, stack top holds `0x7fffffffea00` (`main`'s saved `rbp`)

4. The actual work happens: `mov eax,0x0`.

   After: `eax=0x0` (before this it held leftover garbage, `0x401111`); `rsp`, `rbp`, and the stack are untouched

5. Once the work is done, `return_zero()` pops `rbp` off of the stack (restoring `main`'s base pointer) and `ret` pops the return address back into the instruction pointer `rip`-- execution lands on the `nop` in `main`.

   After: `rip=0x40111a` (the `nop`), `rsp=0x7fffffffea00`, `rbp=0x7fffffffea00`-- exactly the state at the end of step 1

6. `main`'s own epilogue mirrors this: `pop rbp`, then `ret`.  Whatever is in `eax` (our 0) becomes `main`'s return value.

   After: `rsp=0x7fffffffea08`, `rbp=0x1` (the caller's, restored), `rip` back in libc's startup code, `eax=0x0` on its way to becoming our exit code



## Appendix 1:  What is this other stuff in my disassembly

Our disassembly is actually even more complicated.  It does not just contain `0000000000401111 <main>:` and `0000000000401106 <return_zero()>:`, we also have:

- `_init` (section `.init`): a small initialization stub glibc runs before `main`-- the historical home of global-constructor glue, kept around for compatibility.
- `_start` (section `.text`): the program's true entry point.  The kernel hands control here, not to `main`-- it sets up the stack per the ABI and calls glibc's `__libc_start_main`, which calls `main` and then calls `exit()` with whatever `main` left in `eax`.
- `_dl_relocate_static_pie`: a self-relocation helper that is a no-op here-- it only does real work in static position-independent (static-PIE) builds.
- `deregister_tm_clones` and `register_tm_clones`: hooks for gcc's transactional-memory runtime (libitm)-- effectively no-ops unless that feature is used.
- `__do_global_dtors_aux`: runs global/static destructors during a normal exit.
- `frame_dummy`: runs before `main` to register the stack-unwinding tables used for C++ exceptions.
- `return_zero()` and `main`: the only two functions we actually wrote.
- `_fini` (section `.fini`): the teardown twin of `_init`, run at exit.

This is a far longer discussion and worth another blog of my own.

For a similar analysis, look at this other [great blog](https://oneraynyday.github.io/dev/2020/05/03/Analyzing-The-Simplest-C++-Program/)
## Resources:

- [C++ as a Microscope Into Hardware - Linus Boehm - C++Now 2025 (YouTube)](https://www.youtube.com/watch?v=KFe6LCcDjL8)
- [Slides (PDF, official C++Now 2025 repo)](https://github.com/boostcon/cppnow_presentations_2025/blob/main/Presentations/Cpp_as_a_Microscope_Into_Hardware.pdf)
- [Intel® 64 and IA-32 Architectures Software Developer's Manual, Combined Volumes 1-4 (PDF; Volume 2 is the instruction set reference)](https://cdrdv2.intel.com/v1/dl/getContent/671200)