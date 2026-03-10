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

V1 is the most readable but most naive-- it takes 12 bytes.  V2 tries to match V3's size while keeping named fields-- 4 bytes is pretty much as small as you could get.  V3 throws readability away in exchange for a single integer that can be compared, sorted, and serialized with minimal work.

## Benchmark Results

Machine: Apple M-series, clang++ `-O3 -march=native -funroll-loops -flto=thin`.

```
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
BM_GetNextWeekday_V1       11.3 ns         11.2 ns     59616580
BM_GetNextWeekday_V2       15.4 ns         15.2 ns     44145934
BM_GetNextWeekday_V3       14.1 ns         14.0 ns     49997500
BM_Sort_V1               146446 ns       145531 ns         4728
BM_Sort_V2               145543 ns       145007 ns         4829
BM_Sort_V3                61343 ns        61140 ns        10979
BM_Serialize_V1             138 ns          137 ns      5041920
BM_Serialize_V2             139 ns          138 ns      5014291
BM_Serialize_V3            70.4 ns         70.0 ns      9901270
BM_Deserialize_V1          90.6 ns         90.3 ns      7664513
BM_Deserialize_V2          89.7 ns         89.5 ns      7647515
BM_Deserialize_V3          88.7 ns         88.3 ns      7943082
```

| Operation | Winner | Ratio | Reason |
|---|---|---|---|
| `getNextWeekday` | V1 struct | 1.3x faster than V3 | V3 pays div/mod to unpack; V1 accesses fields directly |
| Sort (~2200 dates) | V3 int | 2.4x faster than V1/V2 | Single integer compare; 3x smaller element (4 vs 12 bytes) |
| Serialize to `"YYYYMMDD"` | V3 int | 2x faster | One `snprintf("%08d")` vs three format specifiers |
| Deserialize from `"YYYYMMDD"` | Tied | ~1 ns | `sscanf` dominates; unpacking arithmetic is noise |

## Why Each Result Makes Sense

**Calendar arithmetic favors structs.**  Computing `getNextWeekday` requires knowing the year, month, and day as separate values.  V1 reads them directly from named fields.  V3 has to unpack first:

```cpp
constexpr int year(int date)  { return date / 10000; }
constexpr int month(int date) { return (date / 100) % 100; }
constexpr int day(int date)   { return date % 100; }
```

Division and modulo are cheap but not free-- typically 30 to 50 cycles compared to a memory load's 4 cycles (L1 cache hit).  At 11 ns vs 14 ns, V1 beats V3 by a meaningful margin on a tight loop.

**Sorting favors the int.**  `std::sort` calls the comparator millions of times over ~2200 dates.  V3 compares two 4-byte integers in a single instruction.  V1 compares three `int` fields with a lexicographic `operator<=>` — three comparisons in the worst case.  V1 elements are also 12 bytes each vs 4 bytes for V3, so V1's array is 3x larger: more cache pressure, more memory bandwidth.  The result is a 2.4x speed difference.

**Serialization favors the int.**  Formatting `20260309` as `"20260309"` is one `snprintf` call:

```cpp
// V3
std::snprintf(buf, sizeof(buf), "%08d", d);  // one call

// V1
std::snprintf(buf, sizeof(buf), "%04d%02d%02d", d.year, d.month, d.day);  // three format conversions
```

The single conversion wins 2x.

**Deserialization is a wash.**  All three variants call `sscanf` on the same 8-character string.  Parsing dominates; the cost of unpacking the integer or filling struct fields is negligible.

## Why Finance Uses Int Dates

The hot path in most financial systems is: load a batch of records, sort them by date, look up dates in a table, serialize to a wire format or file.  The int representation wins every one of those operations.

The one thing it loses — calendar arithmetic — is comparatively a rare operation.  This optimization is so significant because it is *not* strictly better in every use case-- just the cases that we care about the most.

## The Integer Encoding Is Not a Coincidence

`yyyymmdd` as an integer has a useful property: the natural integer ordering matches chronological order.  `20260101 < 20260309 < 20261231` is true both arithmetically and calendrically.  That's why you can sort a list of int dates with a plain `std::sort` and get the right answer — no custom comparator needed.

If the encoding were `ddmmyyyy` or any other arrangement, sorting by integer value would give the wrong order.  The `yyyymmdd` layout is a deliberate choice.

## Resources:

- [int-date-vs-struct-date on GitHub](https://github.com/jfreun123/int-date-vs-struct-date/tree/main)
