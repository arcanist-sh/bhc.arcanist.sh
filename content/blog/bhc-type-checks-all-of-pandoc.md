+++
title = "BHC Type-Checks All of Pandoc"
description = "bhc check now passes every one of Pandoc's 221 library modules — zero failed, zero skipped, Template Haskell included. Two days ago the number was 153. This is the story of the endgame: the 84-error module that was one missing feature, the bugs that hide by deleting themselves, and what a permissive type checker is actually for."
date = 2026-08-04
template = "blog-post.html"

[extra]
tag = "milestone"
+++

In February, [BHC parsed all of Pandoc](@/blog/bhc-parses-all-of-pandoc.md) — 221 of 221 source files through the lexer and parser, with 10 modules making it all the way through `bhc check`. On Saturday, [the number was 153](@/blog/reading-pandoc-for-compiler-bugs.md). Today:

| Metric | Count |
|--------|-------|
| Total library modules | 221 |
| Pass `bhc check` | **221 (100%)** |
| Failed | 0 |
| Skipped | 0 |

Every module. The parser-combinator readers, the seventeen writer backends, the ODT reader with its custom arrow transformers, the Template Haskell data embedders, and `Text.Pandoc.App` — the module that ties the entire application together and therefore transitively depends on everything else.

`bhc check` is the front end without the back end: lex, layout, parse, name resolution, HIR lowering, type inference. It stops before code generation, and it models Pandoc's external dependencies as permissive stubs rather than real libraries. Both caveats matter, and we'll come back to what they mean. But first the good part, because the endgame taught us more per bug than any stretch of this project so far.

## The 84-error module that was one missing feature

`Text.Pandoc.Readers.Org.Meta` sat at the bottom of our list for weeks, labelled "the deep one." Eighty-four type errors, all variations on a theme:

```
type mismatch: expected ((ParsecT Sources) OrgParserState), found (,)
type mismatch: expected Future, found ((ParsecT Sources) OrgParserState)
```

A parser expected, a tuple found — over and over, across the whole module. Errors like that, at that volume, smell like broken inference. We had quietly budgeted days for it.

The actual cause is three lines of Haskell:

```haskell
infix 0 ~~>
(~~>) :: a -> b -> (a, b)
a ~~> b = (a, b)

keywordHandlers :: PandocMonad m => Map Text (OrgParser m ())
keywordHandlers = Map.fromList
  [ "author" ~~> lineOfInlines `parseThen` collectLines "author"
  , ...
  ]
```

BHC had no support for fixity declarations. None. The parser carried a hard-coded table of standard operator precedences, and everything else defaulted to `infixl 9`. For most real code that default is invisible. Here it was fatal: with `~~>` at precedence 9 instead of its declared 0, every entry in that table groups as

```haskell
("author" ~~> lineOfInlines) `parseThen` collectLines "author"
```

— feeding `parseThen` a *tuple* where it expects a parser. One entry per handler, several errors per entry: eighty-four errors, one cause.

The fix is a token-stream pre-scan. Before parsing begins, the parser walks the raw tokens looking for `infix`/`infixl`/`infixr` declarations and records them in a map that the precedence table consults first. A pre-scan rather than an in-order pass because Haskell lets you declare fixity after use; the declaration's position in the file must not matter.

The lesson is one we keep re-learning: **error count is not difficulty**. The scariest module on the board was a parser feature we simply hadn't built, and the one-error modules were routinely harder. When a type checker produces dozens of *structurally identical* errors, suspect a single upstream cause — usually in the parse.

## Bugs that hide by deleting themselves

The dominant failure class of the whole endgame was the silent drop: a binding whose parse fails is discarded by error recovery, and the only symptom is `unbound variable` at *someone else's* use site. We wrote about this class on Saturday. What the final stretch added is a catalogue of just how many ways real Haskell can trip a recovering parser:

- **View patterns containing type applications** — `entity (TR.decimal @Integer -> Right (x, "")) = ...` in the Pod reader. The pattern parser folded `@` into an as-pattern, or left it dangling after a qualified name, depending on the path.
- **View patterns containing compositions** — `findEntryByPathE (normalise . unEscapeString -> path)` in the EPUB reader. The pattern parser stops at an operator; it now hands the prefix to the expression parser's precedence climber and resumes at the `->`. A subtlety that cost an hour: `.` is not an operator token in our lexer, it's `Dot`.
- **Guard commas inside parenthesized case blocks** — `(\case t | not (p t), t /= "x" -> t; ...)`. Our layout heuristic that closes a `case`-in-tuple on a comma fired on the *guard* comma. The lexer now tracks whether a `|` is more recent than an `->` and holds fire mid-guard.
- **Bare `!` in every operator position** — sections `(!)` and `(! x)`, infix-in-parens `(H.ol ! A.start n $ ...)`, definitions `(!) a b = ...`, and fixity declarations `infixl 9 !`. The lexer emits `!` as a strictness marker; five separate parser sites needed to accept it as the operator.
- **GADT record constructors** — the ODT reader's `XMLConverterState :: NameSpaceID nsID => { parentElements :: ..., ... } -> XMLConverterState nsID extraState`. Now desugared to an ordinary record declaration so the field accessors register.
- **Guarded pattern bindings** — `(sampOrVar, cs') | "sample" `elem` cs = ... | otherwise = ...` in a `where` block. The pattern-binding parser demanded a bare `=`.

Each of these deleted at least one function from at least one module. And each produced the same testing trap: check a file containing *only* the offending construct and BHC reports `OK`, because the dropped binding is simply gone and single-file mode resolves the missing name as a stub. Twice we "verified" a repro that was silently reproducing the bug. Every drop-class repro now carries a second top-level binding that *references* the suspect — if the suspect vanishes, the reference fails loudly.

The diagnostic that finally made this class cheap to hunt: parse the module, collect every name with a type signature, collect every surviving function binding, and diff. `Writers.HTML` — the biggest module in the tail, 180 reported errors — showed exactly three dropped top-levels and 282 buried parse errors. The 180 was noise; the 3 were the work.

## Two namespaces, one name

The second big class was resolution, not parsing. BHC models external packages as *stub modules*: lists of names that resolve to permissively-typed placeholders. For most packages, a qualified use like `T.pack` routes through the unqualified name — one shared stub per name, cheap and effective.

The scheme has a failure mode: it assumes a name means the same thing everywhere. Real Haskell namespaces disagree, constantly:

- `H.div` — blaze-html's `div :: Html -> Html` — resolved to the *integer division builtin* `div :: a -> a -> a`. Every element combinator in `Writers.HTML` that shares a Prelude name (`head`, `span`, `id`, `min`, `max`) had the same problem.
- `Ipynb.Code` — a notebook cell-type record — resolved to pandoc-types' `Code :: Attr -> Text -> Inline`.
- `Citeproc.SuppressAuthor` — a citation-item type — resolved to pandoc-types' `SuppressAuthor :: CitationMode`. Same spelling, entirely different type, one import away.
- `H.Table{ tableHeaderRows = ... }` — haddock-library's table record — resolved to pandoc-types' seven-argument `Table` constructor.

The fix is a *dedicated-stub path*: modules (or specific exports) whose names are known to collide get their own definitions under their qualified names, never routing through the shared namespace. Blaze's element and attribute modules, jira-wiki-markup, `Data.Ipynb`, haddock-library's types, and citeproc's citation family all live there now. The rule of thumb we ended up with: a stub module earns the dedicated path the moment any of its exports is spelled like a Prelude builtin or a pandoc-types constructor — and it's cheaper to check that up front than to debug `secttag $ x` failing because `secttag` is secretly integer division.

## Template Haskell, the permissive way

Three modules embed data files at compile time:

```haskell
dataFiles' :: [(FilePath, B.ByteString)]
dataFiles' = ("MANUAL.txt", $(embedFile "MANUAL.txt")) : ...
```

BHC has no Template Haskell evaluator on this path, and for `bhc check` it doesn't need one. A `$` in prefix position followed by `(` can only be a splice — the infix `f $ (x)` case is consumed by the expression parser before prefix position is ever consulted — so the parser now accepts `$(expr)`, type-checks the inner expression, and gives the splice itself a fresh type per use. `embedFile`'s result unifies with `ByteString` because nothing constrains it otherwise.

That is not Template Haskell support. It is precisely enough to say: this module's *surrounding* code is coherent, and the splice's uses agree with its declared types. For a front-end integration target, that's the right trade. (Arrow `proc` notation got the same treatment: `proc pat -> cmd` desugars to `arr (\pat -> cmd)` with permissive feed operators, which is exactly enough to check the ODT reader's converter arrows without implementing the full arrow-command translation.)

## The failure you can't see

One find deserves its own heading because of how it *presented*. Mid-endgame, the scoreboard read: zero failed, eleven skipped — every skip saying `dependency failed`. No module in the report had failed. The skips chained back to `Text.Pandoc.Readers.Docx.Lists`, whose imports are all green modules… plus `Text.Pandoc.JSON`.

`Text.Pandoc.JSON` isn't one of the 221. It comes from pandoc-types, which `bhc check` loads as a supporting package — resolved and checked, but never scored or reported. It was failing on a missing `Data.ByteString.Lazy.getContents` stub, invisibly, and poisoning eleven scored modules downstream.

If your integration harness has unscored dependencies, their failures will surface as unexplained skips in the scored set. Check the support tier directly before assuming the skip logic is wrong.

## What this does and doesn't mean

Honesty section, as always.

**What it means.** BHC's front end — lexer, layout algorithm, parser, name resolution, lowering, and type inference — handles the full breadth of syntax and idiom in a 60,000-line production Haskell codebase: 30+ GHC extensions, custom operator vocabularies with declared fixities, view patterns, GADTs, arrow notation, associated type families, and the long tail of layout interactions that only real code exercises. Every fix along the way landed with a regression test, and the passing set never went backwards: we diff the full module list after every change, and a fix that flipped one module while breaking another was rejected, every time.

**What it doesn't mean.** Two things. First, `bhc check` stops before code generation — this is understanding Pandoc, not compiling it. Second, and more fundamentally: Pandoc's ~80 external dependencies are stubs, and stubs are *permissive*. When `parseCommonmarkWith` is a placeholder with a fresh type, the checker verifies that Pandoc uses it consistently with its own annotations, not that Pandoc uses it correctly. Some of the endgame explicitly traded precision for coverage — an unresolvable cross-module associated type family (`StyleName t` whose instances live in another module) now unifies permissively instead of erroring, because at stub granularity the mismatch is an artifact, not a finding. GHC this is not, and it isn't trying to be: `bhc check`'s job is to be a fast, honest oracle for *our* front end against real-world source, and to fail loudly where the front end is wrong.

By that standard, the oracle has run out of things to say about Pandoc. Two days from 153 to 221, roughly forty distinct root causes over the full campaign, and the last error in the last module — for the record — was `decodeUtf8With` curated with one argument when it takes two. The final boss was an arity typo.

## What's next

The check-level story is done; the numbers now have to survive contact with the back end.

- **Code generation.** Lowering Pandoc's modules through Core to native code is the next integration target, and it will be a longer road than the front end — the type checker's permissiveness has no equivalent in codegen, where every stub becomes a real obligation.
- **More oracles.** The [bug-mining loop](@/blog/reading-pandoc-for-compiler-bugs.md) — point `bhc check` at a real Hackage package, harvest the failures, land the general fixes — works on any package now. Each new codebase is a new distribution of idioms.
- **The gaps we know about.** Fixity declarations don't yet propagate across module boundaries (the ODT arrow operators are covered by table defaults for now), backtick operators don't consult declared fixities, and `Class(..)` export lists don't expand to methods. All three are marked with failing constructs in our test suite.

Ten modules in February. Two hundred twenty-one in August. The parser was never the hard part, and neither, it turns out, was the type checker. The hard part is the same as it ever was: the enormous, patient enumeration of everything real code actually does.
