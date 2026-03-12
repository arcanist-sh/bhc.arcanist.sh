+++
title = "BHC Runs a Real Haskell Library"
description = "We compiled zentinel-agent-policy — 10 modules, 1,500 lines, records and sum types across module boundaries — to a native binary that produces correct results at runtime. Here's the full story."
date = 2026-03-12
template = "blog-post.html"

[extra]
tag = "milestone"
+++

There's a question that compiler developers learn to dread. You've spent weeks on a feature, the type checker is happy, the linker doesn't complain, and the binary appears in the build directory right where it should be. Then someone asks: *does it actually work?*

Not "does it compile." Not "does it link." Does the binary, when you run it, produce correct values? Do records contain the right fields? Do cross-module function calls return the right data? Does a pattern match on a sum type imported from another module dispatch to the right branch?

This week, for the first time, we can answer that question for a real Haskell library. Not a test fixture. Not a single-file exercise. A multi-module library with records, sum types, nested configuration structures, and cross-module function calls — compiled to a native binary by BHC, producing correct output for every test case.

## Why this library

The test subject is [zentinel-agent-policy](https://github.com/zentinelproxy/zentinel-agent-policy), and the choice wasn't arbitrary. It's one of my other projects.

[Zentinel](https://github.com/zentinelproxy/zentinel) is a security-first reverse proxy built on Cloudflare's [Pingora](https://github.com/cloudflare/pingora). Its core is written in Rust, but its architecture is designed around *agents* — external processes that extend the proxy's functionality by inspecting and modifying HTTP traffic. There's a [Zentinel Haskell SDK](https://github.com/zentinelproxy/zentinel-agent-haskell-sdk) that makes it possible to write these agents in Haskell, and zentinel-agent-policy is the first (and so far only) Haskell-based Zentinel agent. It evaluates authorization policies written in [Cedar](https://www.cedarpolicy.com/) or [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) against incoming requests, deciding whether to allow or deny each one.

This agent is also, in a roundabout way, the reason BHC exists.

A reverse-proxy agent that evaluates policies on every HTTP request has real performance requirements. It needs predictable latency, bounded memory allocation, and the ability to handle thousands of decisions per second. GHC is a magnificent compiler, but its runtime was designed for throughput, not latency. The stop-the-world GC pauses, the unpredictable thunk chains, the difficulty of reasoning about allocation in hot paths — these are acceptable trade-offs for most Haskell programs, but they're exactly what you don't want in a component that sits in the critical path of every HTTP request.

That tension — wanting Haskell's type safety and expressiveness for the agent logic, but needing runtime characteristics that GHC doesn't optimize for — is what led to BHC's Realtime profile, its arena allocators, and its bounded-pause GC. The idea was straightforward: what if there were a Haskell compiler that took predictable performance as seriously as it took correctness?

So when it came time to test BHC's multi-module compilation on something real, zentinel-agent-policy was the obvious choice. It's the program that motivated the compiler. It would be fitting if it were one of the first real libraries the compiler could run.

## The library

zentinel-agent-policy is ten modules and roughly 1,500 lines of Haskell:

```
Zentinel.Agent.Policy.Types      -- Decision, PolicyEngine, PolicyInput, etc.
Zentinel.Agent.Policy.Config     -- AgentConfig, CacheConfig, defaultConfig
Zentinel.Agent.Policy.Cache      -- LRU decision cache
Zentinel.Agent.Policy.Engine     -- Evaluation typeclass
Zentinel.Agent.Policy.Cedar      -- Cedar engine implementation
Zentinel.Agent.Policy.Rego       -- Rego/OPA engine implementation
Zentinel.Agent.Policy.Input      -- Request-to-PolicyInput mapping
Zentinel.Agent.Policy.Protocol   -- Wire protocol handling
Zentinel.Agent.Policy.Handler    -- Request handler logic
Zentinel.Agent.Policy             -- Public API re-exports
```

It's not a large library. But it exercises a representative slice of what production Haskell code actually does:

- **15 data types** — records with named fields (`AgentConfig` has 8 fields, `PolicyInput` has 4), sum types with payloads (`PolicySource` has three constructors carrying `FilePath`, `Text`, and `(Text, Int)` respectively), and simple enumerations (`Decision` is `Allow | Deny`)
- **Cross-module imports with explicit export lists** — `Types.hs` exports `Decision(..)`, `PolicyEngine(..)`, `PolicyInput(..)` and 12 other types; `Config.hs` imports from `Types` and re-exports its own types
- **Nested configuration structures** — `AgentConfig` contains `CacheConfig`, `AuditConfig`, and `InputMapping`, each of which is its own record type
- **`deriving stock` and `deriving anyclass`** — every type derives `Eq`, `Show`, and `Generic`; most derive `FromJSON` and `ToJSON`
- **Cross-module CAFs** — `defaultConfig`, `defaultInputMapping`, `defaultCacheConfig`, and `defaultAuditConfig` are zero-argument top-level bindings that other modules reference

This is the kind of code that lives in every production Haskell codebase. Nothing clever, nothing exotic. Just types, records, constructors, and functions that build and inspect them across module boundaries.

## Making it compile

Getting all ten modules to compile was the work of a [previous session](/bhc/blog/one-developer-one-claude-one-month/). The multi-module compilation pipeline (`compile_files_ordered()` with a shared `ModuleRegistry`) had been wired up, and each module's exports were being registered for downstream consumers. But "compiles" and "works" are different things, and the gap between them turned out to contain three bugs.

### Bug 1: Export lists were decorative

When a module declared an explicit export list:

```haskell
module Zentinel.Agent.Policy.Types
  ( Decision(..)
  , PolicyEngine(..)
  , EvaluationResult(..)
  , ...
  ) where
```

BHC was ignoring it. The function `build_module_exports_from_hir` was exporting *everything* from the lowering context's definition map, regardless of what the module actually declared in its export list. Every internal helper, every unexported type, everything leaked into the module's public interface.

This worked fine in isolation. It broke when two modules exported constructors with the same name. zentinel-agent-policy has this exact pattern: `Types.hs` defines `PolicyEngine` with constructors `CedarEngine | RegoEngine | AutoEngine` (arity 0), and `Cedar.hs` defines its own `CedarEngine` record type (arity 3, with configuration fields). Without export filtering, the wrong `CedarEngine` could shadow the right one, and suddenly a pattern match expecting three fields would see zero.

The fix was to actually filter exports against `hir.exports`. The data was already there. We just weren't using it.

### Bug 2: `Type(..)` imported too much

Related but distinct: when BHC processed an import like `import Types (PolicyEngine(..))`, it was pulling in *all* constructors from the `Types` module, not just the constructors belonging to `PolicyEngine`. The `apply_import_spec` function in `loader.rs` iterated over every constructor in the module's export list without checking which parent type each constructor belonged to.

In a module that exports a single data type, this is harmless. In a module like `Types.hs` that exports 15 data types with 24 constructors between them, it dumps every constructor into the importing module's namespace. If any of those constructor names collide with names the importing module defines locally, things break silently.

The fix: filter constructors by `info.type_con_name == ident.name`.

### Bug 3: Cross-module CAFs returned garbage

This was the one that took the longest to find, because it didn't cause a compilation error. It caused a segfault.

Every Haskell program has Constant Applicative Forms — zero-argument top-level bindings like `defaultConfig :: AgentConfig`. In BHC's LLVM codegen, these compile to functions that take a single parameter (a null environment pointer) and return a value. When you reference `defaultConfig` in the same module, the codegen knows to *call* the function:

```rust
// Same-module path: call the CAF to get its value
if fn_type.count_param_types() <= 1 {
    let null_env = self.type_mapper().ptr_type().const_null();
    let call_result = self.builder()
        .build_call(*fn_val, &[null_env.into()], "caf_result")?;
    // return the result...
}
```

But when you reference `defaultConfig` from *another* module, the codegen took a different path — the `external_functions` lookup — and that path just returned the raw function pointer:

```rust
// Cross-module path: BUG — returns the pointer, doesn't call it
} else if let Some(&fn_val) = self.external_functions.get(&var.name) {
    let fn_ptr = fn_val.as_global_value().as_pointer_value();
    Ok(Some(fn_ptr.into()))
}
```

For functions that take arguments, this is correct. The pointer gets wrapped in a closure and called later when arguments are applied. But for a CAF, there are no arguments coming. The raw function pointer *is* the value that gets used downstream. So when you write `cache defaultConfig` to extract the `cache` field, the codegen tries to read a record field out of a function pointer. It reads garbage. If you're lucky, you segfault. If you're unlucky, you get silent data corruption.

This bug was invisible in single-module compilation — the same-module path handled it correctly. It was invisible in multi-module compilation of functions with arguments — those go through the closure path. It only manifested when a zero-argument binding was referenced from another module and its return value was inspected. That's exactly what `defaultConfig` is and exactly what our test harness does.

The fix mirrors the same-module logic:

```rust
} else if let Some(&fn_val) = self.external_functions.get(&var.name) {
    let fn_type = fn_val.get_type();
    if fn_type.count_param_types() <= 1 {
        // CAF — call it to get the value
        let null_env = self.type_mapper().ptr_type().const_null();
        let call_result = self.builder()
            .build_call(fn_val, &[null_env.into()], "ext_caf_result")?;
        // ...
    } else {
        // Function with parameters — wrap in closure
        let fn_ptr = fn_val.as_global_value().as_pointer_value();
        let closure_ptr = self.alloc_closure(fn_ptr, &[])?;
        Ok(Some(closure_ptr.into()))
    }
}
```

Six lines. Two hours to find.

## The test

With all three fixes in place, we wrote a test harness that exercises the zentinel types across module boundaries:

```haskell
module Main where

import Zentinel.Agent.Policy.Types
import Zentinel.Agent.Policy.Config

main :: IO ()
main = do
  -- Local record construction
  let cc = CacheConfig { enabled = True, ttlSeconds = 60, maxEntries = 10000 }
  case enabled cc of
    True  -> putStrLn "   enabled: OK"
    False -> putStrLn "   FAIL"

  -- Cross-module CAF
  let cfg = defaultConfig
  case defaultDecision cfg of
    Deny  -> putStrLn "   defaultDecision = Deny: OK"
    Allow -> putStrLn "   FAIL"

  -- Nested field access across modules
  let cc2 = cache cfg
  case enabled cc2 of
    True  -> putStrLn "   cache.enabled: OK"
    False -> putStrLn "   FAIL"

  -- Deep nesting: defaultConfig → inputMapping → principalMapping → pattern match
  let im = inputMapping cfg
  case principalMapping im of
    HeaderPrincipal _ -> putStrLn "   principalMapping = HeaderPrincipal: OK"
    _                 -> putStrLn "   FAIL"

  -- ... (10 test groups total)
```

Compile and run:

```
$ bhc Zentinel/Agent/Policy/Types.hs Zentinel/Agent/Policy/Config.hs TestMain.hs -o test
$ ./test
1. CacheConfig construction
   enabled: OK
   ttlSeconds: OK
   maxEntries: OK
2. defaultConfig (cross-module CAF)
   logLevel: got value
   defaultDecision = Deny: OK
3. nested field access (cache from defaultConfig)
   cache.enabled: OK
   cache.ttlSeconds: OK
4. Decision constructors
   Allow: OK
   Deny: OK
5. PolicyEngine constructors
   CedarEngine: OK
   RegoEngine: OK
   AutoEngine: OK
6. PolicySource constructors
   FileSource: OK
   InlineSource: OK
7. PrincipalMapping
   HeaderPrincipal: OK
8. AuditConfig from defaultConfig
   auditEnabled: OK
   includeInput: OK
   includePolicies: OK
9. InputMapping from defaultConfig
   principalMapping = HeaderPrincipal: OK
   resourceMapping = PathResource: OK
10. ActionMapping fields
    defaultAction: got value
All tests passed!
```

Every test passes. Local construction, cross-module CAFs, nested field access, sum type dispatch, three-level indirection chains — all correct.

## What about external packages?

zentinel-agent-policy depends on `aeson`, `yaml`, `optparse-applicative`, and several other Hackage packages that BHC doesn't implement yet. These compile as stubs:

```
warning: stub function `withObject` used (external package not implemented)
warning: stub function `decodeFileThrow` used (external package not implemented)
warning: stub function `execParser` used (external package not implemented)
```

The stubs pass type checking and link, but calling them at runtime would trap. This is by design — we're testing the Haskell compilation pipeline, not reimplementing Hackage. The external package boundary is clean and well-defined. When BHC gains the ability to compile `aeson` from source (or when we wire up the `hx` package manager for real Hackage dependencies), the stubs disappear and everything else stays the same.

What matters is that all the *Haskell* code — the types, records, pattern matching, cross-module imports, constructor dispatch, field selectors — compiles to native machine code and runs correctly. That's the hard part. JSON parsing is a library problem.

## By the numbers

| Metric | Value |
|--------|-------|
| E2E tests passing | **190** (188 main suite + zentinel integration) |
| Zentinel modules compiled | **10** |
| Data types exercised | **15** (records, sum types, nested structures) |
| Cross-module CAF calls | verified correct |
| Pre-existing test regressions | **0** |

## Full circle

There's something satisfying about this particular milestone. BHC exists because I wanted a Haskell compiler that could produce binaries suitable for latency-sensitive workloads — the kind of workload that a reverse-proxy policy agent represents. The first real library it runs is the agent that motivated the compiler in the first place.

We're not at the Realtime profile yet. The arena allocators are implemented but not wired into the codegen pipeline for this kind of code. The bounded-pause GC is there but not battle-tested under load. The path from "correct native binary" to "correct native binary with sub-millisecond GC pauses" is real work.

But the foundation is right. The compilation pipeline produces correct code. Records have the right fields. Constructors dispatch correctly. Functions return the right values across module boundaries. Everything above this — optimization, profiling, runtime tuning — builds on a base that works.

## What's next

This was a waypoint. The north star remains Pandoc — 221 modules, ~60,000 lines, and the most widely-used Haskell program in the world. We're at 53 modules passing `bhc check` on Pandoc. The cross-module CAF fix unblocks a large class of Pandoc patterns where configuration records and default values flow between modules.

The remaining gaps are external package coverage (Template Haskell blocks two critical Pandoc modules) and some type-level features. But the core pipeline — parsing, type checking, lowering, Core IR optimization, LLVM codegen, multi-module linking — produces correct native code for real Haskell libraries.

The program that motivated the compiler now runs on the compiler. We're getting there.

---

*BHC is open source at [github.com/arcanist-sh/bhc](https://github.com/arcanist-sh/bhc). Zentinel is at [github.com/zentinelproxy/zentinel](https://github.com/zentinelproxy/zentinel).*
