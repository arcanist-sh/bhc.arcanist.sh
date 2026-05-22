+++
title = "BHC and the Verification Camp"
description = "agentlanguages.dev makes the emerging field of AI-authored programming languages visible. BHC belongs near the verification camp, but it makes a different bet from agent-native languages like Vera."
date = 2026-05-22
template = "blog-post.html"

[extra]
tag = "positioning"
+++

Alasdair Allan, the creator of [Vera](https://veralang.dev/), has started [agentlanguages.dev](https://agentlanguages.dev/), a public catalogue of programming languages designed for AI agents to write.

The catalogue matters because it changes the argument. Before, this space looked like a handful of disconnected experiments. Vera over here. Aver somewhere else. MoonBit from a more industrial angle. Orchestration systems that look less like languages and more like execution environments. Put them next to each other and the pattern becomes harder to dismiss.

Alasdair describes three camps:

- **syntactic**: reduce representational ambiguity so models have fewer ways to get lost
- **verification**: make generated programs mechanically checkable
- **orchestration**: treat agent coordination, not syntax, as the primary problem

BHC belongs near the verification camp, but with one important boundary: BHC is not a new agent-native language.

It is a compiler for Haskell.

## The BHC bet

BHC starts from a conservative premise: Haskell already has many of the properties that matter in an AI-authored programming environment.

Pure functions make dependencies visible. Types carry meaning. Algebraic data types make domain structure explicit. Composition keeps transformations local. The language already pushes programmers away from incidental mutation and toward describing what a value is.

Those properties matter more when generation gets cheap.

If a human writes every line by hand, familiarity and typing comfort carry a lot of weight. If an agent generates most of the implementation, the language should optimize for what can be checked after generation: type alignment, effect boundaries, invariants, reproducible builds, and predictable lowering.

That is where Haskell remains interesting.

The problem is not the semantic shape of the language. The problem is the operational surface around it: toolchain setup, diagnostics, runtime profiles, target selection, compiler feedback, and the cost of getting from source to a trusted executable.

BHC is one part of that surface. [hx](https://arcanist.sh/hx/) is the other.

hx gives agents and humans a coherent development loop. BHC gives Haskell another compiler path: compatibility-first, with explicit runtime profiles, multiple targets, and room for numeric and tensor-oriented compilation strategies.

## Adjacent to Vera, not competing with it

Vera takes the clean-sheet route. It asks what a language should look like if it is designed around model cognition from the beginning. That leads to contracts, typed effects, solver-backed verification, and De Bruijn-style slot references instead of variable names.

That is a sharp design.

BHC starts from the other side. It asks whether an existing language with the right semantic density can be made operationally good enough for the same era.

The difference is useful:

```text
Vera: design the language around the agent.
BHC: preserve the semantic substrate; rebuild the compiler and tooling around it.
```

Those projects may eventually compete for attention, users, or deployment budgets. They do not compete philosophically. Vera is a bespoke agent-native language. BHC is a Haskell-first infrastructure bet.

Both care about the same failure mode: plausible generated code is cheap, and plausible code is not the same thing as correct code.

## Verification is the crowded camp for a reason

The verification camp is crowded because anyone who works with coding agents for long enough sees the same failure pattern.

The model can produce syntax. It can produce plausible architecture. It can imitate style. What it cannot do reliably is deserve trust without a checking layer.

That changes the job of the compiler.

A compiler for AI-authored software should not just translate source into executable code. It should reject ambiguity, preserve meaning, expose obligations, and give feedback that both humans and agents can act on.

For BHC, that means:

- compatibility modes that make the baseline explicit
- runtime profiles that make execution contracts explicit
- diagnostics that explain failures in terms of the program being compiled
- numeric and tensor lowering that preserves structure instead of erasing it too early
- future proof and verification hooks where the compiler has enough semantic information to expose them

The model does not need to be trusted. The artifact needs to be checkable.

That is the center of the verification camp, and it is the part BHC cares about.

## Verifiable compute is the deeper target

The near-term story is AI writing source code. The deeper story is verifiable compute.

For BHC, the long-term target is not simply "agents write Haskell." That is too small. The more interesting target is semantically rich Haskell lowered into execution forms that can be trusted: native code, WebAssembly, and eventually GPU-oriented numeric pipelines.

GPU compute today is usually expressed through systems that optimize for throughput first and meaning second. That is understandable. GPUs are throughput machines. But when compute becomes economically and operationally critical, meaning starts to matter again.

What ran? Under which assumptions? With which shapes? Which transformations were fused? Which effects were allowed? Which obligations were checked before execution, and which remained runtime contracts?

A functional, typed representation gives the compiler more structure to preserve through that pipeline. That structure is useful for optimization, but it is also useful for auditability.

This is where BHC touches the same future Vera points at from a different angle: generated artifacts that are checked by compilers and solvers, lowered into efficient execution targets, and trusted because the artifact carried enough meaning to inspect.

## What agentlanguages.dev makes visible

agentlanguages.dev does not prove which camp will win. It proves the pressure is real.

Independent projects arrived in the same neighborhood at roughly the same time. That usually means the underlying constraint changed. Here, the constraint is authorship.

The old language-design question was:

```text
What can humans write, read, and maintain?
```

The new question is:

```text
What can agents generate, compilers verify, runtimes constrain,
and humans audit when needed?
```

Human readability does not disappear. It changes position. It becomes part of auditability rather than the only authoring constraint.

That is the shift the catalogue makes visible.

BHC is not trying to be "a language designed for AI agents to write." It is trying to make Haskell a serious substrate for the same world: explicit where it matters, compatible where possible, and structured enough that generated software can be checked before it is trusted.
