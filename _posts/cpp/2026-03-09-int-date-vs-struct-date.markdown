---
layout: post
title:  "int Date vs struct Date: A C++ Benchmark"
date:   2026-03-09 00:00:00 -0500
categories: cpp
---

You just joined some finance company interested in performance.  As you work on your first project, you find a need to do operations on a Date object.  Unfortunately for you, it is stored as int:  20260309.  However, this is not just a weird legacy decision.  This is a matter of performance.

The full project is at [int-date-vs-struct-date](https://github.com/jfreun123/int-date-vs-struct-date/tree/main).

## The Three Representations

```cpp
// V1: three separate ints (12 bytes)
struct Date { int year, month, day; };

// V2: packed struct (4 bytes)
struct Date { short year; char month; char day; };

// V3: single integer (4 bytes)
using Date = int; // yyyymmdd, e.g. 20260309
```

V1 is the most readable but uses 12 bytes.  V2 matches V3's size while keeping named fields.  V3 discards readability for a single integer that can be compared, sorted, and serialized with minimal work.

## Benchmark Results

Machine: Apple M-series, clang++ `-O3 -march=native -funroll-loops -flto=thin`.

```
Benchmark                          Time             CPU   Iterations
--------------------------------------------------------------------
BM_GetNextWeekday_V1            10.2 ns         10.1 ns     65559645
BM_GetNextWeekday_V2            15.3 ns         15.0 ns     45117337
BM_GetNextWeekday_V3            14.3 ns         14.1 ns     49427349
BM_Sort_V1                    155007 ns       154298 ns         4192
BM_Sort_V2                    139792 ns       139201 ns         4905
BM_Sort_V3                     61464 ns        61147 ns        11130
BM_Serialize_V1                0.551 ns        0.548 ns   1213571193
BM_Serialize_V2                0.618 ns        0.615 ns   1110705615
BM_Serialize_V3                0.271 ns        0.270 ns   2562028541
BM_Deserialize_V1               1.73 ns         1.72 ns    398855853
BM_Deserialize_V2               1.69 ns         1.68 ns    403996099
BM_Deserialize_V3              0.272 ns        0.271 ns   2554548739
BM_Serialize_V3_CharPtr         68.2 ns         67.9 ns      9983883
BM_Deserialize_V3_CharPtr       27.8 ns         27.7 ns     25041856
```

| Operation | Winner | Ratio | Reason |
|---|---|---|---|
| `getNextWeekday` | V1 struct | 1.4× faster than V3 | V3 pays div/mod to unpack; V1 accesses fields directly |
| Sort (~2200 dates) | V3 int | 2.5× faster than V1 | Single integer compare; 3× smaller element (4 vs 12 bytes) |
| Binary serialize | V3 int | 2× faster than V1/V2 | V3 IS the wire int — one store, zero arithmetic |
| Binary deserialize | V3 int | 6× faster than V1/V2 | V3 IS the wire int — one load; V1/V2 must also unpack fields |

## Why Each Result Makes Sense

**Calendar arithmetic favors structs.**  `getNextWeekday` needs year, month, and day as separate values.  V1 reads them directly.  V3 unpacks first:

```cpp
constexpr int year(int date)  { return date / 10000; }
constexpr int month(int date) { return (date / 100) % 100; }
constexpr int day(int date)   { return date % 100; }
```

Division and modulo aren't free.  At 10.2 ns vs 14.3 ns, V1 beats V3 by a meaningful margin on a tight loop.

**Sorting favors the int.**  V3 compares two 4-byte integers in a single instruction.  V1 compares three `int` fields lexicographically — up to three comparisons.  V1 elements are also 12 bytes each vs 4 bytes for V3, so its array is 3× larger: more cache pressure, more memory bandwidth.  Result: 2.5× speed difference.

**Binary serialization favors the int.**  The wire format in binary protocols (FIX Binary, Cap'n Proto, memory-mapped files) is a 32-bit integer in yyyymmdd encoding.  V3 *is* that integer — serialize is a single `memcpy`:

```cpp
// V3: one store
std::memcpy(buf.data(), &d, sizeof(d));

// V1: pack fields first, then store
int wire = d.year * 10000 + d.month * 100 + d.day;
std::memcpy(buf.data(), &wire, sizeof(wire));
```

**Binary deserialization favors the int even more.**  On read, V3 is again one `memcpy`.  V1 must also decompose:

```cpp
// V3: one load
date_v3::Date d;
std::memcpy(&d, buf.data(), sizeof(d));

// V1: load, then unpack
int w;
std::memcpy(&w, buf.data(), sizeof(w));
date_v1::Date d{w / 10000, (w / 100) % 100, w % 100};
```

V1 does everything V3 does, then pays three more div/mod operations.  The gap opens to 6×: 0.272 ns vs 1.73 ns.

**Claude's Initial Mistake:  Char\* serialization/ deserialization.**  Some older systems serialize dates as their decimal text representation (`"20260309"`) into a char buffer and parse with `sscanf`/`atoi` on read.  For whatever reason, when first doing research with Claude, claude decided that this was the most optimal way to do things-- this is insane.

```
BM_Serialize_V3               0.271 ns   ← binary memcpy
BM_Serialize_V3_CharPtr        68.2 ns   ← snprintf
```

`snprintf` carries locale handling, format parsing, and digit formatting overhead — 250× slower than a 4-byte store.  The read side is similarly awful:

```
BM_Deserialize_V3             0.272 ns   ← binary memcpy
BM_Deserialize_V3_CharPtr      27.8 ns   ← atoi
```

100× slower than a load, for the same information.  In a world of binary encodings, why would you ever use a wasteful, large, and awkward Char\*.  This mishap is yet another example of how, though AI can be useful for boilerplate, it can make devastating performance mistakes.

(As an aside:  there are other ways to make a Char\* encoding faster.  However, these tricks will never approach the sub-nanosecond level of memcpy and will bring on a significant amount of code bloat.  Rather than focusing on optimizing encoding Char\*, it is best just to use a binary protocol.)

## Why Finance Uses Int Dates

The hot path is sort, compare, and binary I/O — all V3 wins.  Calendar arithmetic (`getNextWeekday`, `nextDay`) is computed once at startup into a holiday calendar lookup table, so its overhead is amortized away.  This optimization is significant precisely because it is *not* strictly better in every use case — just the cases that matter most.

## Resources:

- [int-date-vs-struct-date on GitHub](https://github.com/jfreun123/int-date-vs-struct-date/tree/main)
