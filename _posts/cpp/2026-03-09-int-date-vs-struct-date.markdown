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
BM_GetNextWeekday_V1       10.6 ns         10.5 ns     67552570
BM_GetNextWeekday_V2       19.7 ns         18.4 ns     41524544
BM_GetNextWeekday_V3       16.3 ns         14.4 ns     45893213
BM_Sort_V1               164997 ns       162007 ns         4444
BM_Sort_V2               145696 ns       143983 ns         4844
BM_Sort_V3                65616 ns        64115 ns        10872
BM_Serialize_V1             5.65 ns         5.57 ns    122097992
BM_Serialize_V2             6.02 ns         6.00 ns    115612665
BM_Serialize_V3             8.71 ns         8.67 ns     80073210
BM_Deserialize_V1           4.58 ns         4.51 ns    161837363
BM_Deserialize_V2           4.29 ns         4.27 ns    162654900
BM_Deserialize_V3           1.92 ns         1.91 ns    369595979
```

| Operation | Winner | Ratio | Reason |
|---|---|---|---|
| `getNextWeekday` | V1 struct | 1.5x faster than V3 | V3 pays div/mod to unpack; V1 accesses fields directly |
| Sort (~2200 dates) | V3 int | 2.5x faster than V1 | Single integer compare; 3x smaller element (4 vs 12 bytes) |
| Serialize to `"YYYYMMDD"` | V1/V2 struct | 1.5x faster than V3 | Fields already split — no decomposition needed |
| Deserialize from `"YYYYMMDD"` | V3 int | 2.4x faster than V1/V2 | Parsed integer is immediately a valid date; V1/V2 must then split it |

## Why Each Result Makes Sense

**Calendar arithmetic favors structs.**  Computing `getNextWeekday` requires knowing the year, month, and day as separate values.  V1 reads them directly from named fields.  V3 has to unpack first:

```cpp
constexpr int year(int date)  { return date / 10000; }
constexpr int month(int date) { return (date / 100) % 100; }
constexpr int day(int date)   { return date % 100; }
```

Division and modulo are cheap but not free-- typically 30 to 50 cycles compared to a memory load's 4 cycles (L1 cache hit).  At 10.6 ns vs 16.3 ns, V1 beats V3 by a meaningful margin on a tight loop.

**Sorting favors the int.**  `std::sort` calls the comparator millions of times over ~2200 dates.  V3 compares two 4-byte integers in a single instruction.  V1 compares three `int` fields with a lexicographic `operator<=>` — three comparisons in the worst case.  V1 elements are also 12 bytes each vs 4 bytes for V3, so V1's array is 3x larger: more cache pressure, more memory bandwidth.  The result is a 2.5x speed difference.

**Serialization favors structs.**  The benchmark uses manual digit extraction instead of `snprintf`.  V1's fields are already split — writing each digit is a few divisions on small values (year, month, day separately).  V3 has to extract all 8 digits from one packed integer in sequence:

```cpp
// V3: must unpack all 8 digits from the packed int
buf[7] = '0' + tmp % 10; tmp /= 10;
buf[6] = '0' + tmp % 10; tmp /= 10;
// ... 6 more times

// V1: fields are already separate, work directly from d.year, d.month, d.day
buf[0] = '0' + d.year / 1000;
buf[1] = '0' + (d.year / 100) % 10;
// ...
```

This is the symmetric opposite of `getNextWeekday`: when your input is already decomposed into fields, V1/V2 win.  V3 pays to unpack.

**Deserialization favors the int.**  Parsing `"20260309"` into digits and accumulating an integer gives you a valid V3 date immediately — the parsed value *is* the date:

```cpp
// V3: done after parsing
date_v3::Date d = (buf[0]-'0')*10000000 + (buf[1]-'0')*1000000 + ...;

// V1: parse the same integer, then decompose it
int n = (buf[0]-'0')*10000000 + ...;
date_v1::Date d{n / 10000, (n / 100) % 100, n % 100};  // extra div/mod
```

V1/V2 have to do everything V3 does, then split the result.  V3 wins by 2.4x.

## Why Finance Uses Int Dates

The hot path in most financial systems is: receive records from the wire (FIX, CSV), sort them by date, and look them up in a table.  The int representation wins all of those — deserialize, sort, compare.  Serialization goes to structs once you switch from `snprintf` to manual digit extraction, but writing to the wire is less common than reading from it.

The one thing it loses — calendar arithmetic — is comparatively a rare operation.  This optimization is so significant because it is *not* strictly better in every use case-- just the cases that we care about the most.

## Resources:

- [int-date-vs-struct-date on GitHub](https://github.com/jfreun123/int-date-vs-struct-date/tree/main)
