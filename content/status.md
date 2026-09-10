+++
title = "Status"
description = "What BHC compiles today — concrete numbers, updated as the work lands."
template = "page.html"
+++

# Status

This page is the load-bearing answer to "how far along is BHC, really?"
It is updated when the numbers move. The [roadmap](@/roadmap.md) covers the
longer arc; this page covers what is true right now.

**Last updated:** 2026-09-10 · **Release:** bhc v0.2.22 ([install](@/_index.md#try-it-now))

## Headline numbers

| Surface | Today | Detail |
|---|---|---|
| Pandoc — `bhc check` | **221 / 221** modules pass | 0 fail, 0 skipped, Template Haskell included (2026-08-04) |
| Pandoc — compile to native | **221 / 221** modules | every module produces a real object file; a `pandoc` binary links and runs (2026-09-02) |
| Pandoc — run | **not yet** | `pandoc --version` crashes in the option parser; the Markdown reader is never reached |
| Pandoc — parse | **221 / 221** modules | the parser sees the whole tree |
| Zentinel agent (`zentinel-agent-policy`) | **10 / 10** modules compile | canonical real-library target |
| Native compilation E2E | **6 / 6** | hello, arithmetic, fibonacci, IO, recursion |
| Workspace unit tests | **2,756 passing** | 0 failing (2026-07-23) |
| Conformance milestones | **70** (E.1 – E.70) | 175 native E2E tests |

## Pandoc, by stage

[Pandoc](https://pandoc.org) is BHC's north-star integration target — ~60 kLOC
of real-world Haskell with ~80 transitive package dependencies. The numbers
above don't compress into a single percentage, because Pandoc fails at
different stages for different reasons:

- **Parse: 221 / 221.** Every source file in `pandoc-3.6.4/src/` produces a
  syntactically valid AST. This was the first milestone we shipped and it has
  stayed solid through every parser change since.
- **`bhc check`: 221 / 221.** Zero failed, zero skipped, Template Haskell
  included. The story of the endgame is in
  [BHC Type-Checks All of Pandoc](@/blog/bhc-type-checks-all-of-pandoc.md).
- **Compile to native: 221 / 221.** Every module produces a real object file
  and a `pandoc` binary links and runs. See
  [Compiling Pandoc](https://arcanist.sh/blog/compiling-pandoc/).
- **Run: not yet.** Ask the binary for Markdown to HTML and it crashes in
  the option parser before the reader is reached. The runtime failures so
  far, each deeper than the last, are catalogued in
  [The Crash Ladder](@/blog/the-crash-ladder.md).

The detailed Pandoc tracking document lives in the
[repo](https://github.com/arcanist-sh/bhc/blob/main/.claude/TODO-pandoc.md)
along with categorised failure breakdowns.

## Backends

| Backend | Status | Notes |
|---|---|---|
| Native (LLVM) | ✅ Working | All E2E tests pass; this is the supported path |
| WebAssembly (WASI) | 🟢 ~95% | Real programs run in `wasmtime`; 236 / 243 differential fixtures byte-identical to native; file IO host-backed via WASI. Missing: the `Handle` API and `System.Directory` |
| Metal (Apple silicon) | 🟢 Runs on hardware | Runtime compiles Metal Shading Language at run time and executes on the on-board GPU; on-GPU tests pass on M-series Macs. macOS only, behind the `metal` build feature; no `--target` flag yet |
| CUDA (PTX) | 🟡 80% | 2 / 2 mock tests pass; real hardware testing pending |
| ROCm (AMDGCN) | 🟡 60% | Structure complete; needs hardware |

## Profiles

| Profile | Status |
|---|---|
| `default` | 🟡 Lazy evaluation, the everyday path. The collector is not yet on the compiled-code path: compiled programs leak, which is fine for batch work and not for long-running servers |
| `numeric` | 🟡 Tensor IR, fusion, vectorisation, parallelisation all pass internal tests; not yet realised in native codegen, so a numeric build runs at default-profile speed |
| `server` | 🟡 Structured concurrency, STM, cancellation, deadlines — 35+ RTS tests pass; not yet wired to compiled Haskell |
| `edge` | 🟡 Minimal-runtime variant ready, no end-to-end deployment yet |
| `realtime` | 🟡 Incremental GC with pause measurement; needs a game-loop demo |
| `embedded` | 🟡 No-GC mode and static allocator land; bare-metal codegen deferred |

## Tooling

| Tool | Status |
|---|---|
| `bhc` (compiler driver) | ✅ Native works; `check`, `build`, `run`, `--dump-ir=core/tensor/loop`, `--kernel-report` |
| [`hx`](https://arcanist.sh/hx/) (build / package) | ✅ v0.9.2 — `hx build/run/test --backend bhc --native` with dependencies, BHC platform install, doctor checks |
| `bhci` (REPL) | 🟡 Compiles and parses; evaluation is stubbed |
| `bhc-lsp` | 🟡 Code present, not independently verified |
| `bhi` (IR inspector) | 🟡 Compiles; needs integration tests |

## What changed recently

- **2026-09-02** — [Compiling Pandoc](https://arcanist.sh/blog/compiling-pandoc/) — all 221 modules compile to native object files; a `pandoc` binary links and runs. It cannot convert a document yet.
- **2026-08-20** — [The Crash Ladder](@/blog/the-crash-ladder.md) — BHC-compiled Pandoc stopped failing at compile time and started failing at runtime; six crashes, each deeper into the Markdown reader.
- **2026-08-04** — [BHC Type-Checks All of Pandoc](@/blog/bhc-type-checks-all-of-pandoc.md) — `bhc check` passes 221 / 221, zero failed, zero skipped.
- **2026-07-02** — [BHC runs on WebAssembly](@/blog/bhc-runs-on-webassembly.md) — the WASM backend runs real programs, 236 / 243 differential fixtures byte-identical to native.
- **2026-05-25** — Pandoc check 77 → 78 modules. Four parser correctness fixes
  landed: backtick precedence in chained infix, qualified constructors at the
  start of pattern bindings, the layout rule's virtual semicolon between
  multi-line documentation comments, and several missing stubs for `Djot.AST`,
  `Text.DocLayout`, and `Skylighting`. None of these were responsible for the
  module count alone, but each closed a class of silent AST corruption that
  was leaking into multiple readers and writers.
- **2026-05-22** — [BHC and the Verification Camp](@/blog/bhc-and-the-verification-camp.md) — positioning post.
- **2026-05-04** — [All ten zentinel modules compile](@/blog/zentinel-ten-modules-compile.md) — the agent canonical target reaches `bhc check` parity with `ghc check`.

## Caveats

A few things to set expectations honestly:

- "Compiles" now means real object files for all of Pandoc, and a binary
  that links. *Running* Pandoc is the current horizon: the binary crashes
  in its option parser before any reader is reached.
- The garbage collector exists as a tested module but is not yet on the
  compiled-code path. Compiled programs leak. That is adequate for
  short-lived batch work and not for a server.
- Standard library coverage is partial. Where BHC stubs an external
  package, modules can pass the lowering pass and still fail later when
  type-class constraints can't be discharged against the stub.
- WASM runs real programs and is differential-tested against native. On
  GPU, Metal runs on Apple hardware; CUDA and ROCm are mock-validated only.
- The numeric profile's guaranteed fusion is a design contract that native
  code generation does not yet honour: a numeric build currently runs at
  default-profile speed.

If you're evaluating BHC for a specific project, the most useful thing you
can do is run `bhc check` against your code and
[open an issue](https://github.com/arcanist-sh/bhc/issues) with what fails.
The pandoc number moves precisely because real codebases keep exposing
specific gaps.
