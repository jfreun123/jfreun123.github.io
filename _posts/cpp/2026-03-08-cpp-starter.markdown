---
layout: post
title:  "A Modern C++23 Project Template"
date:   2026-03-08 00:00:00 -0500
categories: cpp
---

Starting a new C++ project from scratch means making a lot of tooling decisions up front: build system, testing framework, static analysis, sanitizers, CI.  Get it wrong early and you're either fighting your build system later or flying blind without good diagnostics.  I put together [cpp-starter](https://github.com/jfreun123/cpp-starter) to solve this once.

## What's Included

- **CMake + Ninja** as the build system, targeting C++23
- **Google Test** for unit testing
- **clang-tidy** and **cppcheck** for static analysis
- **clang-format** for consistent formatting
- **AddressSanitizer (ASAN)** and **ThreadSanitizer (TSAN)** for runtime memory and concurrency checks
- **GitHub Actions** CI that runs tests and analysis on every push

## Project Structure

```
cpp-starter/
├── source/          # Header-only library
├── test/            # Google Test unit tests
├── benchmark/       # Benchmarks
├── cmake/           # CMake modules (lint, gtest, dev-mode)
├── CMakeLists.txt
└── CMakePresets.json
```

The layout follows [Pitchfork](https://github.com/vector-of-bool/pitchfork) conventions for standardized C/C++ projects.  The core argument: if most open source C++ projects share the same structure, you can orient yourself in an unfamiliar codebase almost immediately — no documentation required.

There's also a mechanical benefit.  A flat list of files means finding what you need is O(N).  A well-organized folder hierarchy turns that into O(log N) — you make a sequence of narrowing decisions and arrive at the right place much faster.  The folder structure acts as an index.

## CMake Presets

Four presets cover the main development modes:

| Preset | Purpose |
|--------|---------|
| `dev`  | Debug build, strict warnings, static analysis |
| `asan` | AddressSanitizer — catches heap overflows, use-after-free |
| `tsan` | ThreadSanitizer — catches data races |
| `bench`| RelWithDebInfo build for benchmarking |

The `dev` preset enables `-Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion -Wshadow`.  ASAN and TSAN run as separate presets since they can't be combined; both add `-fno-omit-frame-pointer` for readable stack traces.

## Static Analysis

`clang-tidy` and `cppcheck` are wired into the build targets via CMake target properties, so analysis runs automatically as part of every build — no separate step needed.

## Jason Turner and the Production vs. Learning Tradeoff

Jason Turner's [C++ Best Practices](https://lefticus.gitbooks.io/cpp-best-practices/content/02-Use_the_Tools_Available.html) is a thorough survey of every tool worth having: package managers, fuzz testers, mutation testers, coverage tools, multiple CI backends, Docker, WebAssembly builds.  His [cmake_template](https://github.com/cpp-best-practices/cmake_template) operationalizes all of it.

It's an excellent production template.  It's also a lot to absorb if your goal is to quickly explore a topic in C++.

`cpp-starter` is aimed at a different use case: *I want to learn about this topic and get up to speed without fully productionizing it*.  Fewer moving parts, less to configure, quicker to clone and start writing code.

That said, `clang-tidy`, `cppcheck`, and `clang-format` are non-negotiable even for learning projects.  Adding them later means cleaning up accumulated warnings and reformatting an entire codebase at once — a painful migration that's easy to avoid by starting with them wired in from the first commit.

Resources:

- [cpp-starter on GitHub](https://github.com/jfreun123/cpp-starter)
- [Pitchfork layout conventions](https://github.com/vector-of-bool/pitchfork)
- [Jason Turner's C++ Best Practices](https://lefticus.gitbooks.io/cpp-best-practices/content/02-Use_the_Tools_Available.html)
- [cmake_template (production-grade)](https://github.com/cpp-best-practices/cmake_template)
- [CMake Presets documentation](https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html)
- [Google Test](https://github.com/google/googletest)
- [Clang sanitizers overview](https://clang.llvm.org/docs/index.html)
