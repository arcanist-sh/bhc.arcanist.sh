+++
title = "Phase 1: Native Code Generation is Complete"
description = "BHC can now parse Haskell, type check, lower to Core IR, generate LLVM IR, link with the RTS, and produce working native executables. Here's what we've achieved."
date = 2026-01-25
template = "blog-post.html"

[extra]
tag = "milestone"
+++

We're thrilled to announce that **Phase 1 of BHC development is complete**. The Basel Haskell Compiler can now take Haskell source code and produce working native executables.

This is a significant milestone. When we started BHC, the goal was ambitious: build a new Haskell compiler from scratch in Rust, with a focus on predictable performance, multiple compilation targets, and a modern architecture. Today, that compiler runs real Haskell programs.

## What Works Now

Here's a summary of the features we've validated:

| Test | Program | Result |
|------|---------|--------|
| Hello World | `putStrLn "Hello, World!"` | Runs correctly |
| Integers | `print 42` | Outputs `42` |
| Arithmetic | `print (1 + 2 * 3)` | Outputs `7` |
| Let bindings | `let x = 5 in print x` | Outputs `5` |
| Functions | `double x = x + x; print (double 21)` | Outputs `42` |
| Factorial | `factorial 10` | Outputs `3,628,800` |
| Fibonacci | `fib 20` | Outputs `6,765` |

These aren't just toy examples. The factorial and fibonacci tests exercise recursion, pattern matching, and numeric operations at scale. `factorial 10` computes 3,628,800 correctly. `fib 20` returns 6,765 with the expected recursive call tree.

## The Complete Pipeline

BHC now implements the full compilation pipeline:

1. **Parsing** - Haskell source code is parsed into an AST
2. **Type checking** - Full Hindley-Milner type inference with extensions
3. **Core IR lowering** - AST is transformed to a typed intermediate representation
4. **LLVM IR generation** - Core IR compiles to LLVM's intermediate format
5. **RTS linking** - The runtime system provides memory management and primitives
6. **Native executable** - LLVM produces optimized machine code

Each stage has been tested independently and as part of the integrated pipeline. The result: you can write Haskell, run `bhc`, and get a binary that executes.

## What This Means

Phase 1 proves the architecture works. The compiler frontend (parsing, type checking) connects cleanly to the backend (LLVM codegen, RTS). The Core IR serves as a stable interface between them.

More importantly, this validates our approach:

- **Rust as implementation language** - Memory safety and performance work well for compiler internals
- **LLVM for codegen** - We get battle-tested optimization and broad target support
- **Clean-slate design** - No legacy constraints, modern architecture from the start

## What's Next

Phase 2 focuses on real-world Haskell compatibility:

- **Module system** - Full support for imports, exports, and separate compilation
- **Type classes** - Instances, deriving, and the standard class hierarchy
- **Pattern matching** - Complete coverage checking and compilation
- **GHC extensions** - GADTs, TypeFamilies, and the extensions real code uses

We're also expanding the test suite with packages from Hackage. The goal: compile significant portions of the Haskell ecosystem without modification.

## Try It Out

BHC is open source and available now:

```bash
curl -fsSL https://arcanist.sh/bhc/install.sh | sh
```

Or build from source:

```bash
git clone https://github.com/arcanist-sh/bhc
cd bhc
cargo build --release
```

Write a simple Haskell program and compile it:

```haskell
-- hello.hs
main :: IO ()
main = putStrLn "Hello from BHC!"
```

```bash
bhc hello.hs -o hello
./hello
```

## Thank You

This milestone wouldn't be possible without the Haskell community's decades of research and implementation work. BHC builds on that foundation while exploring new directions.

We're just getting started. Follow development on [GitHub](https://github.com/arcanist-sh/bhc), and let us know what you'd like to see next.

---

*The BHC Team*
