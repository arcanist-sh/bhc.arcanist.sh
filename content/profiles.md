+++
title = "Runtime Profiles"
description = "Same language, different compilation and runtime contracts"
template = "page.html"
+++

# Runtime Profiles

BHC supports runtime profiles: the same source language, different compilation and runtime contracts. Profiles are explicit and localizable (package/module).

## How Profiles Work

- Profiles do not "rename the language"
- Profiles are explicit: you choose them; nothing silently changes
- Profile differences are documented and testable
- You can select profiles per package or per module

## Available Profiles

### default

General-purpose Haskell compilation. Focuses on correctness and broad compatibility.

```bash
bhc Main.hs
# or explicitly:
bhc --profile=default Main.hs
```

Best for:
- Library code
- Applications prioritizing compatibility
- Code that needs to work with both BHC and GHC

### server

Structured concurrency, predictable latency, observability hooks.

```bash
bhc --profile=server Main.hs
```

Features:
- Cancellation and deadlines for concurrent operations
- Scoped resource management
- Tracing and observability hooks
- Predictable scheduling behavior

Best for:
- Web services
- API servers
- Long-running daemons
- Applications needing observability

### numeric

Strict-by-default hot paths, unboxed-first numerics, tensor/array lowering, fusion guarantees, SIMD + parallel loops.

```bash
bhc --profile=numeric Main.hs
```

Features:
- Strict-by-default evaluation in hot paths (fewer thunks, fewer surprises)
- Unboxed numerics and flat arrays by default
- Array/tensor IR stage that understands shapes/strides/layouts
- Fusion as a contract for core patterns (map/zip/reduce)
- SIMD and parallel lowering when legal and profitable
- Pinned/non-moving buffers for FFI and large arrays

Best for:
- Scientific computing
- Data processing pipelines
- Machine learning workloads
- Performance-critical numeric code

See [Numeric Performance](@/numeric.md) for details.

### edge

Smaller runtime footprint and restricted environment assumptions.

```bash
bhc --profile=edge Main.hs
```

Features:
- Minimal runtime size
- Works within WASI constraints
- Reduced memory overhead
- Suitable for sandboxed execution

Best for:
- WebAssembly targets
- Serverless functions
- Size-constrained deployments

## Planned Profiles

Two more profiles are designed but not yet selectable:

- **realtime**: bounded pauses and per-frame arena allocation for games, audio and control loops. The incremental collector and pause measurement exist as runtime modules; the profile is not wired into the compiler yet.
- **embedded**: no garbage collector and static allocation for microcontrollers and bare-metal targets. A no-GC mode and static allocator exist; bare-metal code generation is deferred.

They will be documented here, with their contracts, when they can be selected.

## Selecting Profiles

### Command line

```bash
bhc --profile=numeric Main.hs
```

### Per-package (in cabal file)

```cabal
-- In your .cabal file
ghc-options: -profile=server
```

### Per-module

```haskell
{-# OPTIONS_BHC -profile=numeric #-}
module HotPath where

-- | This module handles the performance-critical inner loop of our application.
-- By specifying the numeric profile at the module level, we get:
--   - Strict evaluation by default (no thunk buildup)
--   - Unboxed numeric types (no pointer indirection)
--   - Guaranteed fusion for map/filter/fold chains
--
-- The rest of our application can use the default profile for flexibility,
-- while this module gets the specialized numeric treatment.

import qualified Data.Vector.Unboxed as V
import Data.Vector.Unboxed (Vector)

-- | Process sensor readings with guaranteed performance characteristics.
-- In numeric profile, this entire pipeline fuses into a single loop.
processSensorData :: Vector Double -> (Double, Double, Int)
processSensorData readings = (avg, peak, outlierCount)
  where
    -- Remove invalid readings (negative values are sensor errors)
    valid = V.filter (>= 0) readings

    -- Compute statistics in a single pass
    n    = V.length valid
    avg  = V.sum valid / fromIntegral n
    peak = V.maximum valid

    -- Count outliers (values more than 3 standard deviations from mean)
    sumSq    = V.foldl' (\acc x -> acc + x*x) 0 valid
    variance = (sumSq / fromIntegral n) - avg * avg
    stdDev   = sqrt variance
    outlierCount = V.length $ V.filter (\x -> abs (x - avg) > 3 * stdDev) valid
```

## Profile Combinations

Profiles are mutually exclusive at the module level. A module compiles with exactly one profile. Different modules in the same package can use different profiles.

## Compatibility

All profiles compile the same Haskell source language. The differences are:
- Compilation strategy (strictness, unboxing decisions)
- Runtime behavior (scheduling, allocation)
- Available runtime features (tracing, concurrency primitives)

Code compiled with different profiles can interoperate, subject to documented constraints.
