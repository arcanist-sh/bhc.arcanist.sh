+++
title = "The Honest Hard Part: Type Classes and Hashable Code"
description = "Content-addressing a language means naming a definition by what it means, not by how it was typed. For a language with type classes, that requires elaborating instance resolution away first. Dictionary passing is that step — and BHC already does it."
date = 2026-06-24
template = "blog-post.html"

[extra]
tag = "engineering"
+++

The [commons](https://arcanist.sh/commons/) makes one move past a library: instead of sharing code you trust by reputation, you share a definition named by what it *means*, with the evidence of what was checked travelling alongside it. Verification stops being something every agent redoes from nothing and becomes something you cite.

The idea is not ours. Unison content-addresses definitions by the hash of their syntax tree. What makes it work is that the name is not the identity — the meaning is. Rename a function, reformat it, flip an argument and flip it back, and it is still the same definition with the same address.

There is an honest hard part in bringing that to Haskell, and it is worth naming plainly: **type classes.**

## Why source text is the wrong thing to hash

Take the most ordinary line of Haskell:

```haskell
render x = show x
```

What does `show` mean here? It depends entirely on the type of `x`, and on which `Show` instance is in scope for that type. The same three characters resolve to integer formatting in one context, list formatting in another, and your own hand-written instance in a third. The source text is identical; the meaning is not.

The reverse is also true. Two definitions that read completely differently can resolve to exactly the same behaviour once their instances are pinned down.

So a hash over the syntax tree — Unison's move — does not survive contact with type classes. It would give one address to several different meanings, and several addresses to one. To name a Haskell definition by what it does, you first have to remove the ambiguity that instance resolution leaves in the source. You have to make the implicit explicit.

## Dictionary passing is exactly that step

This is not exotic. It is how Haskell's class system has been compiled for thirty years, and BHC does it in [`bhc-hir-to-core`](https://github.com/arcanist-sh/bhc).

A constraint becomes an argument. This signature:

```haskell
show :: Show a => a -> String
```

elaborates to a function that takes the `Show` dictionary explicitly — a record of the methods that the instance provides:

```text
show :: ShowDict a -> a -> String
```

At the call site, the instance the type checker selected is passed in as a value. `show x` becomes `show showDict_Int x`. After this pass there is no ambient resolution left to do: every class method is a plain function call against a dictionary that is right there in the term. The deriving machinery feeds in the same way — a derived `Eq` or `Show` is just an instance whose dictionary BHC constructs — and existential constructors carry their dictionaries as ordinary fields, so a value that packed up a constraint can still resolve its methods after it is unpacked.

The point for the commons is this: **the elaborated Core, not the source, is the canonical meaning.** Two source files that differ only in style elaborate to the same Core. Two that look identical but resolve different instances do not. That is precisely the property an address needs. The pass we built to generate code turns out to be the pass that makes a definition nameable by what it means.

We are honest about the edge. Dictionary passing is complete for the cases where the instance is determined by the argument types, which is the overwhelming majority of real code. Where the instance is determined by the *result* type, or sits behind higher-rank quantification, the dictionary cannot be chosen from the arguments alone — it needs evidence threaded down from the type checker. That residue is known, scoped, and not yet wired through. It is the difference between "most of Haskell is addressable today" and "all of it is," and we would rather say which one we mean.

## The other half: recompile and get the same thing

A canonical meaning gives you an address. It does not yet give you the load-bearing attestation underneath the commons, which is reproducibility: anyone can recompile a definition and confirm they reach the same artifact, so nothing has to be trusted on reputation. For that, the backends have to agree.

BHC compiles the same Core to native code via LLVM and to WebAssembly. Two backends are two chances to disagree — and per-backend test suites cannot catch a disagreement, because each one only checks a fixture against its own expected output, on the backends its config happens to list.

So we built a [differential harness](https://github.com/arcanist-sh/bhc) that runs every fixture through *both* backends, diffs their output, and classifies each divergence by which backend matches the expected result: `agree`, `native right`, `wasm right`, or a backend that failed outright. It turns "I happened to notice native prints lists with spaces" into a repeatable sweep.

The correctness divergences it surfaced are closed. Native and WASM agree on observable output across the fixture suite; what remains is WASM coverage — features it does not implement yet — not the two backends silently disagreeing on a feature they both claim. That is the discipline reproducible addressing rests on, arrived at because we wanted two trustworthy targets, not because a manifesto asked for it.

## Where this leaves the commons

We are not claiming BHC content-addresses anything today. It does not. Hashing the elaborated Core, emitting a manifest of what was checked, and binding the runtime profile into the address are [exploratory items on the roadmap](@/roadmap.md), post-v1, and listed there honestly.

What we are claiming is narrower and, we think, more useful. The two prerequisites that sounded hardest when we wrote the commons down — a canonical meaning for a definition in a language with type classes, and agreement across the targets it lowers to — are not greenfield research. They are passes BHC already runs, built for ordinary compiler reasons, that happen to be exactly what content-addressing needs.

The commons is the direction, not yet the destination. But the honest hard part has a head start.
