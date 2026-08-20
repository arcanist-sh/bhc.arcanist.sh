+++
title = "The Crash Ladder: a Day Inside readMarkdown"
description = "BHC-compiled Pandoc stopped failing at compile time and started failing at runtime — which is progress. Six distinct crashes, each one deeper into the real Markdown parser than the last, starting from a single missing dictionary that shifted every argument one slot."
date = 2026-08-20
template = "blog-post.html"

[extra]
tag = "compiler"
+++

For most of this campaign the Pandoc number has measured compilation:
221 of 221 modules type-check, and as of this week 197 of 221 compile
to native objects. Those numbers move slowly and tell you nothing about
whether the machine code is *right*. This post is about the other
number, the one that moved six times in a day: how far a BHC-compiled
`readMarkdown` gets through an actual document before it dies.

The test harness is deliberately small. A 15-line `Main.hs` reads
`README.md`, calls `readMarkdown`, feeds the result to
`writeHtml5String`, and prints the HTML. We link it against every
object the sweep produces — pandoc's, pandoc-types', and our compiled
copy of parsec — plus the BHC runtime. At the start of the day it
printed the input length and then jumped to address zero. At the end of
the day it was merging parse errors inside the Markdown grammar, which
is a different world.

## Rung one: the null function pointer

The crash we started with looked hopeless in the way that only
tail-call chains can. The program counter was `0x0`. The link register
pointed at `runParserT+108`. Everything between the two had been
destroyed by tail branches, so the backtrace was two frames long and
one of them was the void.

macOS's crash report still has all the registers, and the registers
told the story. The branch target had come out of `x6`, which
`runParsecT` loads from the first word of the parser closure it
dispatches:

```
ldp  x6, x8, [x0]     ; fn ptr, arity — closure header
...
br   x6               ; x6 = 0
```

So the "parser closure" had a zero where its code pointer belongs. We
wrote a sixty-line C signal handler — lldb refuses to launch on this
machine, a separate saga — that dumps registers and walks heap objects
at crash time, linked it into the probe binary, and looked at the thing
being dispatched. It wasn't a closure at all. Tag word zero, a pointer
in the second slot, a fan of record fields behind it. It was the
**ParserState**.

If the state record is sitting where the parser belongs, everything is
off by one slot. And the only thing that shifts every argument of a
call by one slot is a missing leading argument. `runParserT` is
constrained — `Stream s m t => …` — so its compiled form takes a
dictionary first. The call site in `readWithM` wasn't passing one.

## Why the dictionary went missing

Three separate compiler gaps had to line up for that dictionary to
silently vanish, which is why it survived so long.

First, the instance it needed — pandoc's `instance Monad m => Stream
Sources m Char` — could never be *found*, because the module that
defines it had recorded it wrong. BHC splits a multi-parameter instance
head by the class's parameter count, and for an *imported* class that
count defaulted to 1. So `Stream Sources m Char` was serialized into
the interface file as a single flattened type application, `(Sources m)
Char`, which no consumer could ever match against a three-parameter
constraint. The class's parameter list was sitting in the interface
format all along; nobody was reading it.

Second, even with a correct head, the functional-dependency completion
refused the match. At the call site `s` is pinned to `Sources` but `m`
is the caller's own type variable — `readWithM` is polymorphic in its
monad. The completion logic insisted on instances becoming *fully*
concrete, and `Monad m => Stream Sources m Char` never does; `m` stays
open by design. It now merges per position: the instance's concrete
`Char` fills our unknown `t`, and our open `m` stays ours.

Third, when dictionary resolution failed, the call site simply…
proceeded. Applied the function to its value arguments and hoped. That
is the single worst failure mode available, because nothing goes wrong
at the site itself — the corruption surfaces arbitrarily far away, as
a state record being branched through. There is now a warning printed
whenever a dictionary the callee expects cannot be resolved, and the
sweep logs immediately lit up with other sites that had been silently
shifting for weeks.

With the dictionary passed, the probe stopped jumping to zero and
started executing parsec's continuation machinery. First rung climbed.

## Rungs two through six

Each subsequent crash was strictly deeper, and each had the same
pleasant property: a concrete, mechanical cause.

**`errorIsUnknown` received the integer 3.** parsec's
`updateParserState` ends `eok s' s' $ unknownError s'` — application
through `$`. BHC's `$` lowering evaluated the left side as a value,
which two-argument-called a continuation whose physical arity is three.
The third register was never set; the callee read whatever was in
`x3`, which happened to be a 3. The fix folds `f a b $ x` into one
saturated application so the spine collector sees all the arguments.
The under-application had been perfectly camouflaged until the moment
the arguments became correct enough to reach it.

**A `BTreeSet` walked garbage.** `extensionEnabled` calls `Set.member`
on the reader's extension set, and the set was noise. The trail led
back to `emptyExtensions = Extensions mempty`: BHC had no `Monoid`
instance for its builtin `Set`, so `mempty` compiled to a stub that
returns garbage, and the default `ReaderOptions` carried that garbage
in its `readerExtensions` field. Registering builtin
`Semigroup`/`Monoid` instances for `Set` and `Map` (mapping onto the
existing union/empty/unions primitives) was most of the fix — and then
two more layers of the same onion: a nullary builtin used as a value
was being wrapped in an arity-zero closure that nothing would ever
force, and the recorded type constructor was the alias-qualified
`Set.Set`, which didn't match the instance head registered under
`Set`.

**`stub: guard`.** The extension check now *worked*, and
`guardEnabled` promptly hit `Control.Monad.guard`, which BHC had never
implemented — it's not a class method, just an externally-imported
constrained function, so it had quietly become a runtime stub. When
the occurrence type pins the target monad, BHC now synthesizes the
body — `\b -> if b then pure () else empty` — with both methods
resolved at parsec's instances.

**`stub: mappend`.** parsec compares source positions with the
`Ordering` semigroup: `mappend (compare line1 line2) (compare col1
col2)`. `Ordering` had no instance anywhere either. This one is small
enough to synthesize inline: `\x y -> case x of EQ -> y; _ -> x`.

Every one of these fixes was gated the same way: the full 221-module
sweep re-run from a clean database, the 204-test end-to-end suite run
serially, and the probe binary re-linked and re-executed. The sweep
finished the day at 197 of 221 — one module *up*, because the `$` fix
exposed a latent bug in `fromMaybe`'s handling of floating-point
default values whose repair also cleared DocBook, a module that had
been failing since the codegen campaign began.

## The rung we're standing on

The current crash is the best one yet. `sourceLine` faults reading a
`SourcePos` whose pointer value is, on inspection, two ARM
instructions. Somewhere in `parserPlus`'s error-merging continuations,
an `err` slot got captured from a register that was never written —
the same under-application family as the `$` bug, but reached through
a call whose arity mismatch only exists at runtime. The honest fix is
a real partial-application object: when a closure receives fewer
arguments than its recorded arity, the apply machinery must package
what it has instead of calling anyway and letting the callee read
uninitialized registers. That's a small runtime feature, not a patch,
and it's next.

It is worth being precise about what today was and wasn't. `pandoc
README.md -o out.html` does not run yet. What changed is the *kind* of
failure: for the first time, the compiled parser is executing the real
Markdown grammar — threading its state through parsec's CPS machinery,
checking reader extensions, constructing and merging parse errors —
and every failure left is a runtime bug with a register-level
explanation, in a binary we can interrogate one crash at a time. The
ladder only goes down from here.
