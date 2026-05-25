+++
title = "Four Parser Fixes, One Module"
description = "A week of chasing parser correctness bugs in BHC. Net effect on the Pandoc number: one module. Why each fix was still worth landing — and what the data says about where the real work is."
date = 2026-05-25
template = "blog-post.html"

[extra]
tag = "compiler"
+++

The current Pandoc check number is 78 out of 221 modules. A week ago it
was 77. Four parser correctness bugs landed in between, none of them
individually responsible for the move. This post is the story of those
four bugs, and why we shipped them anyway.

## The setup

`pandoc check` is the integration target. For each of the 221 source files
under `pandoc-3.6.4/src/`, BHC runs lex → parse → lower → type-check, and
records whether the module makes it through. A failure at any stage
counts as a failure for the module, with the count broken down by stage
in the run summary.

When you're trying to move that number, the cheap experiment is to ask
which modules have the fewest errors. If a module fails with one
`unbound variable` error, it's tempting to assume one fix unlocks one
module. The data this week does not back that up.

## Bug 1 — Backtick precedence

The first repro came from `Text.Pandoc.UUID`, a 60-line file with two
lowering errors: `unbound variable: g` and `unbound variable: getUUID`.
The function definition is straightforward:

```haskell
getUUID :: RandomGen g => g -> UUID
getUUID gen =
  case take 16 (randoms gen :: [Word8]) of
    [a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p] ->
      let i' = i `setBit` 7 `clearBit` 6
          g' = g `clearBit` 7 `setBit` 6 `clearBit` 5 `clearBit` 4
      in  UUID a b c d e f g' h i' j k l m n o p
    _ -> error "not enough random numbers"
```

Bisecting the repro down took maybe twenty minutes. The minimal case is
two `let` bindings inside a layout block, where the *first* binding's
right-hand side ends with two or more chained backtick operators:

```haskell
let i' = a `setBit` 7 `clearBit` 6     -- binding ends with `op` `op`
    g' = g `clearBit` 7                -- this binding gets lost
```

The lexer is fine; it emits the `VirtualSemi` cleanly between the two
bindings. The parser is where it goes wrong. The `Backtick` arm of the
precedence-climbing infix parser advances past the open backtick, the
function name, and the closing backtick *before* checking whether the
operator's precedence is high enough to use in the current context.

For a normal operator like `+` the arm reads the operator's precedence
from the precedence table and breaks early if it's too low — the operator
is left for an outer call to consume. For backticks the prec check
happened too late, after consumption. When the recursive call for a
backtick's right-hand side hit another backtick with the same precedence,
the inner operator was already consumed before the break fired, and the
parser exited the loop with the trailing operand stranded.

The fix is one line — an early prec-9-vs-min-prec check before any
`advance`:

```rust
TokenKind::Backtick => {
    if 9 < min_prec {
        break;
    }
    self.advance(); // `
    // ...
}
```

That's commit [`4f9a7d9`][bt-fix]. UUID went from failing to passing.
Pandoc moved 77 → 78.

[bt-fix]: https://github.com/arcanist-sh/bhc/commit/4f9a7d9

## Bug 2 — Qualified constructors at the start of pattern bindings

The next module on the short-error list was `Text.Pandoc.Writers.Djot`,
which had 103 lowering errors at the start of the week and a top-level
unbound-variable for `bodyToDjot` — a function that is defined in the
same module. Forward references like this should be handled by the
lowerer's pre-pass that walks the top-level declarations once before
descending into bodies. Standard machinery.

A small parser dump showed the actual problem. `bodyToDjot` was missing
from the AST entirely. Only its type signature had been parsed; the
function binding had been silently dropped by error recovery. In its
place, two top-level declarations had appeared: `autos` and `refs`. Both
of these are local `let` bindings inside `bodyToDjot`:

```haskell
bodyToDjot opts bls = do
  (bs, st) <- runStateT (blocksToDjot bls) (DjotState ...)
  let D.ReferenceMap autos = autoReferences st
  let D.ReferenceMap refs  = references st
  ...
```

The pattern bindings use a qualified constructor on the left-hand side:
`D.ReferenceMap autos` where `D` is the alias for `Djot.AST`. BHC's parser
handles qualified constructors in pattern positions perfectly well —
except for one missing case. The path that decides "is this a pattern
binding starting with a constructor?" was only matching unqualified
`ConId`, not `QualConId`. The fall-through tried to parse the qualified
name as a variable, errored, and recovery picked up the *inner* `let`-
bindings as if they were top-level.

The diff is two lines: add `QualConId(_, _)` to the match arm at
`decl.rs:807`. Pattern parsing one level deeper already knew how to
handle qualified constructors.

```rust
if matches!(
    self.current_kind(),
    Some(TokenKind::ConId(_) | TokenKind::QualConId(_, _))
) {
    // ...
}
```

The result on `Writers.Djot` was dramatic: 103 lowering errors down to
6, then to 1 after a missing `Djot.toIdentifier` stub was added. The
module still fails — but in type-checking, not lowering. The 91-error
column got one entry less in the lowering bucket and one more in the
type-check bucket. Module count: still 78. Commit: [`e3df9f6`][qc-fix].

[qc-fix]: https://github.com/arcanist-sh/bhc/commit/e3df9f6

## Bug 3 — Multi-line Haddock comments

`Text.Pandoc.Readers.DokuWiki` was failing because `splitInterwiki`, a
function defined at line 312, was reported as unbound. Same shape as the
Djot bug — the function had been dropped from the AST. The first
diagnostic in the parser dump pointed at a Haddock comment:

```haskell
parseLink f l r = f
  <$  textStr l
  -- ... (multi-line operator chain)
  <* textStr r

-- | Split Interwiki link into left and right part
-- | Return Nothing if it is not Interwiki link
splitInterwiki :: Text -> Maybe (Text, Text)
```

The lexer emits a `VirtualSemi` between two same-column items at the
module layout level. Two Haddock lines starting at column 1 produce a
`VirtualSemi` between them. The `collect_doc_comments` routine in the
parser collected the first line, hit the `VirtualSemi`, and broke. The
*second* line was then seen by `parse_top_decl` with "expected
declaration", recovery kicked in, and the next genuine declaration —
including its body — got dropped.

The fix: inside `collect_doc_comments`, peek across a `VirtualSemi` and
keep going if the next token is another doc comment. If it isn't, restore
position and break so trailing/leading-doc detection still works at the
end of an item.

```rust
TokenKind::VirtualSemi if !texts.is_empty() => {
    let save = self.pos;
    self.advance();
    if !matches!(
        self.current().map(|t| &t.node.kind),
        Some(TokenKind::DocCommentLine(_) | TokenKind::DocCommentBlock(_))
    ) {
        self.pos = save;
        break;
    }
}
```

Commit: [`aa2f901`][hd-fix]. No module-count movement. DokuWiki has an
unrelated ViewPattern issue at line 315 that still drops `splitInterwiki`
from the AST through a different code path. But the Haddock fix is real
silent-corruption work; it eliminated parse errors in many other files
that had been getting along on recovery, just below the diagnostic line.

[hd-fix]: https://github.com/arcanist-sh/bhc/commit/aa2f901

## Bug 4 — Sixty missing stubs

Not really a single bug, more an admission. BHC stubs external packages
the way a tracer stub does — register the names so imports resolve and
lowering can continue, then provide as much or as little type
information as the type-checker needs. The stubs for `Djot.AST`,
`Text.DocLayout`, and `Skylighting` were narrow enough that they
unblocked a few writers but left the bulk of the surface area
unresolved.

Expanding three stub tables was mechanical. Enumerate every `D.foo`
reference under `pandoc-3.6.4/src/`, look up which alias resolves to
which module, add the missing names. About sixty entries across the
three modules.

```rust
"Text.DocLayout" => &[
    // ...existing primitives
    "bold", "italic", "underlined", "strikeout",
    "fg", "bg", "link",
    "red", "green", "blue", "yellow", "cyan", "magenta", "white", "black",
    "renderANSI", "replicateChar",
],
```

The result on the lowering numbers was the largest of the week. Writers
that had been bleeding lowering errors closed up: `Writers.Djot` dropped
from 103 to 6, `Readers.Djot` from 70 to 31, `Writers.ANSI` from 25 to
14. The module count didn't move; everything that crossed lowering ran
straight into type-check failures, because the stubs are typed thinly
and don't satisfy class constraints like `PandocMonad m =>`.

Commit: [`f1ed1c7`][stub-fix].

[stub-fix]: https://github.com/arcanist-sh/bhc/commit/f1ed1c7

## What the data says

Four commits. Net Pandoc module gain: one. The honest reading of that is
not "parser correctness work is unproductive" — three of the four were
real silent-corruption bugs, the kind that quietly drop AST nodes and
break things downstream in ways that look like other bugs. The honest
reading is that *most modules fail for more than one reason at a time*.

A representative failing module has a parse-recovery loss, a missing
stub, and a type-check problem against an under-typed stub, all at once.
Fixing one of the three drops the error count and moves the failure
into a different bucket, but doesn't make the module pass.

That has implications for prioritisation. Lining up parser fixes by
"shortest error list" — the strategy we tried first — is the wrong
heuristic. A module with one parse error is not one fix away from
passing; it's one fix away from being one fix away from passing, with
the next fix being in a different layer.

The next push is going to look different. Either at type-class stubs
with enough type information that the lowering passes can be allowed to
propagate through type-check, or at the type-check failures themselves
on the modules that already cross lowering. The parser-correctness work
is necessary but it is not, on its own, sufficient.

## What this means in practice

If you have a Haskell module that doesn't compile under BHC, the most
useful thing you can do is:

- Minimise it to a small repro and [open an issue](https://github.com/arcanist-sh/bhc/issues).
- Tell us which stage fails (parse, lowering, type-check). The Pandoc
  runs show the stage in the summary; small repros usually fail at the
  same stage as the original.
- Send the diagnostic too. Cascading errors are common; the first one
  is usually the load-bearing one.

That's where the next batch of fixes will come from.
