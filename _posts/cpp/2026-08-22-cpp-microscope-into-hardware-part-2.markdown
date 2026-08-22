---
layout: post
title:  "C++ as a Microscope Into Hardware, Part 2 (Memory)"
date:   2026-08-22 00:00:00 -0400
categories: cpp
series: "C++ as a Microscope Into Hardware"
part: 2
---


*The following was written by me, but reviewed by Claude for grammar and general mistakes.*

*This is Part 2 of a series on Linus Boehm's C++Now 2025 talk, [C++ as a Microscope Into Hardware](https://www.youtube.com/watch?v=KFe6LCcDjL8).*

It is known that the CPU has improved far far more than memory.  As a result, it is critical to understand how memory works as it is the key bottleneck to watch out for when developing a system that does something useful.

## Thinking about memory

([memory_layout.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/memory_layout.cpp) in my [cpp_study](https://github.com/jfreun123/cpp_study) repo)

```cpp
#include <algorithm>
#include <cstdio>
#include <cstdlib>
#include <string>
#include <vector>

int global = 10;
void foo() {}

int main(int argc, char** argv) {
  int stack1;
  int stack2;
  int* heap1 = new int();
  int* heap2 = new int();

  std::vector<std::pair<std::string, void*>> items;
  items.emplace_back("globals", reinterpret_cast<void*>(&global));
  items.emplace_back("functions", reinterpret_cast<void*>(&foo));
  items.emplace_back("stack1", reinterpret_cast<void*>(&stack1));
  items.emplace_back("stack2", reinterpret_cast<void*>(&stack2));
  items.emplace_back("heap1", reinterpret_cast<void*>(heap1));
  items.emplace_back("heap2", reinterpret_cast<void*>(heap2));
  items.emplace_back("command line args", reinterpret_cast<void*>(argv));
  items.emplace_back("environment variables", getenv("HOME"));

  std::ranges::sort(items, std::greater<>{}, [](const auto& p) { return p.second; });
  for (const auto& [name, ptr] : items) {
    printf("%-25s %p\n", name.c_str(), ptr);
  }
}
```

```bash
g++ -std=c++23 -O0 memory_layout.cpp -o memory_layout
./memory_layout
```

```
environment variables     0x7ffe697bae7c
command line args         0x7ffe697b9468
stack1                    0x7ffe697b92bc
stack2                    0x7ffe697b92b8
heap2                     0x15d22d0
heap1                     0x15d22b0
globals                   0x40b080
functions                 0x4021f6
```

Note, the globals address is encoded in the binary!  In fact, disassembling and grepping for the address gives

```bash
objdump -d -C -M intel memory_layout | grep -A1 "mov.*0x40b080"
```

```
  402294:	48 c7 85 70 ff ff ff 	mov    QWORD PTR [rbp-0x90],0x40b080
  40229b:	80 b0 40 00
```

(a `mov` instruction storing the global's address-- and the instruction is so long that objdump wraps its raw bytes onto a second line, which happens to show the address as `80 b0 40 00`: little-endian, just like Part 1)

([same_address.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/same_address.cpp) in my [cpp_study](https://github.com/jfreun123/cpp_study) repo)

```cpp
#include <stdio.h>

#include <atomic>
#include <chrono>
#include <thread>

std::atomic<int> num = 40;

int main() {
  ++num;
  std::this_thread::sleep_for(std::chrono::seconds(1));
  printf("addr: %p, val: %d\n", reinterpret_cast<void*>(&num), num.load());
}
```

```bash
g++ -O2 same_address.cpp -o same_address
./same_address & ./same_address
```

```
addr: 0x404028, val: 41
addr: 0x404028, val: 41
```

Note, they have the same address but two independent values-- each process incremented its own copy of `num` from 40 to 41 (if the address were shared, the second copy would have printed 42)!  Therefore, the address can not be real.  Indeed, it is a virtual address.

How that translation actually works (page tables, the TLB, huge pages) deserves its own post-- that will be Part 2.5.

Now, how big *is* the stack?

([stack_size.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/stack_size.cpp) in my [cpp_study](https://github.com/jfreun123/cpp_study) repo)

```cpp
#include <iostream>

static constexpr auto ONE_MiB = (1U << 20U);
struct OneMiB { char data[ONE_MiB]; };

void measureStackSize(OneMiB* first_addr) {
  OneMiB local_var;
  const auto diff = first_addr - &local_var;
  std::cout << "stack size: " << diff << " MiB" << std::endl;
  measureStackSize(first_addr);  // recurse
}

int main() {
  OneMiB first;
  measureStackSize(&first);
  return 0;
}
```

```bash
g++ -O0 stack_size.cpp -o stack_size
./stack_size
```

```
stack size: 1 MiB
stack size: 2 MiB
stack size: 3 MiB
stack size: 4 MiB
stack size: 5 MiB
stack size: 6 MiB
Segmentation fault
```

That matches `ulimit -s` reporting 8192 KiB: `main`'s own 1 MiB frame plus 6 printed MiB puts the 8th over the limit.

What about the heap?

([heap_limit.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/heap_limit.cpp) in my [cpp_study](https://github.com/jfreun123/cpp_study) repo)

```cpp
#include <chrono>
#include <iostream>
#include <thread>

static constexpr auto ONE_GiB = (1U << 30U);
struct OneGiB { char data[ONE_GiB]; };

int main() {
  OneGiB* curr_addr;

  for (int cnt = 0;; ++cnt) {
    try {
      curr_addr = new OneGiB;
    } catch (const std::bad_alloc& e) {
      std::cout << "malloc failed after: " << cnt << " GiB" << std::endl;
      break;
    }
  }
  // for htop
  std::this_thread::sleep_for(std::chrono::seconds(30));
}
```

```bash
g++ -O0 heap_limit.cpp -o heap_limit   # -O2 deletes the never-touched allocations and the loop never fails!
./heap_limit
```

```
malloc failed after: 131071 GiB
```

But, I guarantee your machine does not have that much.  Indeed, let's use `htop` (here, its batch-friendly cousin `top -b`) to find out how much virtual memory we are using compared to actual memory.

```bash
top -b -n 1 -p $(pgrep heap_limit)
```

```
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  255 root      20   0  128.0t 529772   2988 S   0.0   6.6   0:00.35 heap_limit
```

So, the OS will happily give us 128 TiB of virtual memory but will not back it up with real memory until we actually touch it-- and that first touch of each page is a page fault, which we will measure next.  (Why exactly 131071 GiB?  See Appendix 1.)

Now, let's change this code to actually back it (and therefore, value initialize the data).  Now, the memory is getting backed

([heap_no_write.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/heap_no_write.cpp) and [heap_write.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/heap_write.cpp) in my [cpp_study](https://github.com/jfreun123/cpp_study) repo-- the talk allocates 30 GiB, scaled down to 4 GiB here to fit the docker VM.  Note the empty `asm volatile`: without it, gcc 15 at `-O3` deletes the untouched allocation in one case and the never-read zeroing in the other, and the whole demo silently vanishes.)

```cpp
// heap_no_write.cpp
static constexpr auto ONE_GiB = (1U << 30U);
struct OneGiB { char data[ONE_GiB]; };

int main() {
  OneGiB* curr_addr;

  for (int cnt = 0; cnt < 4; ++cnt) {
    curr_addr = new OneGiB;
    // keep gcc 15 from optimizing the unused allocation away
    asm volatile("" : : "g"(curr_addr) : "memory");
  }
}
```

<iframe width="100%" height="400px" src="https://godbolt.org/e?hideEditorToolbars=true#compiler:g152,filters:'demangle,commentOnly,libraryCode,trim,intel,directives,labels',options:'-O3',source:'//+heap_no_write.cpp%0Astatic+constexpr+auto+ONE_GiB+%3D+(1U+%3C%3C+30U)%3B%0Astruct+OneGiB+%7B+char+data%5BONE_GiB%5D%3B+%7D%3B%0A%0Aint+main()+%7B%0A++OneGiB*+curr_addr%3B%0A%0A++for+(int+cnt+%3D+0%3B+cnt+%3C+4%3B+%2B%2Bcnt)+%7B%0A++++curr_addr+%3D+new+OneGiB%3B%0A++++//+keep+gcc+15+from+optimizing+the+unused+allocation+away%0A++++asm+volatile(%22%22+%3A+%3A+%22g%22(curr_addr)+%3A+%22memory%22)%3B%0A++%7D%0A%7D%0A'"></iframe>

([open in Compiler Explorer](https://godbolt.org/z/Kx9PqK4qr)-- delete the `asm volatile` line and watch gcc reduce `main` to `xor eax, eax` + `ret`: all four GiB of allocations, gone)

```cpp
// heap_write.cpp
static constexpr auto ONE_GiB = (1U << 30U);
struct OneGiB { char data[ONE_GiB]; };

int main() {
  OneGiB* curr_addr;

  for (int cnt = 0; cnt < 4; ++cnt) {
    curr_addr = new OneGiB{};  // value init
    asm volatile("" : : "g"(curr_addr) : "memory");
  }
}
```

Running perf, we can see how these codes differ.  In fact, the value initialized version takes so long because we have a huge amount of page faults.

```bash
g++ -O3 heap_no_write.cpp -o heap_no_write
g++ -O3 heap_write.cpp -o heap_write
perf stat -e duration_time ./heap_no_write
perf stat -e page-faults ./heap_no_write
perf stat -e duration_time ./heap_write
perf stat -e page-faults ./heap_write
```

(inside docker, perf needs the container started with `--privileged` and an `apt-get install linux-perf`)

```
 Performance counter stats for 'system wide':

           3337515 ns   duration_time

 Performance counter stats for './heap_no_write':

               133      page-faults

 Performance counter stats for 'system wide':

        5422437556 ns   duration_time

 Performance counter stats for './heap_write':

           1048709      page-faults
```

3 milliseconds vs 5.4 seconds, and 1,048,709 page faults is almost exactly the 4 GiB / 4 KiB = 1,048,576 pages we touched-- one fault per page.

Before continuing, let's bring in `tsc`-- a time stamp counter.  The profiler below reads it with `__rdtsc()` and pairs it with the process's page fault count from `getrusage`, so every scope reports both time and faults ([profiler.hpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/page_faults/profiler.hpp) / [profiler.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/page_faults/profiler.cpp) / [linux_stuff.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/page_faults/linux_stuff.cpp), ported from Linus's [perf_stuff](https://github.com/linusboehm/perf_stuff) repo into [cpp_study](https://github.com/jfreun123/cpp_study/tree/main/blog/cpp_microscope_into_hardware/page_faults)).

We can use `tsc` and the following read only program to see that:

([page_faults_read.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/page_faults/page_faults_read.cpp) in my [cpp_study](https://github.com/jfreun123/cpp_study) repo)

```cpp
#include <cstdint>
#include <cstring>

#include "profiler.hpp"

static constexpr auto ONE_GiB = (1U << 30U);
static constexpr auto THREE_GiB = 3 * ONE_GiB;

uint64_t read(char* array, std::size_t size) {
  auto data64 = reinterpret_cast<const uint64_t*>(array);
  uint64_t res = 0;
  for (std::size_t i = 0; i < size / sizeof(uint64_t);) {
    res += data64[i++];
    res += data64[i++];
    res += data64[i++];
    res += data64[i++];
  }
  return res;
}

int main() {
  char* address;
  uint64_t result = 0;

  PerfStuff::Profile::Profiler profiler;
  // ///////////////
  // ALLOCATE MEMORY
  {
    PerfStuff::Profile::ProfileScope a(0, profiler);
    address = do_mmap(THREE_GiB);
  }

  // ///////////////
  // READ MEMORY (touch every byte)
  for (const int run : {1, 2}) {
    PerfStuff::Profile::ProfileScope a(run, profiler);
    result += read(address, THREE_GiB);
  }
  std::cout << "Result: " << result << '\n';
}
```

```bash
g++ -O2 page_faults_read.cpp profiler.cpp linux_stuff.cpp -o page_faults_read
./page_faults_read
```

```
Result: 0

Total time: 692.4736 ms
Total phys mem: 1 MiB
Total virt mem: 3072 MiB
Total page faults: 786593 (bytes per fault: 4095 bytes/fault)
  0. ProfileScope:  #calls: 1
    Time: 7.080us | cycles: 9848 | 0.00%
    page faults: 0
  1. ProfileScope:  #calls: 1
    Time: 628.920ms | cycles: 874850620 | 90.82%
    page faults: 786432
  2. ProfileScope:  #calls: 1
    Time: 62.109ms | cycles: 86396050 | 8.97%
    page faults: 0
```

- mmap is fast
- reading the data on the first iteration takes forever due to page faults.
- reading the data on the second iteration is much much faster
- While we use 3072 MiB of virtual memory, we only use 1 MiB of physical memory.  Since everything is just 0s, the OS is clever enough to re-use the same "zero" page for all the virtual pages.
- Moreover, bytes per fault shows 4095 bytes/fault which turns out to be the page size

This is why allocators and memory pools is critical!  Allocating memory on startup, not in the hot path, can greatly minimize page faults.

Now, when we write ([page_faults.cpp](https://github.com/jfreun123/cpp_study/blob/main/blog/cpp_microscope_into_hardware/page_faults/page_faults.cpp), the same program plus this block), we see

```cpp
  // ///////////////
  // WRITE MEMORY
  for (const int run : {3, 4}) {
    PerfStuff::Profile::ProfileScope a(run, profiler);
    std::memset(address, 1, THREE_GiB);
  }
```

```bash
g++ -O2 page_faults.cpp profiler.cpp linux_stuff.cpp -o page_faults
./page_faults
```

```
Result: 1736164148112261120

Total time: 11879.0835 ms
Total phys mem: 3073 MiB
Total virt mem: 3072 MiB
Total page faults: 1573024 (bytes per fault: 2047 bytes/fault)
  0. ProfileScope:  #calls: 1
    Time: 6.530us | cycles: 9084 | 0.00%
    page faults: 0
  1. ProfileScope:  #calls: 1
    Time: 613.908ms | cycles: 853966726 | 5.17%
    page faults: 786432
  2. ProfileScope:  #calls: 1
    Time: 62.012ms | cycles: 86261276 | 0.52%
    page faults: 0
  3. ProfileScope:  #calls: 1
    Time: 10.815s | cycles: 15044181732 | 91.04%
    page faults: 786432
  4. ProfileScope:  #calls: 1
    Time: 126.602ms | cycles: 176107616 | 1.07%
    page faults: 0
```

- the first time we write is much slower than the second time we write.
- when we decide to write to a page we end up page faulting again (because each page can no longer map to the same zero page)

This blog is getting long.  Now that we understand memory, the next blog will talk about the importance of access patterns.

## Appendix 1:  Why fail at 131071 GiB?

- x86-64 uses 48-bit virtual addresses, split canonically down the middle: the lower half belongs to user space, the upper half to the kernel.
- The user half is 2^47 bytes = 128 TiB = 131072 GiB.
- Our binary, libraries, stack, and bookkeeping already occupy a sliver of that, so the 131072nd GiB doesn't fit-- `new` throws `std::bad_alloc` after 131071 (the talk's machine reported 131070 for the same reason).
- Nothing about physical RAM is involved; the wall we hit is the size of the *address space* itself.

<!-- TODO(Jacob): Appendix 2:  What does asm volatile("" : : "g"(curr_addr) : "memory") actually do? -->

## Resources:

- [C++ as a Microscope Into Hardware - Linus Boehm - C++Now 2025 (YouTube)](https://www.youtube.com/watch?v=KFe6LCcDjL8)
- [Slides (PDF, official C++Now 2025 repo)](https://github.com/boostcon/cppnow_presentations_2025/blob/main/Presentations/Cpp_as_a_Microscope_Into_Hardware.pdf)
- [perf_stuff](https://github.com/linusboehm/perf_stuff): Linus Boehm's repo with the talk's actual demo and benchmark code (the profiler and page fault programs above are ported from it).
