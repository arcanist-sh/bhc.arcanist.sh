+++
title = "Status"
description = "What BHC compiles today — concrete numbers, updated as the work lands."
template = "page.html"
+++

# Status

This page is the load-bearing answer to "how far along is BHC, really?"
It is updated when the numbers move. The [roadmap](@/roadmap.md) covers the
longer arc; this page covers what is true right now.

**Last updated:** 2026-05-25 · **Release:** bhc v0.2.1 ([install](@/_index.md#try-it-now))

## Headline numbers

| Surface | Today | Detail |
|---|---|---|
| Pandoc — `bhc check` | **78 / 221** modules pass | 91 fail, 52 skipped (external deps) |
| Pandoc — parse | **221 / 221** modules | the parser sees the whole tree |
| Zentinel agent (`zentinel-agent-policy`) | **10 / 10** modules compile | canonical real-library target |
| Native compilation E2E | **6 / 6** | hello, arithmetic, fibonacci, IO, recursion |
| Workspace unit tests | **217 passing** | 9 known interpreter IO-capture failures |
| Conformance milestones | **70** (E.1 – E.70) | 175 native E2E tests |

## Pandoc, by stage

[Pandoc](https://pandoc.org) is BHC's north-star integration target — ~60 kLOC
of real-world Haskell with ~80 transitive package dependencies. The numbers
above don't compress into a single percentage, because Pandoc fails at
different stages for different reasons:

- **Parse: 221 / 221.** Every source file in `pandoc-3.6.4/src/` produces a
  syntactically valid AST. This was the first milestone we shipped and it has
  stayed solid through every parser change since.
- **`bhc check`: 78 / 221.** The 91 failing modules split roughly 1:1 between
  remaining lowering issues (stub coverage, scoping, layout edge cases) and
  type-check failures (the standard library stubs are typed thinly and don't
  always satisfy class constraints like `PandocMonad m =>`).
- **Compile to native: not measured at module count yet.** The integration
  target is end-to-end Pandoc CLI; we're still in the check phase.

The detailed Pandoc tracking document lives in the
[repo](https://github.com/arcanist-sh/bhc/blob/main/.claude/TODO-pandoc.md)
along with categorised failure breakdowns.

## Backends

| Backend | Status | Notes |
|---|---|---|
| Native (LLVM) | ✅ Working | All E2E tests pass; this is the supported path |
| WebAssembly (WASI) | 🟡 70% | Binaries validate against `wasmtime`; 0 / 6 output tests because Core IR → WASM lowering is incomplete |
| CUDA (PTX) | 🟡 80% | 2 / 2 mock tests pass; real hardware testing pending |
| ROCm (AMDGCN) | 🟡 60% | Structure complete; needs hardware |

## Profiles

| Profile | Status |
|---|---|
| `default` | ✅ Lazy evaluation, generational GC, the everyday path |
| `numeric` | ✅ Tensor IR, fusion, vectorisation, parallelisation all pass internal tests |
| `server` | ✅ Structured concurrency, STM, cancellation, deadlines — 35+ tests pass |
| `edge` | 🟡 Minimal-runtime variant ready, no end-to-end deployment yet |
| `realtime` | 🟡 Incremental GC with pause measurement; needs a game-loop demo |
| `embedded` | 🟡 No-GC mode and static allocator land; bare-metal codegen deferred |

## Tooling

| Tool | Status |
|---|---|
| `bhc` (compiler driver) | ✅ Native works; `check`, `build`, `run`, `--dump-ir=core/tensor/loop`, `--kernel-report` |
| [`hx`](https://arcanist.sh/hx/) (build / package) | ✅ v0.6.0 — auto-detect backend, BHC platform install, doctor checks |
| `bhci` (REPL) | 🟡 Compiles and parses; evaluation is stubbed |
| `bhc-lsp` | 🟡 Code present, not independently verified |
| `bhi` (IR inspector) | 🟡 Compiles; needs integration tests |

## What changed recently

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

- "Compiles" means `bhc check` in most rows above. Native code generation
  for full Pandoc (or zentinel-agent-policy linked end-to-end with the
  proxy) is the *next* horizon, not the current one.
- Standard library coverage is partial. Where BHC stubs an external
  package, modules can pass the lowering pass and still fail later when
  type-class constraints can't be discharged against the stub.
- WASM and GPU work has substantial code but no working end-to-end demo
  yet. The numbers reflect that honestly.

If you're evaluating BHC for a specific project, the most useful thing you
can do is run `bhc check` against your code and
[open an issue](https://github.com/arcanist-sh/bhc/issues) with what fails.
The pandoc number moves precisely because real codebases keep exposing
specific gaps.
