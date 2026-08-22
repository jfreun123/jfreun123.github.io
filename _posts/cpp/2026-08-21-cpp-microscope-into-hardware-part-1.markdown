---
layout: post
title:  "C++ as a Microscope Into Hardware, Part 1 (Assembly)"
date:   2026-08-21 00:00:00 -0400
categories: cpp
series: "C++ as a Microscope Into Hardware"
part: 1
---

*The following was written by me, but reviewed by Claude for grammar and general mistakes.*

*This is Part 1 of a series on Linus Boehm's C++Now 2025 talk, [C++ as a Microscope Into Hardware](https://www.youtube.com/watch?v=KFe6LCcDjL8).*

This is one of the best C++ talks available on YouTube-- even with the awful audio and microphone sensitivity.  Almost everything mentioned here is worth a full talk or at least a deeper exploration-- a deeper exploration I'd like to take on in a series of blogs.

## From source to bytes

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

But why is the object dump so not human readable?  It was never meant to be human readable and is worth a blog of its own.

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

## From bytes to assembly

As a SWE, if we had to run this file ourselves, we'd write an emulator: read the bytes in, work out which instruction each clump of 1's and 0's encodes, and then call a function that does what that instruction is supposed to do.  A CPU runs that same loop in hardware, with official names for each stage: fetch -> decode -> execute.

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

Here, mov is defined as `mov <destination> <value>` and is actually a copy.  So, we are copying the hex value 0x0 (which is just 0) into register `eax` which is just the 32-bit return register.  What about `b8 00 00 00 00`?  These five bytes (one hex digit is 4 bits and there are 10 hex digits here) detail the instruction and the argument!  For instance, `b8` in hex translates to `10111000` which directly appears in the previous raw binary dump.  Moreover, if we compiled with gcc at `-O2` we would've seen `xor eax, eax` which has the same effect but is only 2 bytes instead of 5 bytes-- that optimization relieves pressure on the instruction cache (the small, fast memory that holds recently fetched code; smaller instructions mean more of them fit).


## What happens on a call

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

With the following register meanings

| Register | What it holds |
|----------|---------------|
| `rip` | The instruction pointer: the address of the next instruction the CPU will execute.  No instruction writes it directly-- `call`, `ret`, and jumps change it. |
| `rsp` | The stack pointer: the address of the current top of the stack.  The stack grows downward, so `push` subtracts 8 and `pop` adds 8. |
| `rbp` | The base pointer (frame pointer): anchors the current function's stack frame, so locals and saved state sit at fixed offsets from it. |
| `eax` | The lower 32 bits of `rax`, the register the x86-64 ABI uses for return values.  Writing `eax` also zeroes the upper 32 bits of `rax`. |

There are a few interesting things happening here.  Step by step:

1. When `main` is entered, before we ever reach our call, we push (and save) the caller's base pointer `rbp` (the register that anchors the current function's variables on the stack) and then move our stack pointer `rsp` into `rbp`.  `rbp` now holds `main`'s frame.

   After: `rip=0x401115` (our `call`), `rsp=0x7fffffffea00`, `rbp=0x7fffffffea00`, stack top holds the caller's saved `rbp`

2. When we `call return_zero()`, the CPU pushes the return address onto the stack (the address of the next instruction in `main`, here `40111a`) and jumps to `401106`.  Appendix 3 breaks down how the five bytes `e8 ec ff ff ff` encode all of this.

   After: `rip=0x401106`, `rsp=0x7fffffffe9f8`, `rbp=0x7fffffffea00` (unchanged!), stack top holds `0x40111a` (the return address)

3. `return_zero()` runs a similar prologue: it pushes `rbp` (saving `main`'s base pointer) and moves `rsp` into `rbp` to anchor its own frame.

   After: `rip=0x40110a`, `rsp=0x7fffffffe9f0`, `rbp=0x7fffffffe9f0`, stack top holds `0x7fffffffea00` (`main`'s saved `rbp`)

4. The actual work happens: `mov eax,0x0`.

   After: `eax=0x0` (before this it held leftover garbage, `0x401111`); `rsp`, `rbp`, and the stack are untouched

5. Once the work is done, `return_zero()` pops `rbp` off of the stack (restoring `main`'s base pointer) and `ret` pops the return address back into the instruction pointer `rip`-- execution lands on the `nop` in `main`.

   After: `rip=0x40111a` (the `nop`), `rsp=0x7fffffffea00`, `rbp=0x7fffffffea00`-- exactly the state at the end of step 1

6. `main`'s own epilogue mirrors this: `pop rbp`, then `ret`.  Whatever is in `eax` (our 0) becomes `main`'s return value.

   After: `rsp=0x7fffffffea08`, `rbp=0x1` (the caller's, restored), `rip` back in libc's startup code, `eax=0x0` on its way to becoming our exit code

One thing that tripped me up: right after each prologue, `rbp` and `rsp` hold the same value, so why have both?  The difference is what happens next.  `rsp` moves constantly-- every `push`, `pop`, and `call` changes it-- while `rbp` is a copy of `rsp` frozen at function entry.  That frozen anchor is what gives local variables stable addresses for the whole function body.  Our `return_zero` has no locals, so the `push rbp` / `mov rbp,rsp` dance is pure ceremony here-- which is exactly why gcc dropped it at `-O1`, where the whole function was just `mov eax,0x0` and `ret`.  Appendix 4 shows a function where `rbp` has more interesting work to do.

## Analyzing gcc itself

We can keep going to explore more and more binaries but that is tedious.  Instead, we can just analyze gcc itself
```bash
objdump -d $(which gcc)
```

Here, we can get analysis with what registers are most often used and what are the conventions (caller vs callee) and something similar for instructions.  Interestingly, for instructions, `mov` is by FAR the most common instruction.  In fact, the top 12 instructions account for roughly 75% of instructions used!  You do not need to know *that* much assembly to get around reading assembly.

For a more crazy fact, you only need `mov` to be Turing complete (see Resources).  In the cppnow talk, Linus Boehm has a great objdump of using a `mov` only compiler to compile a general `find_primes` function that takes 3-4 thousand lines with `mov` only and only 52 with the full instruction set.  Evidently, `mov` only compilers offer horrific performance.

This blog post is already fairly long and we are only 15 minutes in...will need many more posts.

Next up we will talk about memory.

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

## Appendix 2:  Recreating the gcc instruction analysis

([analyze_instructions.py](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/analyze_instructions.py) in my [cpp_study](https://github.com/jfreun123/cpp_study) repo)

```python
import collections
import shutil
import subprocess
import sys

target = sys.argv[1] if len(sys.argv) > 1 else shutil.which("gcc")
asm = subprocess.run(["objdump", "-d", target], capture_output=True, text=True, check=True).stdout

counts = collections.Counter()
for line in asm.splitlines():
    # instruction lines look like: "  401106:<tab>55<tab>push   rbp"
    parts = line.split("\t")
    if len(parts) >= 3:
        mnemonic = parts[2].split()[0]
        counts[mnemonic] += 1

total = sum(counts.values())
print(f"{target}: {total} instructions, {len(counts)} distinct mnemonics\n")

top = counts.most_common(12)
for mnemonic, n in top:
    print(f"{mnemonic:8} {n:8}  {100 * n / total:5.1f}%")

print(f"\ntop 12 = {100 * sum(n for _, n in top) / total:.1f}% of all instructions")
```

```bash
docker run --rm -v "$PWD":/work -w /work gcc:15 python3 analyze_instructions.py
```

```
/usr/local/bin/gcc: 225790 instructions, 165 distinct mnemonics

mov         75308   33.4%
call        12135    5.4%
cmp         10545    4.7%
je          10479    4.6%
jmp          9700    4.3%
test         8885    3.9%
lea          7944    3.5%
add          7835    3.5%
push         7668    3.4%
jne          6765    3.0%
xor          6586    2.9%
pop          6500    2.9%

top 12 = 75.4% of all instructions
```

## Appendix 3:  Breaking down the call instruction

```
  401115:       e8 ec ff ff ff          call   401106 <return_zero()>
```

- `call` does two things as one instruction: (1) pushes the address of the *next* instruction (`0x40111a`) onto the stack (`rsp -= 8`, then write), (2) sets `rip` to the target.  Morally it is `push 0x40111a` + `jmp 0x401106` fused together.
- Encoding: opcode `e8` means "call rel32"-- the four bytes after it are a signed 32-bit displacement, stored little-endian: `ec ff ff ff` read back-to-front is `0xffffffec`, which as a signed number is `-0x14` (-20).
- The displacement is relative to the NEXT instruction, not the call itself: `0x40111a + (-0x14) = 0x401106`.  `return_zero()` sits exactly 20 bytes before the `nop`.  The absolute address 0x401106 appears nowhere in the machine code.
- Because the target is stored relative, the code still works if the whole block is loaded at a different address-- nothing needs patching.
- `ret` (`c3`) is the exact inverse: it pops the top of the stack into `rip`.  call/ret are a matched pair around the stack.

## Appendix 4:  Why rbp exists

([locals_demo.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/locals_demo.cpp) in my [cpp_study](https://github.com/jfreun123/cpp_study) repo)

```cpp
int locals_demo(int x) {
    int a = x + 1;
    int b = a * 2;
    return a + b;
}
```

```bash
g++ -g -c -O0 locals_demo.cpp
objdump -d -C -M intel locals_demo.o
```

```
0000000000000000 <locals_demo(int)>:
   0:	55                   	push   rbp
   1:	48 89 e5             	mov    rbp,rsp
   4:	89 7d ec             	mov    DWORD PTR [rbp-0x14],edi
   7:	8b 45 ec             	mov    eax,DWORD PTR [rbp-0x14]
   a:	83 c0 01             	add    eax,0x1
   d:	89 45 fc             	mov    DWORD PTR [rbp-0x4],eax
  10:	8b 45 fc             	mov    eax,DWORD PTR [rbp-0x4]
  13:	01 c0                	add    eax,eax
  15:	89 45 f8             	mov    DWORD PTR [rbp-0x8],eax
  18:	8b 55 fc             	mov    edx,DWORD PTR [rbp-0x4]
  1b:	8b 45 f8             	mov    eax,DWORD PTR [rbp-0x8]
  1e:	01 d0                	add    eax,edx
  20:	5d                   	pop    rbp
  21:	c3                   	ret
```

- Every variable is a fixed offset from `rbp`: the argument `x` lives at `[rbp-0x14]`, `a` at `[rbp-0x4]`, `b` at `[rbp-0x8]`-- and those offsets stay valid for the entire function body no matter how much `rsp` moves.
- If the compiler addressed locals relative to `rsp` instead, the right offset would change every time something got pushed-- anchoring to a frozen register makes "where is `a`?" a constant.
- There is no `sub rsp, ...` making room for the locals because a function that calls nothing may freely use the 128 bytes below `rsp` (the "red zone") without moving it.
- The memory at `[rbp]` always holds the caller's saved `rbp`, so the frames form a linked list-- that chain is exactly how a debugger prints a backtrace.

## Appendix 5:  Recreating the register walkthrough

The register values in the step by step came from gdb, executing one instruction at a time.  The gcc:15 image doesn't ship gdb, so install it inside the container (`--security-opt seccomp=unconfined` lets gdb turn off address randomization, keeping the addresses stable between runs):

```bash
docker run --rm -it --security-opt seccomp=unconfined -v "$PWD":/work -w /work gcc:15 bash
apt-get update && apt-get install -y gdb
g++ -g -O0 return_zero.cpp main.cpp -o return_zero
gdb ./return_zero
```

```
(gdb) break main
Breakpoint 1 at 0x401115: file main.cpp, line 4.
(gdb) run
Breakpoint 1, main () at main.cpp:4
4	    return return_zero();
(gdb) info registers rip rsp rbp
rip            0x401115            0x401115 <main()+4>
rsp            0x7fffffffea10      0x7fffffffea10
rbp            0x7fffffffea10      0x7fffffffea10
(gdb) x/gx $rsp
0x7fffffffea10:	0x0000000000000001
(gdb) stepi
return_zero () at return_zero.cpp:1
1	int return_zero() {
(gdb) info registers rip rsp rbp
rip            0x401106            0x401106 <return_zero()>
rsp            0x7fffffffea08      0x7fffffffea08
rbp            0x7fffffffea10      0x7fffffffea10
(gdb) x/gx $rsp
0x7fffffffea08:	0x000000000040111a
```

- `break main` skips past the prologue, so the first stop is exactly the end-of-step-1 state: sitting on the `call`, with `rsp` equal to `rbp`.
- One `stepi` executes the `call`, and there it all is: `rip` at `401106`, `rsp` 8 lower, and the return address `0x40111a` sitting on the stack top.
- Keep alternating `stepi`, `info registers rip rsp rbp`, and `x/gx $rsp` to follow the rest of the walkthrough.
- The exact stack addresses depend on the environment (your environment variables live on the stack above all of this), so yours may differ slightly from the walkthrough's.

## Resources:

- [C++ as a Microscope Into Hardware - Linus Boehm - C++Now 2025 (YouTube)](https://www.youtube.com/watch?v=KFe6LCcDjL8)
- [Slides (PDF, official C++Now 2025 repo)](https://github.com/boostcon/cppnow_presentations_2025/blob/main/Presentations/Cpp_as_a_Microscope_Into_Hardware.pdf)
- [Intel® 64 and IA-32 Architectures Software Developer's Manual, Combined Volumes 1-4 (PDF; Volume 2 is the instruction set reference)](https://cdrdv2.intel.com/v1/dl/getContent/671200)
- [mov is Turing-complete (PDF), Stephen Dolan](https://drwho.virtadpt.net/files/mov.pdf): the proof that x86's `mov` (plus a single jump to loop) can compute anything.
- [M/o/Vfuscator](https://github.com/xoreaxeaxeax/movfuscator): Christopher Domas's C compiler that emits only `mov` instructions-- the compiler behind the `find_primes` demo in the talk.