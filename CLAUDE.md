# CLAUDE.md

## Blog posts

- Jacob writes the first pass of every post section himself — posts must not be pure AI writing. Never draft new sections, prose, or output blocks into a post ahead of him; wait until his first pass exists, then fill in only what he asks for (explicit TODOs, command outputs, links, embeds).
- Claude's role is scaffolding and verification: capture real command outputs (actually compile/run everything before pasting), fix typos and markdown-rendering bugs, fact-check claims, build Compiler Explorer embeds.
- Code shown in posts lives in the cpp_study repo (https://github.com/jfreun123/cpp_study) under `blog/<series-name>/`, committed and pushed so post links resolve.
- Command outputs for the C++ series come from Linux/ELF via Docker (`gcc:15` image), not macOS — native `g++` here is Apple clang and produces Mach-O.
