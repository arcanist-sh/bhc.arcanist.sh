+++
title = "About BHC"
description = "About the Basel Haskell Compiler project"
template = "page.html"
+++

# About BHC

BHC (Basel Haskell Compiler) is an alternative compiler for Haskell, built around a simple premise: keep compatibility as the baseline, and make performance and deployment feel modern again.

## What BHC Is

BHC targets Haskell 2010 and selected GHC editions (for a documented subset of extensions), while introducing explicit runtime profiles for server latency, numeric workloads, and constrained targets. Innovations are opt-in and clearly namespaced (`BHC*`), so you can choose portability or specialization explicitly.

## Why Basel?

Because I'm from Basel and still live there. That's the real reason.

But it works on other levels too. Glasgow gave Haskell its compiler — thirty years of work, production deployments everywhere. Basel is another European city with its own history. The Bernoullis were from here. Euler worked here. There's a tradition of mathematics in this town that I grew up taking for granted.

So: a Haskell compiler from Basel, named after Basel. It's personal, not a marketing exercise.

## How We Got Here

BHC wasn't the first project. [hx](https://arcanist.sh/hx) came before it — a package manager and build tool for Haskell, also written in Rust. The idea was simple: Haskell's tooling situation is fragmented (cabal, stack, ghcup), and a unified tool could help.

Building hx meant understanding how Haskell projects are structured, how dependencies resolve, how builds should work. At some point the question became: if we're rethinking the tooling, why not rethink the compiler too?

So hx and BHC are siblings. hx handles packages, dependencies, and builds. BHC handles compilation. They're designed to work together, though BHC also works standalone with traditional cabal workflows.

## Project Goals

BHC exists to strengthen the Haskell ecosystem:

- We prioritize compatibility and portability
- We publish conformance tests and benchmarks
- We upstream proposals when ideas mature
- We collaborate rather than compete for identity

## Design Decisions

Some of these choices might be wrong. We won't know for years. But here's our reasoning.

### Why Rust?

The honest answer: we wanted a language with good tooling, no garbage collector in the compiler itself, and a type system that catches mistakes. Rust fits.

There's no deeper philosophy here. Writing a compiler in Haskell is elegant — GHC does it — but we didn't want to debug the compiler's memory behavior while also debugging the compiler's correctness. Rust lets us focus on one problem at a time.

It also means you don't need a Haskell compiler to build BHC, which makes bootstrapping easier. Small thing, but it matters for contributors.

### Why LLVM?

Because writing a code generator is a lot of work and we'd probably do it worse than the LLVM team.

LLVM handles register allocation, instruction selection, vectorization, and a dozen other things we'd rather not think about. GHC uses it (optionally), Rust uses it, Swift uses it. It's not exciting, but it works.

We do bypass LLVM for GPU code (CUDA/ROCm) and WebAssembly, where we wanted tighter control. But for native x86/ARM, LLVM saves us years of work.

### Why Profiles?

GHC has a lot of flags. You can spend hours tuning RTS options, optimization levels, and strictness settings. Sometimes you get faster code. Sometimes you get subtle bugs. Often you're not sure what you're doing.

Profiles are our attempt to make this less painful. Instead of "here are 200 knobs," we say "here are six configurations we've actually tested." Pick the one that matches your use case.

It's opinionated, which means we're sometimes wrong about what you need. But we'd rather ship tested combinations than untested flexibility.

There's a quieter reason too. A flag is something you set and forget; a profile is a contract you can name. Once the performance envelope a module was compiled under has a name — `server`, `numeric`, `edge` — it's something you can point at, record, and reason about, instead of a guess reconstructed from a pile of RTS options. We think that matters more as compilers do more of the optimizing: "optimized" is not one thing, and a profile is us saying out loud which thing we meant.

### Why a New Runtime?

Haskell doesn't require a specific runtime — that's an implementation detail. GHC's runtime is good, but it's one runtime trying to serve everyone.

We wanted different runtimes for different jobs. The server profile needs a work-stealing scheduler and cancellation. The numeric profile needs arena allocators and pinned memory. The realtime profile needs bounded GC pauses. The embedded profile needs no GC at all.

You can't really do that with one runtime and some flags. So we wrote multiple.

### Why Start From Scratch?

This is the decision we second-guess the most.

We could have forked GHC. We'd have years of work already done. We'd have compatibility from day one. It would have been the sensible choice.

But we wanted to use Rust (see above), and forking a Haskell codebase into Rust isn't really forking. We also wanted to rethink some architectural decisions — the IR pipeline, incremental compilation, the query system — and doing that in an existing codebase is harder than it sounds.

So we started over. It's slower. We're reimplementing things GHC solved decades ago. But we understand every line of code, and we can make changes without worrying about breaking twenty years of accumulated expectations.

Ask us in five years whether this was the right call.

## What BHC Is Not

BHC is not:

- A new language
- A fork of Haskell
- A replacement for GHC
- An attempt to define a new Haskell standard

## The Runtime

BHC includes its own runtime system (BHC RTS). Runtime systems can differ between compilers even when the surface language is the same. BHC RTS focuses on:

- Structured concurrency (cancellation, deadlines, scoping)
- Observability (events, tracing hooks)
- Profile-driven allocation strategies (arena/pinned where appropriate)
- Predictable scheduling behavior for server and numeric workloads

In compatibility modes, BHC aims to preserve Haskell semantics. Where runtime behavior is observably different, we document it and test it.

## Open Source

BHC is open source under the BSD-3-Clause license.

- [GitHub Repository](https://github.com/arcanist-sh/bhc)
- [Issue Tracker](https://github.com/arcanist-sh/bhc/issues)
- [Discussions](https://github.com/arcanist-sh/bhc/discussions)

## Community

Join the conversation:

- [Twitter / X](https://x.com/raskelll)
- [GitHub Discussions](https://github.com/arcanist-sh/bhc/discussions)

## Acknowledgments

BHC builds on decades of work by the Haskell community, the GHC team, and researchers in functional programming and compiler design. We are grateful for this foundation.

---

*BHC is an independent compiler implementation for Haskell. "Haskell" is a community language; BHC does not define the Haskell standard.*
