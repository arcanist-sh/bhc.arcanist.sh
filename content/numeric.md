+++
title = "Numeric Performance"
description = "Predictable performance for numeric workloads"
template = "page.html"
+++

# Numeric Performance

BHC's numeric profile provides predictable performance for numeric workloads through a tensor-native optimization pipeline.

## What We Provide

- Predictable performance in numeric workloads
- A tensor-native optimization pipeline
- Guaranteed fusion patterns for standard containers in numeric profile
- Reduced heap activity in hot loops via arenas and pinned buffers

## How It Works

In Numeric profile, BHC prioritizes:

### Strict-by-default evaluation
Fewer thunks, fewer surprises. Hot paths evaluate strictly to avoid accumulating unevaluated computations.

### Unboxed numerics
`Int`, `Double`, and other numeric types use unboxed representations by default. No pointer chasing for arithmetic.

### Flat arrays
Arrays are flat, contiguous memory. No indirection through boxed elements.

### Tensor IR stage
An intermediate representation that understands:
- Shapes and dimensions
- Strides and layouts
- Contiguity requirements

### Fusion as a contract
Core patterns fuse reliably:
- `map . map` → single pass
- `zip` combinations → single traversal
- `fold` after `map` → no intermediate allocation

This is a contract, not a best-effort optimization. If fusion should happen and doesn't, that's a bug.

> **Status (September 2026):** this contract is met in the Tensor IR and Loop IR stages, which pass their internal tests. It is not yet realised in native code generation: the native backend does not consume the fused Loop IR, so a numeric-profile build currently produces the same code and the same timing as the default profile. "Guaranteed" is the standard we hold ourselves to, not a description of today's binaries. See the [status page](@/status.md).

### SIMD and parallel lowering
When legal and profitable:
- Vectorized operations using CPU SIMD instructions
- Parallel execution of independent loop iterations
- Automatic tiling for cache efficiency

### GPU acceleration
For large-scale parallelism:
- Apple Metal runs on Apple-silicon GPUs today; CUDA (NVIDIA) and ROCm (AMD) backends are mock-validated pending hardware
- Tensor IR operations lower to GPU kernels
- Automatic host/device memory management
- Kernel fusion across operations

See [Target Backends](@/targets.md) for GPU target details.

### Pinned buffers
Large arrays and FFI buffers use pinned, non-moving memory:
- No GC relocation during computation
- Safe to pass to C code
- Predictable memory layout

## Usage

### Enable numeric profile

```bash
bhc --profile=numeric Main.hs
```

### Per-module

```haskell
{-# OPTIONS_BHC -profile=numeric #-}
module HotPath where

-- | Performance-critical signal processing module.
-- The numeric profile ensures all operations here compile to
-- tight loops with no unexpected heap allocations.

import qualified Data.Vector.Unboxed as V

-- | Apply a low-pass filter to a signal.
-- In numeric profile, this entire pipeline fuses into a single loop.
-- No intermediate vectors are allocated between map/filter/fold.
lowPassFilter :: Double -> V.Vector Double -> V.Vector Double
lowPassFilter cutoff signal = V.zipWith blend signal smoothed
  where
    -- Exponential moving average for smoothing
    smoothed = V.scanl' (\acc x -> acc + cutoff * (x - acc)) 0 signal
    blend original filtered = 0.7 * original + 0.3 * filtered

-- | Compute signal energy (sum of squares).
-- With numeric profile, this is a single fused traversal:
-- no intermediate vector from 'map', direct accumulation.
signalEnergy :: V.Vector Double -> Double
signalEnergy = V.foldl' (+) 0 . V.map (\x -> x * x)

-- | Normalize a signal to unit energy.
-- The compiler recognizes this pattern and avoids
-- multiple traversals of the input vector.
normalize :: V.Vector Double -> V.Vector Double
normalize v = V.map (/ norm) v
  where
    norm = sqrt (signalEnergy v)
```

### With BHC extensions

```haskell
{-# LANGUAGE BHC.TensorIR #-}
{-# LANGUAGE BHC.StrictDefault #-}
module Compute where

-- | Tensor computation module using BHC's specialized IR.
-- BHC.TensorIR enables shape-aware optimizations: the compiler
-- tracks matrix dimensions through operations and generates
-- cache-efficient blocked algorithms automatically.
--
-- BHC.StrictDefault makes all bindings strict by default,
-- eliminating thunk buildup in numeric code.

import BHC.Tensor (Matrix, Vector, Scalar)
import qualified BHC.Tensor as T

-- | General matrix multiplication with shape tracking.
-- The compiler verifies dimension compatibility at compile time
-- and selects optimal block sizes based on the target architecture.
matmul :: Matrix Double -> Matrix Double -> Matrix Double
matmul a b = T.contract a b  -- Einstein summation under the hood

-- | Compute the Frobenius norm of a matrix.
-- This fuses into a single pass: square each element,
-- sum all values, take the square root.
frobeniusNorm :: Matrix Double -> Scalar Double
frobeniusNorm m = T.sqrt (T.sum (T.map (\x -> x * x) m))

-- | Softmax activation function for neural network layers.
-- Numeric stability: subtract max before exp to avoid overflow.
-- The compiler fuses the three traversals (max, map exp, sum) optimally.
softmax :: Vector Double -> Vector Double
softmax v = T.map (/ total) exps
  where
    maxVal = T.maximum v              -- First pass: find max
    exps   = T.map (\x -> exp (x - maxVal)) v  -- Second pass: shifted exp
    total  = T.sum exps               -- Third pass: normalization
    -- BHC fuses these into two passes (max, then exp+sum+divide)
```

## Example: Vector Operations

```haskell
{-# OPTIONS_BHC -profile=numeric #-}
module VectorOps where

-- | Demonstrating BHC's fusion guarantees with real-world vector operations.
-- Every function here compiles to a single loop with no intermediate allocations.

import qualified Data.Vector.Unboxed as V
import Data.Vector.Unboxed (Vector)

-- | Statistical summary of a dataset.
-- Despite appearing to traverse the vector multiple times,
-- BHC's fusion guarantees combine this into minimal passes.
data Stats = Stats
  { mean     :: !Double
  , variance :: !Double
  , minVal   :: !Double
  , maxVal   :: !Double
  } deriving Show

-- | Compute statistics in a single pass using a fold.
-- The strictness annotations (!) ensure no thunks accumulate.
computeStats :: Vector Double -> Stats
computeStats v = Stats m var lo hi
  where
    n  = fromIntegral (V.length v)
    -- Single traversal computing sum, sum of squares, min, and max
    (total, totalSq, lo, hi) = V.foldl' accumulate (0, 0, inf, -inf) v
    accumulate (!s, !sq, !mn, !mx) x = (s + x, sq + x*x, min mn x, max mx x)
    inf = 1/0  -- Positive infinity as initial min
    m   = total / n
    var = (totalSq / n) - m * m

-- | Weighted moving average with configurable window.
-- Fusion ensures the nested operations don't create intermediate vectors.
weightedMovingAvg :: Vector Double -> Vector Double -> Vector Double
weightedMovingAvg weights signal = V.generate (V.length signal) computeAt
  where
    wlen = V.length weights
    wsum = V.sum weights
    computeAt i
      | i < wlen - 1 = signal V.! i  -- Not enough history, pass through
      | otherwise    = weightedSum / wsum
      where
        -- This inner computation is inlined and optimized
        weightedSum = V.sum $ V.zipWith (*) weights $ V.slice (i - wlen + 1) wlen signal

-- | Element-wise operations that fuse completely.
-- The chain: filter -> map -> map -> sum becomes ONE loop.
pipeline :: Vector Double -> Double
pipeline = V.sum              -- 4. Sum all remaining values
         . V.map sqrt         -- 3. Take square root
         . V.map (\x -> x * x) -- 2. Square each value
         . V.filter (> 0)     -- 1. Keep positive values only
```

## Example: Matrix Computation

```haskell
{-# LANGUAGE BHC.TensorIR #-}
module MatrixOps where

-- | Linear algebra operations with BHC's tensor-aware compiler.
-- The TensorIR extension enables:
--   - Compile-time shape verification
--   - Automatic blocking for cache efficiency
--   - SIMD vectorization where applicable
--   - Optional GPU offloading for large matrices

import BHC.Tensor

-- | LU decomposition with partial pivoting.
-- The compiler tracks matrix dimensions and generates
-- efficient in-place updates where safe.
luDecompose :: Matrix Double -> (Matrix Double, Matrix Double, Vector Int)
luDecompose a = (lower, upper, pivots)
  where
    n = rows a
    -- In-place decomposition with pivot tracking
    (lower, upper, pivots) = runST $ do
      -- Mutable operations here are optimized to avoid copies
      work <- thaw a
      piv  <- newVector n
      forM_ [0..n-1] $ \k -> do
        -- Find pivot (compiler vectorizes this search)
        p <- maxIndex (column work k)
        swapRows work k p
        writeVector piv k p
        -- Eliminate below diagonal (blocked for cache)
        forM_ [k+1..n-1] $ \i -> do
          factor <- readMatrix work i k / readMatrix work k k
          writeMatrix work i k factor
          forM_ [k+1..n-1] $ \j -> do
            val <- readMatrix work i j
            kval <- readMatrix work k j
            writeMatrix work i j (val - factor * kval)
      freeze3 work piv

-- | Solve Ax = b using pre-computed LU decomposition.
-- Forward and backward substitution with automatic vectorization.
luSolve :: (Matrix Double, Matrix Double, Vector Int) -> Vector Double -> Vector Double
luSolve (lower, upper, pivots) b = x
  where
    -- Apply permutation from pivoting
    pb = applyPermutation pivots b
    -- Forward substitution: Ly = Pb
    y = forwardSub lower pb
    -- Backward substitution: Ux = y
    x = backwardSub upper y

-- | Cholesky decomposition for symmetric positive-definite matrices.
-- BHC verifies symmetry at compile time when possible.
cholesky :: Matrix Double -> Matrix Double
cholesky a = lower
  where
    n = rows a
    lower = generate (n, n) $ \(i, j) ->
      if j > i then 0  -- Upper triangle is zero
      else if i == j then
        -- Diagonal: sqrt(a_ii - sum of squares in row)
        sqrt (a ! (i, i) - sumSquares lower i)
      else
        -- Below diagonal
        (a ! (i, j) - dotProduct (row lower i) (row lower j)) / (lower ! (j, j))

    sumSquares m i = sum [m ! (i, k) ^ 2 | k <- [0..i-1]]
    dotProduct r1 r2 = sum (zipWith (*) (toList r1) (toList r2))
```

## What We Don't Promise

We avoid overpromising:

- We don't claim "faster than X across the board"
- We don't claim "GC-free Haskell"
- We don't claim "best-in-class for all workloads"

Numeric profile is optimized for numeric workloads. General-purpose code may perform the same or better with the default profile.

## Benchmarks

BHC publishes reproducible benchmarks for numeric workloads. See the [benchmarks repository](https://github.com/arcanist-sh/bhc/tree/main/benchmarks) for methodology and results.

We benchmark against:
- GHC with equivalent optimizations enabled
- Baseline implementations in other languages

All benchmarks include:
- Full source code
- Build instructions
- Hardware specifications
- Statistical methodology
