+++
title = "The Writer Runs End to End"
description = "Pandoc's HTML writer executes a three-deep monad transformer stack — StateT over ExceptT over StateT — and BHC now runs the whole thing to completion. Getting there meant chasing one crash that kept moving one monadic action deeper into the writer's do-block, through the transformer machinery, the monomorphizer, and a string comparison that read a struct as a linked list."
date = 2026-09-15
template = "blog-post.html"

[extra]
tag = "compiler"
+++

Two posts ago, [the crash ladder](/blog/the-crash-ladder/) was about the
*reader* — how far a BHC-compiled `readMarkdown` climbs through an actual
document before it dies. This post is about the other half of the same
15-line test harness: `writeHtml5String`. As of this week it no longer
dies. `runIOorExplode (writeHtml5String def doc)` runs to completion —
`WRITER_START`, then the writer's entire computation, then `WRITER_END`,
then exit 0. No jump to address zero.

That sentence took a while to earn, and the reason is worth spelling out,
because it is the reason the pandoc campaign has always been about the
runtime and not the type checker.

## The spine is a three-deep transformer stack

Pandoc's writers do not run in `IO`. They run in `PandocMonad m`, and the
concrete monad the top-level runner picks is `PandocIO`, which is a newtype
over

```haskell
ExceptT PandocError (StateT CommonState IO)
```

The HTML writer then stacks *another* `StateT` on top of that to carry its
own `WriterState`, so the computation `pandocToHtml` actually executes in

```haskell
StateT WriterState (ExceptT PandocError (StateT CommonState IO))
```

Three transformers over `IO`. Every `>>=` in that do-block has to thread
two independent pieces of state and let the `ExceptT` short-circuit on an
error, and every action is a closure with a specific arity that encodes
exactly that threading. GHC compiles all of this away into tight code you
never think about. A clean-slate compiler has to *decide* how each of
those actions is represented, and then be right about it everywhere, or
the shapes stop fitting together and something reads a pointer where a tag
belongs.

BHC represents that stack as a three-argument closure

```
\(self, s1, s2) -> (Either e (a, s1'), s2')
```

— the writer's state, the common state, and the error layer, all in one
value. Get the arity of *any* action in the do-block wrong and its caller
reads the returned pair at the wrong offset. The failure is always the
same on the outside: a null `Either` two frames deep, in a generated
function with a name like `bhc_stes_then`. The failure is never the same
on the inside.

## The crash that kept moving one action deeper

So this was a crash ladder too, except the rungs were the *statements of
one do-block*. `pandocToHtml` opens like this:

```haskell
pandocToHtml opts (Pandoc meta blocks) = do
  lift $ setupTranslations meta
  let slideLevel = ...
  modify $ \st -> st{ stSlideLevel = slideLevel }
  metadata <- metaToContext opts ...
  ...
```

Every fix pushed the segfault down one line, and each new line was a
different bug wearing the same null-`Either` costume:

- **`lift $ setupTranslations meta`.** The `lift` was materialised as a
  first-class value — a curried closure the writer builds and passes
  around — and the value-position path lowered it to the *ReaderT* lift
  regardless of the actual stack. There is no `ReaderT` anywhere in the
  writer. Called with the three-argument convention it handed back an
  unevaluated closure where an `(Either, state)` pair belonged.

- **`setupTranslations` itself.** It is polymorphic in the monad, so it
  had to be *specialized* at the concrete stack before its own `>>=`,
  `pure`, and `fmap` could be resolved to the right operators. Those
  operators are reached through the class's superclass chain
  (`PandocMonad ⊃ Monad ⊃ Applicative ⊃ Functor`), and the monomorphizer
  only knew how to rewrite the direct `Monad` case. Teaching it the
  superclass hops — naming the extracted dictionary after its class so the
  rewrite could recognise a `Monad` dictionary reached through
  `PandocMonad` — is a general improvement, not a pandoc patch: any
  method reached through a superclass now resolves in a specialized clone.

- **A string comparison.** With the monad plumbing correct,
  `setupTranslations` finally ran its own body — and took the *wrong*
  branch of `case lookupMetaString "lang" meta of "" -> …`. The scrutinee
  was a `Text`, which BHC represents as a struct with a length and a byte
  buffer. The pattern comparison read that struct as a `[Char]` linked
  list, walked off the header, and never matched `""`. The fix is
  type-directed: a `Text` scrutinee now compares against a string-literal
  pattern by reading its actual bytes. This is the same bug that made
  `pandoc --list-input-formats` print blank lines — the format names are
  overloaded-string `Text` literals.

- **`modify`.** And then the last one. `modify` in the writer operates on
  the outer `WriterState`, so it needs the three-argument stes form; the
  value-position builder emitted the two-argument `StateT`-over-`IO` form
  unconditionally, and it was called with three arguments. Same story for
  `put` and `gets`. Route them by stack and the crash is gone — not moved
  down a line, *gone*. The do-block runs to the end.

None of these are surprising in hindsight, and that is the point. Each one
is a place where a representation decision had exactly one correct answer
and the code had a different one, and the only way to find them was to run
real code that exercised the combination.

## What "runs" does and does not mean

The writer executes. The transformer machinery is complete: `evalStateT`,
`runExceptT`, the binds, the lifts, the state operations, the
specialization of every polymorphic-monad helper in the chain — all of it
threads correctly from `main` down through `runIOorExplode`,
`writeHtmlString'`, `pandocToHtml`, `setupTranslations`, and back out with
a value.

The value is an empty document.

That is not a monad bug, and it is worth being precise about why. Pandoc's
HTML writer does not build strings; it builds a `blaze-html` `Markup`
tree — `H.p`, `toHtml`, `H.em` — and renders it at the end. BHC does not
implement `blaze-html` yet, so every one of those combinators is currently
a stub that produces nothing. The writer faithfully runs the entire
computation and hands back the empty markup the stubs gave it. The spine
holds; the muscles it is supposed to move aren't attached yet.

So the honest scoreboard: the hard, compiler-shaped problem — running a
non-trivial monad transformer stack end to end, with polymorphic helpers
specialized at a concrete monad — is solved. The remaining problem is
library coverage: a `Markup` type, its combinators, and a renderer. That
is a different kind of work, and it is the next thing on the bench.

Every fix along the way landed with the gates green — 2828 workspace
tests, the differential suite against GHC at 219 agree / 0 diverge, and
the full 221-of-221 pandoc sweep — because a compiler that is right on a
three-deep transformer stack but wrong on `case x of ".md" -> …` is not
actually right. Correct by construction has to mean all the way down.
