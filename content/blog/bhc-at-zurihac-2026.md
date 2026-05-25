+++
title = "BHC at Zurihac 2026"
description = "We'll be at Zurihac 2026. Here is what compiles today, what we'll have running at the booth, and the questions that are most useful to bring."
date = 2026-05-25
template = "blog-post.html"

[extra]
tag = "events"
+++

[Zurihac 2026](https://zfoh.ch/zurihac2026/) — 6–8 June, OST Eastern
Switzerland University of Applied Sciences, Rapperswil-Jona — is in about
two weeks, and BHC will be there. This post is the short pre-event
status: what we have running, what we'll demo, and what we'd most like
to talk about.

If you're skimming, the three useful links are
[install](@/_index.md#try-it-now), the [status page](@/status.md), and the
[issue tracker](https://github.com/arcanist-sh/bhc/issues).

## What works today

BHC is in beta. The native code path compiles real Haskell:

- **Pandoc — `bhc check`: 78 / 221 modules pass.** Every source file in
  `pandoc-3.6.4/src/` produces a syntactically valid AST; 78 of them survive
  through lowering and type-checking. The number moves regularly; the
  [status page](@/status.md) is the live figure.
- **Zentinel agent — 10 / 10 modules.** `zentinel-agent-policy` is the
  canonical real-library target. The full `Zentinel.Agent.Policy.*` graph
  goes through the multi-module pipeline and produces object code. Story
  in [this earlier post](@/blog/zentinel-ten-modules-compile.md).
- **Native end-to-end — 6 / 6.** Hello world, arithmetic, fibonacci, IO
  sequencing, recursion, the usual suite.
- **Profiles working:** `default`, `numeric` (with Tensor IR fusion and SIMD
  vectorisation), `server` (structured concurrency with STM, cancellation,
  deadlines).

WebAssembly and GPU backends have working code but no end-to-end demos to
show off yet — binaries validate, output tests don't. We'll talk about
those if you ask, but won't be running them at the booth.

## At the booth

We'll have three things prepared to demo on demand:

1. **Native compilation in 30 seconds.** `bhc Main.hs -o main && ./main` on
   real Haskell with type classes, ADTs, and IO. Boring on purpose —
   "boring" is the point.
2. **`bhc check` against a Pandoc module.** Watching real Haskell from a
   well-known project succeed (or, just as honestly, watching it fail and
   reading the diagnostic).
3. **Kernel fusion report.** `bhc --profile=numeric --kernel-report` on
   `sum (map (*2) [1..1_000_000])`. This is the thing GHC users don't
   typically have in front of them.

A printed install card will be at the table with the macOS and Linux
install lines, the [status page](@/status.md) URL, and a QR back to the
[github repo](https://github.com/arcanist-sh/bhc). Stickers permitting.

## What we'd most like to hear

A few questions are particularly useful coming from the Haskell community:

- **"Does it compile *my* code?"** Bring a small program, a single module
  is fine. If `bhc check` fails on it, that becomes an issue and very
  likely a fix within days. The Pandoc number moves precisely because real
  codebases keep exposing specific gaps.
- **"What about extension X?"** The [compatibility charter](@/compatibility.md)
  documents the GHC2021/GHC2024 subset BHC currently supports. Edge-case
  reports are valuable.
- **"How does the numeric profile compare?"** This is the differentiator
  worth pushing on. The fusion guarantees and kernel reports are not
  rhetoric — they fail compilation when fusion is violated.
- **"Where is the gap between you and GHC?"** Honest answer: standard
  library coverage, the type-class story against partially-typed stubs,
  WASM lowering, and bare-metal codegen. We won't pretend otherwise.

Questions that are less useful: "when will it support Template Haskell?"
or "when will it be GHC-compatible for arbitrary code?" — TH is explicitly
out of scope and the *compatibility charter* exists to keep the answer
honest about what subset we're aiming for.

## How to try it

```bash
# macOS / Linux
curl -fsSL https://arcanist.sh/bhc/install.sh | sh

# Or via Homebrew
brew install arcanist-sh/tap/bhc

# Hello world
echo 'main = putStrLn "Hello from BHC!"' > hello.hs
bhc hello.hs -o hello
./hello
```

If you find a thing that should compile and doesn't, the highest-value
report shape is: a minimal `.hs` file (single module, no external deps),
the BHC version, and the diagnostic. That's enough for a fast turnaround.

See you in Rapperswil.
