+++
title = "All Ten Zentinel Modules Compile"
description = "BHC's multi-module pipeline now compiles all ten modules of zentinel-agent-policy to object code in one driver invocation. Cycles, qualified-name shadowing, and a missing IORef primitive — what changed between four out of ten and ten out of ten."
date = 2026-05-04
template = "blog-post.html"

[extra]
tag = "milestone"
+++

A quick refresher on the test subject. [Zentinel](https://zentinelproxy.io) is a reverse proxy on Cloudflare Pingora whose dataplane is intentionally small — security, policy, and inspection logic all live in **agents**: out-of-process programs the proxy talks to over Unix sockets. The agent architecture is by design: anything parsing-heavy or policy-rich runs behind a process boundary, so a misbehaving extension can't take the proxy down. There are 25-plus agents in the Zentinel ecosystem, written across seven SDK languages.

[zentinel-agent-policy](https://github.com/zentinelproxy/zentinel-agent-policy) is the Haskell agent. It's the canonical Haskell-as-glue program: it implements the agent protocol, maps incoming requests into a `PolicyInput`, owns an LRU decision cache, handles configuration and audit logging, and shells out to the [`cedar`](https://www.cedarpolicy.com/) and [`opa`](https://www.openpolicyagent.org/) CLI tools to do the actual policy evaluation. The Haskell code is the orchestration layer, not the policy engine. Which is exactly the shape of code that real Haskell services run in production: configuration-driven IO, `IORef` for shared cache state, `forkIO` per connection, `System.Process` for subprocess calls, and sum types for protocol messages.

A few weeks ago we wrote about [running it on BHC](/bhc/blog/bhc-runs-a-real-haskell-library/) — compiling records and sum types from two of its modules and watching the resulting binary print correct values for every test case in our harness. That story was about runtime correctness: did the bytes in memory match what the source code described.

This post is the compile-time counterpart. By the time that runtime-correctness post went out, all ten of zentinel-agent-policy's modules were going through BHC's full multi-module pipeline and emitting object code. Not stubs. Not a subset of the library. The whole `Zentinel.Agent.Policy.*` graph, processed in one driver invocation, with a shared module registry, producing one `.o` file per module.

The work to get from four out of ten to ten out of ten took about two days. Three things had to land.

## The shape of the graph

zentinel-agent-policy is ten files, but its module graph isn't a tree. Two of the modules form a cycle: one defines a typeclass that the other parameterizes a cache type with, and the cache type's signatures appear in the typeclass's default methods. The cycle is small, but it's enough that no topological order over the modules exists, so processing them one at a time in dependency order — bailing out the moment an import points at a module we haven't seen — doesn't work.

GHC handles this by computing strongly connected components on the module graph and type-checking each SCC as a unit, with all of its members visible to each other during inference. Until early March, BHC's loader walked the import graph eagerly and refused to proceed when it found a back-edge. Cycles became "circular dependency, give up."

The fix is what you'd expect: walk the graph, find SCCs with Tarjan's algorithm, and process each SCC as a single compilation unit. The implementation choice that mattered was *what "process as a unit" means in practice*. We kept the per-module type-check pass and pre-registered each SCC member's type signatures in a shared environment before any module's bodies are checked in detail. Mutual recursion at the value level still flows through the standard `let`-binding generalization machinery; the new piece is just that signatures from across module boundaries are visible during inference of the bodies that use them.

This unblocked the cycle and pushed the count from four modules to five (`ff386da`).

## Qualified names that don't fall back to Prelude

The next blocker was a class of bugs that all looked the same from the outside: a qualified function call from one zentinel module would type-check against the wrong type, and the error messages would point at signatures from `Prelude`.

Concrete example. `Cache.hs` imports `Data.Map` qualified and calls `Cache.lookup` from another module that re-exports a wrapper. The type checker would see `Cache.lookup`, fail to find a binding for that exact qualified path in the local environment, and fall back to its builtins table. There is a `Prelude.lookup :: Eq a => a -> [(a, b)] -> Maybe b`. The fallback would resolve `Cache.lookup` to that, and the rest of the body — passing it a `Map`, expecting a `Decision` back — would unify against `[(a, b)]` and produce a cascade of errors about phantom types.

The bug was in two places. First, the resolver. When `register_imported_names` (in `crates/bhc-lower/src/loader.rs`) saw a qualified import, it threaded the imported names through a `qualified_names` map but didn't bind them as values in the type environment. Lookup by qualified path went through `qualified_names` first, and if anything failed there, the resolver fell through to the unqualified builtin lookup. Second, the type checker had no representation-level distinction between "imported value" and "builtin" — both lived in the same `DefId` space, with no way at use-site to ask "is this an import I should respect, or a builtin I should generalize freshly?"

Three changes (`43d4faa`):

1. Bind qualified names directly as values in the type environment when the import is registered, so `resolve_qualified_var` finds the right `DefId` immediately.
2. Add `DefKind::ImportedValue` to mark cross-module imports distinctly from builtins, and gate the builtin fallback in `resolve_qualified_var` on whether the qualifier corresponds to a known import alias. If you wrote `Cache.lookup`, you don't want `Prelude.lookup`. If the import alias is genuinely unknown, error out cleanly instead of silently substituting.
3. Give cross-module imports whose types BHC doesn't yet know — because the source isn't in the compilation set — fresh polymorphic schemes per use site, instead of monomorphic ones. Each call site picks its own instantiation.

We also reordered constraint solving: `Fractional` is now solved before `Num`, which prevents premature `Int` defaulting in expressions where a value flows into a `Fractional`-only context. Without the reorder, `Num a` would resolve to `Int` and the next step would fail the `Fractional Int` lookup.

After these changes, all ten modules cleared `bhc check`.

## A missing IORef primitive

`Zentinel.Agent.Policy.Handler` uses `atomicModifyIORef'` to update an LRU cache that the request handler shares with worker threads. It's in `Data.IORef` and has the type:

```haskell
atomicModifyIORef' :: IORef a -> (a -> (a, b)) -> IO b
```

BHC's type checker had `IORef`, `newIORef`, `readIORef`, and `writeIORef` as builtins, but neither of the atomic variants. Wherever `atomicModifyIORef'` appeared, the checker would either reject the use outright or fall through to a stub with a wrong type that broke seven separate inference attempts in `Handler.hs`.

We added the two atomic primitives as builtins (`DefId`s 10405 and 10406) with their correct types, and implemented codegen as a straight sequence: read the current value, apply the function pointer, extract the tuple fields, write `fst` back, return `snd`. There's no atomic instruction at this layer yet — the codegen has the right shape but is single-threaded — and that's intentional. Wiring it to a real compare-and-swap path is a runtime question, not a type-checker one. The signature is correct, the call site compiles, and the runtime can be sharpened later without re-touching the front end.

This was the gap between "9 of 10 modules emit `.o`" and "10 of 10 modules emit `.o`."

## Compile-only mode using the same pipeline

The last fix is the most boring of the four and had the largest behavioral impact. BHC had two driver paths: full compilation, which used `compile_files_ordered()` with a shared `ModuleRegistry`, and compile-only mode (`bhc -c`), which iterated through files calling `compile_module_only()` on each one independently.

The independent-files path didn't share a registry. So when `Cache.hs` in compile-only mode tried to update a record imported from `Config.hs`, it didn't have the field information for the imported constructor — that lived in the registry the compile-only path wasn't using. The result was spurious "field not in scope" errors that didn't appear in full compilation, on the same source files.

The fix was to make `-c` use the same multi-module pipeline as `-o`, just stopping after object emission instead of going on to link. One driver code path, two stop points (`617eafa`, `crates/bhc-driver/src/lib.rs`). After the change, `bhc -c Zentinel/Agent/Policy/*.hs` produces ten clean `.o` files in one invocation.

## By the numbers

| Metric | March 9 | March 11 evening |
|---|---|---|
| zentinel modules clearing `bhc check` | 4 | 10 |
| zentinel modules emitting `.o` | 0 | 10 |
| Cross-module qualified-name errors | 17 | 0 |
| Cycle-related compilation failures | 1 | 0 |

## Honest about the gap

Compile-only is a smaller claim than runtime-correct. The `.o` files are real machine code, but linking them into a single runnable binary still depends on stub Hackage packages — `aeson`, `yaml`, `optparse-applicative` — that don't have BHC implementations yet. Anything that calls into those stubs would trap at runtime. The two modules in the [previous post's runtime test](/bhc/blog/bhc-runs-a-real-haskell-library/) sit on the no-Hackage side of that line, which is why we could exercise them end-to-end. The other eight live on the other side until BHC compiles their dependencies.

So this is a partial milestone, scoped honestly. The Haskell compilation pipeline — parsing, type checking, lowering, Core IR, LLVM codegen — runs cleanly across the full ten-module graph of a real library. The remaining mile to a single linked binary is external package coverage, and that's a separate project: it's about compiling Hackage, not about improving BHC.

## What it unlocks elsewhere

The same machinery generalizes. `bhc check` now clears 76 of Pandoc's 221 modules, up from 53 when [the parses-all-of-pandoc post](/bhc/blog/bhc-parses-all-of-pandoc/) went out. The cross-module type fixes that unblocked zentinel are the same ones moving Pandoc's count up, with cycles in modules like `Text.Pandoc.Class` and `Text.Pandoc.Logging` that look like the zentinel cycle at a slightly larger scale.

Multi-module compilation is one of those features that doesn't have a satisfying single-test demonstration. You ship it by pointing the compiler at libraries with real graphs and watching what doesn't break. zentinel-agent-policy is the smallest non-trivial library we have access to, so it gets the small-graph runs. Pandoc is the next size up, and it's where most of the daily activity is now.

We'll keep going.

---

*BHC is open source at [github.com/arcanist-sh/bhc](https://github.com/arcanist-sh/bhc).*
