+++
title = "Compiler Architecture"
description = "Deep dive into BHC's internal architecture and compilation pipeline"
template = "page.html"
+++

# Compiler Architecture

BHC is a multi-stage optimizing compiler written in Rust. This page documents the internal architecture, intermediate representations, and compilation pipeline.

## High-Level Pipeline

Compilation in BHC proceeds through a series of well-defined stages. Source code enters the frontend, which parses it and performs type checking. The result is lowered to Core IR—a small, explicitly-typed intermediate language that serves as the foundation for optimization. After extensive Core-to-Core transformations, the code branches into profile-specific intermediate representations before final code generation.

The key insight is that BHC maintains a single frontend and optimization pipeline, then diverges based on the selected runtime profile. This ensures consistent semantics while allowing profile-specific code generation strategies.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BHC COMPILATION PIPELINE                            │
└─────────────────────────────────────────────────────────────────────────────┘

  Source Files (.hs)
        │
        ▼
┌───────────────────┐
│     FRONTEND      │  Parsing, name resolution, type inference
└───────────────────┘
        │
        ▼
┌───────────────────┐
│     CORE IR       │  Desugared, typed lambda calculus
└───────────────────┘
        │
        ▼
┌───────────────────┐
│   OPTIMIZATION    │  Inlining, fusion, strictness analysis
└───────────────────┘
        │
        ├──────────────────┬──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│   TENSOR IR   │  │    STG IR     │  │   DIRECT IR   │
│  (numeric)    │  │  (default)    │  │   (edge)      │
└───────────────┘  └───────────────┘  └───────────────┘
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ LLVM / CUDA   │  │  LLVM / C     │  │  WASM / LLVM  │
│   CODEGEN     │  │   CODEGEN     │  │   CODEGEN     │
└───────────────┘  └───────────────┘  └───────────────┘
        │                  │                  │
        └──────────────────┴──────────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   LINKER    │
                    └─────────────┘
                           │
                           ▼
                      Executable
```

The three profile-specific IRs serve different purposes: **Tensor IR** (numeric profile) understands array shapes and enables aggressive fusion and SIMD lowering. **STG IR** (default profile) implements lazy evaluation with the Spineless Tagless G-machine model. **Direct IR** (edge profile) produces compact code suitable for resource-constrained environments like WebAssembly.

## Frontend Architecture

The frontend transforms source code into typed Core IR through several phases. Each phase has a single responsibility and produces a well-defined output that feeds into the next stage.

### Lexer and Parser

The lexer converts source text into a token stream. Haskell's layout rule (significant whitespace) is handled here—the layout engine inserts virtual braces and semicolons so the parser doesn't need to understand indentation. This separation keeps the parser simple and makes error recovery more predictable.

The parser is a hand-written recursive descent parser rather than a generated one. This choice gives better error messages and makes it easier to handle Haskell's context-sensitive grammar (particularly around operators and layout). The parser produces a Concrete Syntax Tree (CST) that preserves all source structure, which is then simplified to an Abstract Syntax Tree (AST).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND DETAIL                                │
└─────────────────────────────────────────────────────────────────────────────┘

   Source Text
        │
        ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                               LEXER                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Layout    │  │  Keywords   │  │  Operators  │  │  Literals   │      │
│  │   Engine    │──│  & Idents   │──│  & Symbols  │──│  & Strings  │      │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘      │
│         │                                                                  │
│         ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  Token Stream with Layout-Inserted Braces and Semicolons        │      │
│  └─────────────────────────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                               PARSER                                       │
│                                                                            │
│   ┌─────────────────────────────────────────────────────────────────┐     │
│   │                    Recursive Descent Parser                      │     │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │     │
│   │  │  Module  │  │  Decls   │  │  Exprs   │  │ Patterns │        │     │
│   │  │  Header  │──│  Parser  │──│  Parser  │──│  Parser  │        │     │
│   │  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │     │
│   └─────────────────────────────────────────────────────────────────┘     │
│         │                                                                  │
│         ▼                                                                  │
│   ┌─────────────────────────────────────────────────────────────────┐     │
│   │  Concrete Syntax Tree (CST) - preserves all source structure     │     │
│   └─────────────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────────────┘
        │
        ▼
   Abstract Syntax Tree (AST)
```

### Name Resolution

After parsing, names in the AST are just strings. Name resolution assigns each name a unique identifier and connects uses to definitions. This phase runs three sub-passes in parallel:

The **Import Resolver** finds and loads interface files for imported modules, checking that requested names are actually exported. The **Scope Builder** constructs lexical scopes, tracking where each name is bound and detecting shadowing. The **Fixity Resolver** handles operator precedence and associativity, rewriting infix expressions into proper tree structure.

The output is a Resolved AST where every name reference includes a unique ID pointing to its definition. This makes subsequent phases simpler since they never need to worry about scoping rules.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           NAME RESOLUTION                                   │
└─────────────────────────────────────────────────────────────────────────────┘

                              AST (unresolved names)
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          ▼                            ▼                            ▼
   ┌─────────────┐              ┌─────────────┐              ┌─────────────┐
   │   Import    │              │   Scope     │              │  Fixity     │
   │  Resolver   │              │  Builder    │              │  Resolver   │
   │             │              │             │              │             │
   │ - Find      │              │ - Build     │              │ - Resolve   │
   │   modules   │              │   lexical   │              │   operator  │
   │ - Check     │              │   scopes    │              │   prece-    │
   │   exports   │              │ - Track     │              │   dence     │
   │ - Resolve   │              │   bindings  │              │ - Rewrite   │
   │   qualif.   │              │ - Check     │              │   infix     │
   │   names     │              │   shadows   │              │   exprs     │
   └─────────────┘              └─────────────┘              └─────────────┘
          │                            │                            │
          └────────────────────────────┼────────────────────────────┘
                                       ▼
                           ┌───────────────────┐
                           │  Resolved AST     │
                           │  (all names have  │
                           │  unique IDs)      │
                           └───────────────────┘
```

### Type Inference

Type inference in BHC uses a constraint-based approach inspired by OutsideIn(X). Rather than inferring types directly, the algorithm generates typing constraints from the program structure, then solves those constraints to find a valid typing.

**Constraint Generation** walks the AST and emits constraints. For a variable `x`, it looks up the known type. For a lambda `\x -> e`, it creates a fresh type variable for `x`, checks `e`, and produces a function type. For application `f x`, it requires `f` to have function type and `x` to match the argument type.

**Constraint Solving** has two components. The **Unification Engine** handles equality constraints between types, using union-find for efficient variable merging and occurs-check to prevent infinite types. **Instance Resolution** handles type class constraints, finding matching instances and generating evidence (the dictionaries that get passed at runtime).

The output is a Typed AST with explicit type annotations everywhere, plus evidence terms for all type class uses.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            TYPE INFERENCE                                   │
└─────────────────────────────────────────────────────────────────────────────┘

                           Resolved AST
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                        CONSTRAINT GENERATION                               │
│                                                                            │
│    For each expression, generate typing constraints:                       │
│                                                                            │
│    Expression          Generated Constraint                                │
│    ──────────          ─────────────────────                               │
│    x                   x : τ_x  (lookup)                                   │
│    \x -> e             fresh α, check e : β, result α → β                  │
│    f x                 f : α → β, x : α, result β                          │
│    let x = e1 in e2    generalize(check e1), check e2 with x bound         │
│    if c then t else f  c : Bool, t : τ, f : τ, result τ                    │
│                                                                            │
└───────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         CONSTRAINT SOLVING                                 │
│                                                                            │
│   ┌────────────────────────────────────────────────────────────────┐      │
│   │                    Unification Engine                           │      │
│   │                                                                 │      │
│   │   τ₁ ~ τ₂                                                       │      │
│   │    │                                                            │      │
│   │    ├── Both type vars? ──────────────► Union-find merge         │      │
│   │    │                                                            │      │
│   │    ├── One type var? ────────────────► Occurs check, then bind  │      │
│   │    │                                                            │      │
│   │    ├── Both constructors? ───────────► Recurse on arguments     │      │
│   │    │                                                            │      │
│   │    └── Mismatch? ────────────────────► Type error               │      │
│   │                                                                 │      │
│   └────────────────────────────────────────────────────────────────┘      │
│                                                                            │
│   ┌────────────────────────────────────────────────────────────────┐      │
│   │                   Instance Resolution                           │      │
│   │                                                                 │      │
│   │   For constraint (C τ):                                         │      │
│   │    1. Find instances matching τ                                 │      │
│   │    2. Check instance context satisfied                          │      │
│   │    3. Generate evidence (dictionary passing)                    │      │
│   │                                                                 │      │
│   └────────────────────────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                    Typed AST + Evidence
```

## Core IR

Core is BHC's primary intermediate representation: a small, explicitly-typed functional language. Its simplicity makes optimization tractable—there are only a handful of expression forms, and every sub-expression has a known type.

### Core Language

Core is essentially System F (polymorphic lambda calculus) with a few extensions for practical compilation: let-bindings for sharing, case expressions for pattern matching, and type casts for newtypes and coercions.

The grammar below shows all expression forms. Note that Core is fully explicit: lambdas annotate their parameters, let-bindings annotate their variables, and type applications are written out rather than inferred. This explicitness simplifies transformations since the type of any expression can be computed without re-running type inference.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CORE IR GRAMMAR                                │
└─────────────────────────────────────────────────────────────────────────────┘

  Expression e ::=
      │
      ├── x                         Variable reference
      │
      ├── K                         Data constructor
      │
      ├── lit                       Literal (Int#, Float#, String, ...)
      │
      ├── e₁ e₂                     Application
      │
      ├── e @τ                      Type application
      │
      ├── λx:τ. e                   Lambda abstraction
      │
      ├── Λα:κ. e                   Type abstraction
      │
      ├── let x:τ = e₁ in e₂        Non-recursive let
      │
      ├── letrec binds in e         Recursive let (mutually recursive)
      │
      ├── case e of alts            Case expression (pattern matching)
      │
      └── e ▷ τ                     Type cast (for newtypes, coercions)


  Alternatives alt ::=
      │
      ├── K x₁ ... xₙ → e           Constructor pattern
      │
      ├── lit → e                   Literal pattern
      │
      └── _ → e                     Default pattern


  Type τ ::=
      │
      ├── α                         Type variable
      │
      ├── T                         Type constructor
      │
      ├── τ₁ τ₂                     Type application
      │
      ├── τ₁ → τ₂                   Function type
      │
      └── ∀α:κ. τ                   Universal quantification
```

### Desugaring Examples

Haskell's surface syntax includes many conveniences that don't exist in Core. Desugaring translates these into the simpler Core constructs. This table shows some common transformations:

**Do-notation** becomes explicit bind (`>>=`) calls. **List comprehensions** become combinations of `concatMap`, `filter`, and list construction. **Multiple-clause function definitions** become single lambdas with case expressions. **Type classes** become data types (dictionaries) with the class methods as fields, and instances become values of those dictionary types.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DESUGARING TO CORE                                 │
└─────────────────────────────────────────────────────────────────────────────┘

  Source Haskell                      Core IR
  ──────────────                      ───────

  do { x <- m; e }          ═══►      m >>= \x -> e

  [x | x <- xs, p x]        ═══►      concatMap (\x -> if p x
                                                       then [x]
                                                       else []) xs

  f x y = e                 ═══►      f = \x -> \y -> e
  where g = ...                       where g = ...

  data Maybe a =            ═══►      data Maybe a = Nothing | Just a
    Nothing | Just a                  -- generates:
                                      --   Nothing : ∀a. Maybe a
                                      --   Just    : ∀a. a → Maybe a

  class Eq a where          ═══►      data Eq a = MkEq {
    (==) :: a -> a -> Bool              eq :: a -> a -> Bool
                                      }
                                      -- (==) becomes dictionary selector

  instance Eq Int where     ═══►      eqInt :: Eq Int
    x == y = primEqInt x y            eqInt = MkEq { eq = primEqInt }
```

## Optimization Pipeline

BHC's optimizer runs multiple passes over Core IR, each making the program smaller, faster, or both. The passes are composable and the optimizer iterates until reaching a fixed point (no more improvements possible).

### Core-to-Core Transformations

The optimization pipeline has four main phases, each building on the previous:

**The Simplifier** is the workhorse. It runs repeatedly, applying local transformations: inlining small or once-used functions, beta-reducing applications, eliminating dead code, simplifying case expressions on known constructors, folding constants, and floating let-bindings to better positions. Each individual transformation is simple, but their combination is powerful.

**Specialization** creates monomorphic copies of polymorphic functions at call sites where the type arguments are known. A call to `map @Int @Bool` generates a specialized `map_Int_Bool` that works only on those types. This enables further optimization since the specialized version can use unboxed representations.

**Strictness Analysis** determines which function arguments are always evaluated. A function is "strict" in an argument if evaluating the function always evaluates that argument. Knowing strictness lets us evaluate arguments before the call (avoiding thunk allocation) and pass them unboxed.

**Worker/Wrapper** transforms strict functions to separate the "wrapper" (which handles boxed arguments) from the "worker" (which operates on unboxed values). The wrapper unpacks arguments and repacks results; the worker does the real computation using efficient unboxed operations.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OPTIMIZATION PIPELINE                                │
└─────────────────────────────────────────────────────────────────────────────┘

                              Core IR (from frontend)
                                       │
    ┌──────────────────────────────────┼──────────────────────────────────┐
    │                                  │                                   │
    │  ┌───────────────────────────────┴───────────────────────────────┐  │
    │  │                        SIMPLIFIER                              │  │
    │  │                                                                │  │
    │  │   Iterates until fixed point:                                  │  │
    │  │                                                                │  │
    │  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
    │  │   │   Inlining   │  │    Beta      │  │    Dead      │        │  │
    │  │   │              │──│  Reduction   │──│    Code      │        │  │
    │  │   │  - Small     │  │              │  │  Elimination │        │  │
    │  │   │    functions │  │  (λx.e) v    │  │              │        │  │
    │  │   │  - Once-used │  │    ═► e[v/x] │  │  Remove      │        │  │
    │  │   │  - CONLIKE   │  │              │  │  unused      │        │  │
    │  │   └──────────────┘  └──────────────┘  └──────────────┘        │  │
    │  │          │                  │                  │               │  │
    │  │          ▼                  ▼                  ▼               │  │
    │  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
    │  │   │    Case      │  │   Constant   │  │    Let       │        │  │
    │  │   │  of Known    │──│   Folding    │──│   Floating   │        │  │
    │  │   │ Constructor  │  │              │  │              │        │  │
    │  │   │              │  │  2 + 3 ═► 5  │  │  Float lets  │        │  │
    │  │   │ case K x of  │  │  not True    │  │  out of      │        │  │
    │  │   │   K y -> e   │  │    ═► False  │  │  lambdas     │        │  │
    │  │   │   ═► e[x/y]  │  │              │  │              │        │  │
    │  │   └──────────────┘  └──────────────┘  └──────────────┘        │  │
    │  │                                                                │  │
    │  └────────────────────────────────────────────────────────────────┘  │
    │                                  │                                   │
    │                                  ▼                                   │
    │  ┌────────────────────────────────────────────────────────────────┐  │
    │  │                    SPECIALIZATION                               │  │
    │  │                                                                 │  │
    │  │   Polymorphic function           Specialized versions           │  │
    │  │   ─────────────────────           ────────────────────          │  │
    │  │   map :: (a→b) → [a] → [b]  ═══►  map_Int_Bool :: ...           │  │
    │  │                                   map_Char_Char :: ...          │  │
    │  │                                                                 │  │
    │  │   Creates monomorphic copies at known call sites                │  │
    │  │   Enables further optimization (unboxing, fusion)               │  │
    │  │                                                                 │  │
    │  └────────────────────────────────────────────────────────────────┘  │
    │                                  │                                   │
    │                                  ▼                                   │
    │  ┌────────────────────────────────────────────────────────────────┐  │
    │  │                   STRICTNESS ANALYSIS                           │  │
    │  │                                                                 │  │
    │  │   Determines which arguments are always evaluated:              │  │
    │  │                                                                 │  │
    │  │   f x y = case x of           f is strict in x (always eval'd) │  │
    │  │             0 -> y            f is lazy in y (not always)      │  │
    │  │             _ -> x + y                                          │  │
    │  │                                                                 │  │
    │  │   Strict arguments can be:                                      │  │
    │  │   - Evaluated before call (worker/wrapper)                      │  │
    │  │   - Passed unboxed                                              │  │
    │  │   - Allocated on stack instead of heap                          │  │
    │  │                                                                 │  │
    │  └────────────────────────────────────────────────────────────────┘  │
    │                                  │                                   │
    │                                  ▼                                   │
    │  ┌────────────────────────────────────────────────────────────────┐  │
    │  │                   WORKER/WRAPPER                                │  │
    │  │                                                                 │  │
    │  │   Original:                    After W/W:                       │  │
    │  │   ─────────                    ──────────                       │  │
    │  │   f :: Int → Int → Int         f :: Int → Int → Int             │  │
    │  │   f x y = x + y                f x y = case x of                │  │
    │  │                                          I# x' → case y of      │  │
    │  │                                                   I# y' →       │  │
    │  │                                                     I# (f' x' y')│  │
    │  │                                                                 │  │
    │  │                                f' :: Int# → Int# → Int#         │  │
    │  │                                f' x y = x +# y  -- unboxed!     │  │
    │  │                                                                 │  │
    │  └────────────────────────────────────────────────────────────────┘  │
    │                                  │                                   │
    └──────────────────────────────────┼──────────────────────────────────┘
                                       │
                                       ▼
                              Optimized Core IR
```

### Fusion System

Fusion eliminates intermediate data structures in pipelines of operations. Without fusion, `map f (map g xs)` would build a complete intermediate list; with fusion, it becomes a single traversal applying `f . g` to each element.

BHC implements **stream fusion**, which represents list operations as stream transformers. A `Stream` is a state machine that produces elements one at a time. The key rewrite rule is `fromStream (toStream xs) = xs`—converting a list to a stream and back is a no-op. When this rule fires between adjacent operations, intermediate lists vanish.

The diagram shows how `map f (map g xs)` fuses: after desugaring through streams, the `stream/unstream` rule fires, then `mapS/mapS` fusion combines the two maps into one. The result is a single `map (f . g) xs` with no intermediate list.

BHC **guarantees** fusion for common patterns. If you write `sum (map f xs)`, the compiler will produce a single loop—no intermediate list, no extra allocation. Failure to fuse these patterns is considered a compiler bug.

> **Status (September 2026):** this contract is met in the Tensor IR and Loop IR stages, which pass their internal tests. It is not yet realised in native code generation: the native backend does not consume the fused Loop IR, so a numeric-profile build currently produces the same code and the same timing as the default profile. "Guaranteed" is the standard we hold ourselves to, not a description of today's binaries. See the [status page](@/status.md).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             FUSION SYSTEM                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  BHC implements stream fusion with guaranteed fusion for standard patterns.

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                        STREAM REPRESENTATION                             │
  │                                                                          │
  │   data Stream a = ∃s. Stream (s → Step a s) s                           │
  │                                                                          │
  │   data Step a s = Yield a s    -- produce element, continue             │
  │                 | Skip s       -- no element, continue                  │
  │                 | Done         -- finished                              │
  │                                                                          │
  │   toStream   :: [a] → Stream a      -- list to stream                   │
  │   fromStream :: Stream a → [a]      -- stream to list                   │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                         FUSION RULES                                     │
  │                                                                          │
  │   Rule: stream/unstream                                                  │
  │   ─────────────────────                                                  │
  │   fromStream (toStream xs)  ═══►  xs                                    │
  │                                                                          │
  │   This rule eliminates intermediate structures:                          │
  │                                                                          │
  │   map f (map g xs)                                                       │
  │     = fromStream (mapS f (toStream (fromStream (mapS g (toStream xs))))) │
  │     ═══► [stream/unstream]                                               │
  │     = fromStream (mapS f (mapS g (toStream xs)))                         │
  │     ═══► [mapS/mapS fusion]                                              │
  │     = fromStream (mapS (f . g) (toStream xs))                            │
  │     = map (f . g) xs                                                     │
  │                                                                          │
  │   Result: single traversal, no intermediate list!                        │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    GUARANTEED FUSION PATTERNS                            │
  │                                                                          │
  │   BHC guarantees fusion for these patterns (failure = compiler bug):    │
  │                                                                          │
  │   ✓  map f . map g           ═══►  map (f . g)                          │
  │   ✓  filter p . filter q     ═══►  filter (\x → p x && q x)             │
  │   ✓  map f . filter p        ═══►  single pass                          │
  │   ✓  sum . map f             ═══►  foldl' (\a x → a + f x) 0            │
  │   ✓  foldr f z . map g       ═══►  foldr (\x a → f (g x) a) z           │
  │   ✓  zip xs . map f          ═══►  single pass over xs                  │
  │                                                                          │
  │   The numeric profile extends this to tensor operations.                 │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Backend Architecture

After optimization, the compiler lowers Core IR to profile-specific representations for code generation.

### STG Machine (Default Profile)

The default profile uses the **Spineless Tagless G-machine (STG)**, an abstract machine designed for lazy functional languages. "Spineless" means there's no central evaluation stack (continuations are heap-allocated). "Tagless" means we don't store type tags in closures—instead, every closure has a pointer to an info table containing its entry code.

**STG Syntax** is simpler than Core: all applications are saturated (no partial application as an expression), and only atoms (variables or literals) can be arguments. Functions are represented as closures with an "updateability" flag: updatable closures (`\u`) are thunks that get overwritten with their result; non-updatable closures (`\n`) are functions or already-evaluated values.

**Heap Object Layout** is uniform: every object has an info pointer followed by payload fields. The info table (shared by all closures of the same shape) contains the entry code to execute when the closure is "entered," layout information for garbage collection, the closure type (THUNK, FUN, CONSTR, etc.), and a static reference table.

**Evaluation** proceeds by "entering" closures. To evaluate a `case` expression, we push a continuation and jump to the scrutinee's entry code. If it's a thunk, the entry code evaluates it and updates the closure with the result. If it's a constructor, we pop the continuation and match against alternatives.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        STG MACHINE (DEFAULT PROFILE)                        │
└─────────────────────────────────────────────────────────────────────────────┘

  The Spineless Tagless G-machine is BHC's evaluation model for lazy code.

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                          STG SYNTAX                                      │
  │                                                                          │
  │   Binding    b ::= f = \π x₁...xₙ → e                                   │
  │                    where π ∈ {\n, \u}  (updatable / non-updatable)      │
  │                                                                          │
  │   Expression e ::= let b in e                                           │
  │                  | letrec b₁...bₙ in e                                   │
  │                  | case e of alts                                        │
  │                  | f a₁...aₙ           (saturated application)          │
  │                  | C a₁...aₙ           (saturated constructor)          │
  │                  | prim a₁...aₙ        (primitive operation)            │
  │                                                                          │
  │   Atom       a ::= x | lit                                              │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       HEAP OBJECT LAYOUT                                 │
  │                                                                          │
  │   Every heap object has this structure:                                  │
  │                                                                          │
  │   ┌────────────────┬────────────────────────────────────────┐           │
  │   │  Info Pointer  │  Payload (free variables / fields)     │           │
  │   └────────────────┴────────────────────────────────────────┘           │
  │          │                                                               │
  │          ▼                                                               │
  │   ┌──────────────────────────────────────────────────────────┐          │
  │   │                    Info Table                             │          │
  │   │  ┌──────────┬──────────┬──────────┬──────────┐           │          │
  │   │  │  Entry   │  Layout  │  Closure │   SRT    │           │          │
  │   │  │  Code    │  Info    │   Type   │  (GC)    │           │          │
  │   │  └──────────┴──────────┴──────────┴──────────┘           │          │
  │   └──────────────────────────────────────────────────────────┘          │
  │                                                                          │
  │   Closure Types:                                                         │
  │   ─────────────────                                                      │
  │   THUNK     - Unevaluated expression (updatable)                        │
  │   FUN       - Function closure                                          │
  │   PAP       - Partial application                                       │
  │   CONSTR    - Data constructor                                          │
  │   BLACKHOLE - Thunk being evaluated (cycle detection)                   │
  │   IND       - Indirection (after update)                                │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                        EVALUATION MODEL                                  │
  │                                                                          │
  │   Registers:                                                             │
  │   ──────────                                                             │
  │   Sp  - Stack pointer                                                    │
  │   Hp  - Heap pointer                                                     │
  │   R1  - Node register (current closure)                                  │
  │   R2+ - Argument registers                                               │
  │                                                                          │
  │   Evaluation of (case e of alts):                                        │
  │   ──────────────────────────────────                                     │
  │                                                                          │
  │   1. Push continuation (return address + saved registers)                │
  │   2. Enter e (jump to its entry code)                                    │
  │   3. If e is:                                                            │
  │      - THUNK: evaluate, update with result, return                       │
  │      - FUN:   error (case on function)                                   │
  │      - CONSTR: return immediately, match alts                            │
  │   4. Pop continuation, select matching alternative                       │
  │   5. Bind pattern variables, continue                                    │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

### Tensor IR (Numeric Profile)

The numeric profile uses **Tensor IR** for array and matrix operations. Unlike Core's uniform representation, Tensor IR tracks array shapes and strides at the type level. This enables aggressive fusion across array operations and automatic SIMD vectorization.

**Operations** in Tensor IR are higher-level than Core: `map`, `reduce`, `contract` (generalized matrix multiplication), `reshape`, and `slice` are primitives. The compiler understands their semantics and can fuse them.

**Tensor Fusion** works similarly to list fusion but is more aggressive. The example shows `normalize`, which requires squaring each element, summing, taking a square root, then dividing each element by the norm. Naive execution needs three passes over the data. After fusion, BHC produces two passes: one to compute the sum of squares, one to normalize—the theoretical minimum since we need the complete sum before dividing.

**SIMD Lowering** converts element-wise operations into vector instructions. A `map (+1)` over 1024 floats becomes a loop that processes 8 floats at a time using AVX2 vector instructions (or 4 with SSE, or 16 with AVX-512, depending on the target).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       TENSOR IR (NUMERIC PROFILE)                           │
└─────────────────────────────────────────────────────────────────────────────┘

  The numeric profile lowers array operations to Tensor IR for optimization.

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       TENSOR IR OPERATIONS                               │
  │                                                                          │
  │   Tensor τ[d₁,...,dₙ]  -- n-dimensional array of element type τ         │
  │                                                                          │
  │   Operations:                                                            │
  │   ───────────                                                            │
  │   alloc    : dims → Tensor τ dims                                       │
  │   index    : Tensor τ dims → indices → τ                                │
  │   store    : Tensor τ dims → indices → τ → ()                           │
  │   map      : (τ → σ) → Tensor τ d → Tensor σ d                          │
  │   reduce   : (τ → τ → τ) → τ → Tensor τ d → Tensor τ d'                 │
  │   contract : Tensor τ d₁ → Tensor τ d₂ → axes → Tensor τ d₃             │
  │   reshape  : Tensor τ d₁ → shape → Tensor τ d₂                          │
  │   slice    : Tensor τ d → range → Tensor τ d'                           │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    TENSOR FUSION EXAMPLE                                 │
  │                                                                          │
  │   Source:                                                                │
  │   ───────                                                                │
  │   normalize :: Tensor Float [n] → Tensor Float [n]                      │
  │   normalize t = map (/ norm) t                                          │
  │     where norm = sqrt (reduce (+) 0 (map (^2) t))                       │
  │                                                                          │
  │   Naive execution: 3 passes over data                                    │
  │                                                                          │
  │   After fusion:                                                          │
  │   ─────────────                                                          │
  │   normalize t =                                                          │
  │     let sumSq = 0.0                                                      │
  │     for i in 0..n:                                                       │
  │       sumSq += t[i] * t[i]        -- fused: square and sum              │
  │     let norm = sqrt sumSq                                                │
  │     for i in 0..n:                                                       │
  │       result[i] = t[i] / norm     -- single output pass                 │
  │                                                                          │
  │   Result: 2 passes (theoretical minimum: need sum before divide)         │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                      SIMD LOWERING                                       │
  │                                                                          │
  │   Tensor IR lowers to SIMD instructions where profitable:               │
  │                                                                          │
  │   map (+1) (xs :: Tensor Float [1024])                                  │
  │                                                                          │
  │   ═══► (on x86-64 with AVX2)                                            │
  │                                                                          │
  │   for i in 0..1024 step 8:                                               │
  │     v = vload256 xs[i:i+8]        -- load 8 floats                      │
  │     v = vaddps v, ones            -- parallel add                       │
  │     vstore256 result[i:i+8], v    -- store 8 floats                     │
  │                                                                          │
  │   Speedup: ~8x for pure compute (memory-bound less)                      │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

### Code Generation

After profile-specific lowering, code generation produces executable code for the target platform. BHC supports multiple backends, each optimized for different deployment scenarios.

The **LLVM Backend** produces LLVM IR, which then goes through LLVM's optimizer (dead code elimination, inlining, loop optimizations, auto-vectorization) and finally LLVM's code generators for native architectures (x86-64, ARM64, RISC-V). This path gives the best native performance.

The **WASM Backend** produces WebAssembly binaries for browser and edge deployment. The edge profile specifically targets this backend with a minimal runtime. The `wasm-opt` tool provides additional size and speed optimizations.

The **GPU backends** offload Tensor IR operations on large arrays, with the compiler handling memory transfers automatically. Metal is the path that runs on real hardware today: it compiles Metal Shading Language at run time and executes on the shared-memory GPU of any Apple-silicon Mac. The CUDA backend generates PTX for NVIDIA GPUs and the ROCm backend generates AMDGCN for AMD; both are validated in mock mode pending hardware.

All backends eventually feed into the linker, which combines the generated code with the BHC runtime system (garbage collector, scheduler, standard library) to produce a final executable.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CODE GENERATION                                    │
└─────────────────────────────────────────────────────────────────────────────┘

                              Optimized IR
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
           ▼                       ▼                       ▼
    ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
    │    LLVM     │         │    WASM     │         │    CUDA     │
    │   Backend   │         │   Backend   │         │   Backend   │
    └─────────────┘         └─────────────┘         └─────────────┘
           │                       │                       │
           ▼                       ▼                       ▼
    ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
    │  LLVM IR    │         │   WASM      │         │    PTX      │
    │             │         │   Binary    │         │  (NVIDIA)   │
    │  - SSA form │         │             │         │             │
    │  - Typed    │         │  - Stack    │         │  - GPU      │
    │  - Portable │         │    machine  │         │    threads  │
    │             │         │  - Sandboxed│         │  - SIMT     │
    └─────────────┘         └─────────────┘         └─────────────┘
           │                       │                       │
           ▼                       ▼                       │
    ┌─────────────┐         ┌─────────────┐               │
    │   LLVM      │         │   WASM      │               │
    │  Optimizer  │         │  Optimizer  │               │
    │             │         │   (wasm-opt)│               │
    │  - DCE      │         │             │               │
    │  - Inlining │         │  - Dead     │               │
    │  - Loop opt │         │    code     │               │
    │  - Vectorize│         │  - Inline   │               │
    └─────────────┘         └─────────────┘               │
           │                       │                       │
           ▼                       │                       │
    ┌─────────────┐                │                       │
    │   Native    │                │                       │
    │   Codegen   │                │                       │
    │             │                │                       │
    │  - x86-64   │                │                       │
    │  - ARM64    │                │                       │
    │  - RISC-V   │                │                       │
    └─────────────┘                │                       │
           │                       │                       │
           └───────────────────────┴───────────────────────┘
                                   │
                                   ▼
                            ┌─────────────┐
                            │   Linker    │
                            │             │
                            │  - RTS      │
                            │  - GC       │
                            │  - Stdlib   │
                            └─────────────┘
                                   │
                                   ▼
                             Executable
```

## Runtime System

The BHC runtime system (RTS) provides memory management, threading, and I/O services to running programs.

### Memory Management

BHC uses a **generational garbage collector**. The heap is divided into regions by object age:

The **Nursery (Generation 0)** holds newly allocated objects. Allocation is extremely fast: just bump a pointer. When the nursery fills, a minor GC copies live objects to the old generation and resets the nursery. Most objects die young, so most minor GCs are fast.

The **Old Generation (Generation 1+)** holds objects that survived nursery collection. It's collected less frequently since objects here tend to be long-lived. Major GC uses mark-compact: mark all reachable objects, then slide them together to eliminate fragmentation.

The **Large Object Space** holds objects too big for the nursery (arrays, pinned buffers). These aren't copied during GC—they're allocated directly in a separate region and freed in place.

Different profiles use different GC strategies: the server profile uses concurrent marking and incremental compaction to minimize pause times; the edge profile uses a simple two-space collector for smaller code size.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RUNTIME SYSTEM                                      │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                      HEAP LAYOUT                                         │
  │                                                                          │
  │   ┌─────────────────────────────────────────────────────────────────┐   │
  │   │                        HEAP                                      │   │
  │   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │   │
  │   │  │  Nursery     │  │   Old Gen    │  │   Large      │           │   │
  │   │  │  (Gen 0)     │  │   (Gen 1+)   │  │   Object     │           │   │
  │   │  │              │  │              │  │   Space      │           │   │
  │   │  │  - Small     │  │  - Survived  │  │              │           │   │
  │   │  │    allocs    │  │    objects   │  │  - Arrays    │           │   │
  │   │  │  - Bump      │  │  - Compacted │  │  - Pinned    │           │   │
  │   │  │    pointer   │  │    or aging  │  │  - FFI       │           │   │
  │   │  │  - Frequent  │  │  - Less      │  │              │           │   │
  │   │  │    GC        │  │    frequent  │  │              │           │   │
  │   │  │              │  │    GC        │  │              │           │   │
  │   │  └──────────────┘  └──────────────┘  └──────────────┘           │   │
  │   └─────────────────────────────────────────────────────────────────┘   │
  │                                                                          │
  │   Allocation (nursery):                                                  │
  │   ─────────────────────                                                  │
  │   alloc n =                                                              │
  │     if Hp + n > HpLim:                                                   │
  │       gc()              -- trigger garbage collection                    │
  │     ptr = Hp                                                             │
  │     Hp += n             -- bump allocation (very fast)                   │
  │     return ptr                                                           │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    GARBAGE COLLECTOR                                     │
  │                                                                          │
  │   BHC uses a generational, copying collector with optional compaction.   │
  │                                                                          │
  │   Minor GC (nursery only):                                               │
  │   ────────────────────────                                               │
  │   1. Stop mutator threads                                                │
  │   2. Scan roots (stack, registers, globals)                              │
  │   3. Copy live nursery objects to old gen                                │
  │   4. Update pointers                                                     │
  │   5. Reset nursery                                                       │
  │   6. Resume                                                              │
  │                                                                          │
  │   Major GC (all generations):                                            │
  │   ───────────────────────────                                            │
  │   1. Stop all threads                                                    │
  │   2. Mark all reachable objects                                          │
  │   3. Compact (slide) live objects                                        │
  │   4. Update all pointers                                                 │
  │   5. Resume                                                              │
  │                                                                          │
  │   Server profile: concurrent marking, incremental compaction             │
  │   Edge profile: simple two-space copying (smaller runtime)               │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

### Thread Scheduler

BHC uses **M:N threading**: many lightweight Haskell threads multiplexed onto a small number of OS threads (one per CPU core). This lets programs spawn millions of concurrent tasks without exhausting OS resources.

The scheduler maintains a **run queue per CPU**. When a Haskell thread is ready to execute, it sits in a run queue. An OS thread (called a "capability") pulls threads from its queue and executes them. When a thread blocks (on an MVar, STM transaction, or I/O), it's removed from the run queue and added to a wait list; when the blocking condition resolves, it's moved back.

**Work stealing** balances load across CPUs. If one CPU's queue is empty, it steals threads from busy CPUs. This keeps all cores utilized even with irregular workloads.

The **thread state machine** shows the lifecycle: threads start RUNNABLE, transition to RUNNING when scheduled, and either block (waiting for a resource), finish (completed execution), or get killed (exception or cancellation).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       THREAD SCHEDULER                                      │
└─────────────────────────────────────────────────────────────────────────────┘

  BHC uses M:N threading (many Haskell threads on few OS threads).

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                          │
  │     Haskell Threads (lightweight, millions possible)                     │
  │                                                                          │
  │        T₁    T₂    T₃    T₄    T₅    T₆    T₇    T₈    ...              │
  │         │     │     │     │     │     │     │     │                      │
  │         └──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┘                      │
  │            │     │     │     │     │     │                               │
  │            ▼     ▼     ▼     ▼     ▼     ▼                               │
  │     ┌──────────────────────────────────────────────────┐                 │
  │     │              SCHEDULER                            │                 │
  │     │                                                   │                 │
  │     │   ┌─────────────────────────────────────────┐    │                 │
  │     │   │           Run Queues                     │    │                 │
  │     │   │   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐       │    │                 │
  │     │   │   │ CPU │ │ CPU │ │ CPU │ │ CPU │ ...   │    │                 │
  │     │   │   │  0  │ │  1  │ │  2  │ │  3  │       │    │                 │
  │     │   │   └─────┘ └─────┘ └─────┘ └─────┘       │    │                 │
  │     │   └─────────────────────────────────────────┘    │                 │
  │     │                                                   │                 │
  │     │   Work stealing: idle CPUs steal from busy ones   │                 │
  │     │                                                   │                 │
  │     └──────────────────────────────────────────────────┘                 │
  │                        │     │     │     │                               │
  │                        ▼     ▼     ▼     ▼                               │
  │                                                                          │
  │     OS Threads (Capabilities) - one per CPU core                         │
  │                                                                          │
  │                       C₀    C₁    C₂    C₃                               │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                   THREAD STATE MACHINE                                   │
  │                                                                          │
  │                        ┌─────────┐                                       │
  │           forkIO ────► │ RUNNABLE│ ◄──── wakeup                         │
  │                        └────┬────┘                                       │
  │                             │                                            │
  │                             │ scheduled                                  │
  │                             ▼                                            │
  │                        ┌─────────┐                                       │
  │                        │ RUNNING │                                       │
  │                        └────┬────┘                                       │
  │                             │                                            │
  │           ┌─────────────────┼─────────────────┐                          │
  │           │                 │                 │                          │
  │           ▼                 ▼                 ▼                          │
  │      ┌─────────┐       ┌─────────┐       ┌─────────┐                     │
  │      │ BLOCKED │       │ FINISHED│       │  KILLED │                     │
  │      │ (MVar,  │       │         │       │         │                     │
  │      │  STM,   │       └─────────┘       └─────────┘                     │
  │      │  I/O)   │                                                         │
  │      └─────────┘                                                         │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Module System

BHC's module system enables separate compilation and supports the Haskell package ecosystem.

Each source module compiles to two outputs: an **interface file (.bhi)** containing type information for importers, and an **object file (.o)** containing compiled code. The interface file has everything needed to type-check code that imports this module: exported types, values, classes, instances, and inlinings for cross-module optimization. The object file has the actual machine code.

**Dependency resolution** uses a package database that maps module names to packages. When you write `import Data.Text`, the compiler searches the database, finds the `text` package, loads its interface file, and records the dependency. The linker later ensures all required packages are included.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MODULE SYSTEM                                      │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    COMPILATION UNIT STRUCTURE                            │
  │                                                                          │
  │   Source Module (.hs)                                                    │
  │         │                                                                │
  │         ▼                                                                │
  │   ┌─────────────────────────────────────────────────────────────────┐   │
  │   │                     Interface File (.bhi)                        │   │
  │   │                                                                  │   │
  │   │   - Module name and dependencies                                 │   │
  │   │   - Exported declarations (types, values, classes)               │   │
  │   │   - Inlinings for cross-module optimization                      │   │
  │   │   - Instance declarations                                        │   │
  │   │   - Rewrite rules                                                │   │
  │   │                                                                  │   │
  │   └─────────────────────────────────────────────────────────────────┘   │
  │         │                                                                │
  │         ▼                                                                │
  │   ┌─────────────────────────────────────────────────────────────────┐   │
  │   │                     Object File (.o)                             │   │
  │   │                                                                  │   │
  │   │   - Compiled code                                                │   │
  │   │   - Info tables                                                  │   │
  │   │   - Static closures                                              │   │
  │   │   - Relocations                                                  │   │
  │   │                                                                  │   │
  │   └─────────────────────────────────────────────────────────────────┘   │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    DEPENDENCY RESOLUTION                                 │
  │                                                                          │
  │   Package Database                                                       │
  │   ────────────────                                                       │
  │   ┌──────────────────────────────────────────────────────────┐          │
  │   │  base-4.18.0                                              │          │
  │   │    ├── modules: Prelude, Data.List, Control.Monad, ...   │          │
  │   │    ├── depends: ghc-prim, ghc-bignum                     │          │
  │   │    └── abi-hash: abc123...                               │          │
  │   ├──────────────────────────────────────────────────────────┤          │
  │   │  text-2.0                                                 │          │
  │   │    ├── modules: Data.Text, Data.Text.IO, ...             │          │
  │   │    ├── depends: base, bytestring                         │          │
  │   │    └── abi-hash: def456...                               │          │
  │   └──────────────────────────────────────────────────────────┘          │
  │                                                                          │
  │   Import Resolution:                                                     │
  │   ──────────────────                                                     │
  │   import Data.Text (Text)                                                │
  │     1. Search package DB for module "Data.Text"                          │
  │     2. Find package "text-2.0"                                           │
  │     3. Load interface file                                               │
  │     4. Resolve "Text" in exports                                         │
  │     5. Record dependency                                                 │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Compilation Modes

BHC supports several invocation modes for different workflows:

**One-shot compilation** (`bhc Main.hs`) compiles a single file, loading dependencies from precompiled interface files. This is the simplest mode, useful for quick scripts.

**Make mode** (`bhc --make Main.hs`) automatically discovers and compiles all modules that `Main.hs` depends on. It checks timestamps to only recompile changed modules and compiles independent modules in parallel.

**Cabal integration** (`bhc build`) reads your `.cabal` file, resolves dependencies from Hackage, downloads and builds them, then compiles your project. This is the standard workflow for real projects.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         COMPILATION MODES                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                          │
  │   bhc Main.hs                          One-shot compilation              │
  │        │                                                                 │
  │        ├── Parse Main.hs                                                 │
  │        ├── Load dependencies (from .bhi files)                           │
  │        ├── Type check                                                    │
  │        ├── Optimize                                                      │
  │        ├── Generate code                                                 │
  │        └── Link with RTS → executable                                    │
  │                                                                          │
  │   bhc --make Main.hs                   Dependency-aware compilation      │
  │        │                                                                 │
  │        ├── Build dependency graph                                        │
  │        ├── Check timestamps (.hs vs .bhi vs .o)                          │
  │        ├── Compile changed modules (parallel where possible)             │
  │        └── Link all objects → executable                                 │
  │                                                                          │
  │   bhc build                            Cabal integration                 │
  │        │                                                                 │
  │        ├── Read .cabal file                                              │
  │        ├── Resolve Hackage dependencies                                  │
  │        ├── Download/build dependencies                                   │
  │        ├── Compile project modules                                       │
  │        └── Link → executable/library                                     │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Diagnostic System

BHC provides rich error messages inspired by Rust's compiler diagnostics. Each error includes not just what went wrong, but context about why and suggestions for how to fix it.

A **diagnostic** contains: severity (error, warning, note), a unique code for lookup, the error message, source locations (primary span plus related locations), inline labels pointing at specific code, additional notes explaining context, and fix-it suggestions showing concrete changes.

The example shows a type mismatch error. The message explains the expected vs. found types, the source location points to the offending expression with an inline label, and a help suggestion shows how to fix it with `read`.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DIAGNOSTIC SYSTEM                                    │
└─────────────────────────────────────────────────────────────────────────────┘

  BHC provides rich error messages with source locations and suggestions.

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                      ERROR STRUCTURE                                     │
  │                                                                          │
  │   ┌───────────────────────────────────────────────────────────────┐     │
  │   │  Diagnostic                                                    │     │
  │   │    ├── severity: Error | Warning | Note                        │     │
  │   │    ├── code: E0001, W0042, ...                                │     │
  │   │    ├── message: "Type mismatch..."                            │     │
  │   │    ├── primary_span: file:line:col                            │     │
  │   │    ├── secondary_spans: [related locations]                   │     │
  │   │    ├── labels: [span annotations]                             │     │
  │   │    ├── notes: [additional context]                            │     │
  │   │    └── suggestions: [fix-it hints]                            │     │
  │   └───────────────────────────────────────────────────────────────┘     │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                      EXAMPLE OUTPUT                                      │
  │                                                                          │
  │   error[E0308]: type mismatch                                            │
  │     --> src/Main.hs:15:10                                                │
  │      |                                                                   │
  │   14 |   let x :: Int                                                    │
  │   15 |       x = "hello"                                                 │
  │      |           ^^^^^^^ expected `Int`, found `String`                  │
  │      |                                                                   │
  │      = note: expected type `Int`                                         │
  │                 found type `String`                                      │
  │      = help: consider using `read` to parse the string:                  │
  │      |                                                                   │
  │   15 |       x = read "hello"                                            │
  │      |           ++++                                                    │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Further Reading

- [Get Started](@/get-started.md) - Install BHC and write your first program
- [Compatibility](@/compatibility.md) - Supported Haskell features
- [Profiles](@/profiles.md) - Runtime profile details
- [Numeric](@/numeric.md) - Tensor IR and fusion guarantees
- [Targets](@/targets.md) - Backend targets (native, WASM, GPU)
