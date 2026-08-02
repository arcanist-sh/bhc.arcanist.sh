+++
title = "Reading Pandoc for Compiler Bugs"
description = "bhc check now clears 153 of Pandoc's 221 library modules, up from 112 in late July. A large real-world library is a bug oracle: point the type checker at it, and it finds the holes in your front end faster than any hand-written test suite. Nine of the recent fixes, and the two techniques that found them."
date = 2026-08-02
template = "blog-post.html"

[extra]
tag = "milestone"
+++

`bhc check` is BHC's front end without the back end. It runs the full pipeline up to and including type checking — lexing, layout, parsing, name resolution, HIR lowering, inference — and stops before code generation. It is the fastest way to ask a single question of a Haskell module: *does BHC understand this source the way GHC does?* When the answer is no, the module fails in one of two ways. Either lowering reports an unbound name, or the type checker reports a mismatch.

We run `bhc check` against [Pandoc](https://pandoc.org). Pandoc is a good subject for the same reason it is a hard one: it is a large, idiomatic, production Haskell library — 221 modules of readers, writers, and shared infrastructure — written by people who use the language fully and without apology. It has multi-line signatures with stacked class contexts, deeply nested `where` clauses, qualified imports from thirty packages, Template Haskell splices, and every corner of the pattern grammar. None of it is written to exercise a compiler. That is exactly why it exercises a compiler so well.

In late July `bhc check` cleared 112 of those 221 modules. As of today it clears **153**. This post is about the last stretch of that climb, and about a claim that took us a while to believe: a real library is a better bug oracle than a test suite. Every module that fails is a hole in the front end that no one wrote a test for, because no one thinks to test a compiler on `case n of -1 -> ...`. Pandoc thinks to. Pandoc does it in `Text.Pandoc.Writers.ConTeXt`, on line 781, without ceremony.

## A compiler that could not parse negative one

The clearest example first, because it is the one that best explains why this method works.

`Text.Pandoc.Writers.ConTeXt` failed `bhc check` with a strange error: `unbound variable: sectionLevelToText`. Strange because `sectionLevelToText` is defined right there in the module, and defined correctly. It is used twice, and both uses reported it as unbound, and its own definition reported nothing wrong at all.

A binding that is unbound at its use sites but clean on its own body is not a scoping bug. It means the binding was *dropped* — the parser failed somewhere inside it, recovered by discarding the whole declaration, and moved on. The name never made it into the module. The uses then resolve to nothing.

The next question is which construct in `sectionLevelToText` the parser choked on. We reduced the function — a faithful copy, then progressively less of it — and the drop tracked to a `case` expression with a negative-literal pattern:

```haskell
sectionLevelToText opts (_,classes,_) hdrLevel headingType = do
  ...
  return $ case hdrLevel + shift of
             -1         -> literal "part"
             0          -> chapter
             n | n >= 1 -> ...
```

The `-1 ->` alternative. That was the whole trigger. And here the reduction got misleading. Pulling the case into a standalone module — same alternative, same shape — and it passed. Adding the surrounding types, the monad, the tuple parameter: still passed. It looked, for an afternoon, like the drop depended on the *types* flowing through the function, which is nonsense for something that happens at parse time.

It was not the types. It was the signature. Every self-contained reproduction we wrote had a single-line signature. `sectionLevelToText` has a three-line one:

```haskell
sectionLevelToText :: PandocMonad m
                   => WriterOptions -> Attr -> Int -> HeadingType
                   -> WM m (Doc Text)
```

A single-line signature followed by `-1` in a case: fine. A multi-line signature followed by `-1` in a case: the function is dropped. To confirm it was a parse drop and not a lowering discard, we added a one-line probe to `collect_module_definitions` — the pass that registers every top-level name before any body is checked — printing each declaration it collected. `sectionLevelToText` never appeared. The parser had already thrown it away.

The root cause, once cornered, is embarrassing in the way good bugs usually are. BHC's lexer emits `-1` as a `Minus` token followed by an integer literal — it never forms a single negative literal. And the atom-pattern parser had no case for `Minus`. So a negative-literal pattern was, strictly, a parse error. Everywhere. It had been invisible only because parser error recovery usually contained the damage to the one alternative — except when a preceding multi-line signature left recovery pointing at a declaration boundary, at which point it discarded the whole function.

The fix (`25b7b21`) is a `Minus` arm in `parse_atom_pattern`: consume the `-`, consume the following integer or float literal, negate the value, produce the pattern. We were careful to leave the function-argument path untouched, so that `x - y = ...` still parses as a definition of the `-` operator rather than a call applied to `-y`. That distinction is the one genuinely subtle thing about negative literals in Haskell, and it lives in a different part of the grammar.

That one fix flipped ConTeXt. It is also, quietly, a correctness fix for every negative-literal pattern anyone had ever written and not noticed was mis-handled.

## The other kind of dropped binding

The negative-literal drop was one instance of a pattern we saw four or five times over these two weeks: a binding that exists in the source, is correct, and is nonetheless unbound at every use site. The cause is always a parser or lowering hole, never a resolver bug, and the tell is always the same — no error on the binding's own body.

- **As-pattern bindings.** `let key@(Key k) = ...` — a `let` or `where` binding whose left-hand side is an as-pattern. The declaration parser knew how to start a binding with a constructor, an operator, or a plain name, but not with `name@`. It treated the `@` as a stray argument, failed, and dropped the binding. `Text.Pandoc.Readers.RST` does this inside a guard. The fix (`148ce85`) recognizes `name@` as the start of a pattern binding.

- **Nested `where`, past depth two.** The lowering that turns `where` clauses into HIR had been written to handle a `where` inside a `where`, but it lowered the inner bindings' bodies with a helper that did not, in turn, thread *their* `where` clauses. So `a = b where b = c where c = ...` dropped `c`. The lowering is now properly recursive (`b6c56b2`), which is to say it now does what its own recursion always claimed to.

- **`where` on a pattern binding.** `let (a, b) = f x where f 0 = ...` — the named-binding path threaded the attached `where`; the pattern-binding path did not, and dropped `f`. Fixed at every site that lowers a pattern binding (`bd6d7e3`). This one flipped `Text.Pandoc.Writers.ICML`.

The technique that found all of these is worth naming, because it is the load-bearing one. We call it *faithful-copy bisection*. Take the failing module, rename it, and confirm the copy still fails against the real package sources — no simplification yet, just a rename. Then delete, do not rewrite. Cut the module in half; keep the half that still fails; cut again. The rule is that you never paraphrase the code, because paraphrasing is how you accidentally fix the bug and then cannot find it. Our first attempts at every one of these bugs started with a hand-written reproduction that passed, precisely because the hand-written version dropped the one detail that mattered — the `@`, the third level of `where`, the multi-line signature. The library keeps the details. You just have to be disciplined about removing them one at a time.

There is a Haskell-2010 detail in this bucket too, small and clean: `if e [;] then e [;] else e`. When a multi-line `if` places `then` and `else` at the enclosing layout column, the layout algorithm inserts a semicolon before them, and the grammar has permitted that optional semicolon since the DoAndIfThenElse rule landed in the standard. BHC's `if` parser did not permit it, so the layout-inserted semicolon was a parse error that — again — dropped the enclosing binding. `Text.Pandoc.Writers.FB2` writes exactly this shape. The fix (`847e4ec`) is two tolerated semicolons.

## The other lever: reading the error histogram

Not every remaining failure is a dropped binding. Many are honest "this name is not implemented yet" gaps — imports from packages whose signatures BHC stubs but has not filled in completely. Individually these are boring. In aggregate they are the highest-leverage work in the whole exercise, and the way to see that is to stop reading modules one at a time and read the *distribution* of errors instead.

```
$ grep 'unbound variable:' check.txt | sed 's/.*: //' | sort | uniq -c | sort -rn
  19 toValue
  18 !
  16 addAttrs
   ...
   4 tokenItalic
   4 tokenColor
   4 tokenBold
```

Every name in that histogram is a module's worth of failure, sorted by how many modules share it. The `token*` cluster near the bottom turned out to be five `Skylighting` names — `defStyle` and the `TokenStyle` record accessors — present in the type but missing from the stub export list. Both `Text.Pandoc.Writers.Man` and `Text.Pandoc.Writers.Powerpoint.Presentation` build terminal and slide styling out of a highlighting theme, and both used exactly those accessors. Five names added to one list (`7d399c8`) flipped both modules at once.

The next cluster was `Text.XML.Light`'s config pretty-printer — `ppcElement`, `defaultConfigPP`, `useShortEmptyTags`, and two neighbors. They were stubbed under `Text.XML.Light.Output`, but the modules that use them import the parent module `Text.XML.Light`, which re-exports Output in real Haskell and did not in our stub. Adding the re-export (`e84b8fb`) flipped `Text.Pandoc.Writers.DocBook` and cut `Text.Pandoc.Writers.ODT` from twelve errors to four. And `Data.List.deleteFirstsBy`, one line of `Text.Pandoc.Readers.RST`, was simply a `Data.List` function we had never registered (`65a4519`).

None of these are clever. The cleverness, such as it is, is in the histogram: it tells you which boring fix pays for two modules instead of one, and it tells you when a cluster is not worth chasing — the top three names above are all `Text.Blaze` combinators, and they all live in a single 180-error module, so filling them in reduces that module without coming close to flipping it. The histogram is as good at saying *not yet* as it is at saying *here*.

## By the numbers

| `bhc check` on Pandoc | modules passing (of 221) |
|---|---|
| The [parses-all-of-Pandoc](/bhc/blog/bhc-parses-all-of-pandoc/) post | 53 |
| The [ten-zentinel-modules](/bhc/blog/zentinel-ten-modules-compile/) post | 76 |
| July 23 | 112 |
| July 31 | 147 |
| August 2 | **153** |

The nine fixes behind the most recent stretch, in one place: optional semicolons before `then`/`else`; list-comprehension `let`-qualifier scoping; as-pattern bindings; `Data.List.deleteFirstsBy`; recursive nested-`where` lowering; `where` clauses on pattern bindings; negative-literal patterns; the Skylighting `TokenStyle` accessors; and the `Text.XML.Light` config pretty-printer. Six are general front-end bugs. Three are library coverage. Every one landed with a regression test and a full re-run of `bhc check` across all 221 modules to confirm nothing that passed before now fails.

## Honest about the boundary

`bhc check` is type checking, not compilation. A module that clears it has been parsed, resolved, lowered, and inferred without contradiction. It has not been code-generated, linked, or run. The claim is scoped to the front end, and it is the right scope for this work: the front end is where a compiler either agrees with the language or does not, and Pandoc is a very precise instrument for measuring that agreement.

The 68 modules that still fail sort into three groups, and it is worth being clear about which is which. Some depend on Template Haskell splices — `$(embedFile "...")` and friends — which BHC does not yet evaluate; those are a separate, larger piece of work, not a front-end gap. Some are the genuinely large writers, `Text.Pandoc.Writers.HTML` chief among them, where the error count is in the hundreds and no single fix moves the needle far. And a thinning few are ordinary front-end holes of the kind this post is about, waiting for the next histogram.

BHC's contract is compatibility with Haskell as the baseline. Every fix here is a fix toward that baseline: a place where a valid Haskell program meant one thing to GHC and, until we corrected it, meant nothing or the wrong thing to BHC. None of it is a new dialect, and none of it asks Pandoc to change a line. The measure of the front end is how much unmodified, idiomatic Haskell it reads correctly, and the honest number for that today is 153 of 221, climbing.

We will keep going.

---

*BHC is open source at [github.com/arcanist-sh/bhc](https://github.com/arcanist-sh/bhc). `bhc check <files>` runs the front end and stops before code generation; point it at your own library and tell us what breaks.*
