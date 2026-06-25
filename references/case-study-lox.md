# Case study: klox — Lox in K (the simpler, non-stack example)

A second worked instance of the method, deliberately simpler than the Bitcoin one and
**non-stack-based**: an executable K semantics of **Lox** (the language from *Crafting
Interpreters*), built test-first against Lox's own reference suite. Repo:
[`klox`](https://github.com/zojize/klox). Where the Bitcoin study shows the method at
consensus scale, this one shows it on an ordinary imperative + OO language — variables,
lexical scope, closures, control flow, functions, classes, inheritance — so it exercises
the [k-patterns](k-patterns.md) (env+store, closures, control stack, methods/`this`)
rather than a stack machine. It was driven **autonomously** through the book chapter by
chapter, and every number below was *run*, not asserted.

## The three oracles (Step 0) — Lox qualifies cleanly

| Oracle | Lox instance |
| --- | --- |
| Formal spec | *Crafting Interpreters* (free online), with an explicit grammar; features land **chapter by chapter** |
| Conformance suite | `craftinginterpreters/test/` — **265** `.lox` programs with `// expect:` / `// expect runtime error:` / `// Error` annotations, grouped by feature |
| Reference implementation | `jlox` (Java) — and crucially, **its source is the spec you mirror rule-for-rule** |

The single highest-leverage move in this whole study: the jlox source tree
(`Interpreter.java`, `Environment.java`, `LoxClass.java`, `LoxInstance.java`,
`LoxFunction.java`, `Parser.java`) sits *inside* the test repo. Every K rule is a
transcription of a specific jlox method, and the `//> chapter` markers in that source
*are* the layer boundaries. "Read the spec/source first" stops being advice and becomes
copy-this-method.

## Chapters are the version layers (Steps 2–3)

The book's chapters are the dependency-ordered layering the method prescribes, and the
test groups map one-to-one. All of these are **green** against their slice of the suite:

```text
expressions/print (7) → variables/assignment/block (8) → if/while/for/logical (9)
  → function/call/return (10) → closures (11 runtime) → class/field/method/this/init (12)
  → inheritance/super (13)
```

"Run the suite as of chapter N" is the no-regression gate in miniature: until a layer
lands, an earlier test group that uses its feature (a `class` inside `operator/`, a
`fun` inside `while/`) shows up as a *parse failure*, not a silent skip — the ledger
working. Final tally: **215 passed, 3 failed, 47 skipped** of 265, where the entire
remaining gap is one unbuilt layer (the resolver, below).

## The loop, as actually run (Step 4) — where the oracle earned its keep

Every one of these was a red test (or a `kompile` error) that pinned down a real defect.
This is the method's whole value: each fix is a *verifiable repair*, not a guess.

1. **The number-token sign trap.** `1 + 1` passed; `a+1` (no spaces) was a *parse error*.
   K's built-in `Float`/`Int` token allows a leading sign, so the lexer ate `+1` as a
   signed number and `a+1` became `a (+1)` — a syntax error. The fix is a recurring K
   lesson: when the host language's number/string literals differ from K's built-in
   tokens, **define your own `[token]` and convert** (`Int`/`LoxDec` → `Float` via
   `String2Float`). A spaced test would never have caught it; a no-space arithmetic
   vector did.

2. **Mutual recursion forced the configuration.** The first env model was a flat
   `Map` snapshot per closure. It got lexical scope right but **broke mutual recursion**:
   `fun isEven` defined before `fun isOdd` couldn't see `isOdd`. jlox captures the
   *mutable* `Environment` by reference, so a global defined later is visible. The repair
   was an architecture change (a judge-logged design decision): the store maps an integer
   **env id** to `envFrame(bindings, parentId)`, and a closure captures the *id* (a
   reference), not a snapshot. `function/mutual_recursion.lox` flipped green — and so did
   the closure-counter tests, because frames persist in the store and are shared by id.

3. **Argument-list strictness has no natural KResult.** The textbook `Exps ::= Vals`
   subsort made `f()` and `f(true)` parse *ambiguously*, and even with `[avoid]` the
   `seqstrict` couldn't recognize the empty `.Exps` tail as a result, so every call hung.
   The robust fix was to drop the subsort and **evaluate arguments explicitly** with an
   accumulator (`evalArgs → invoke`). Less clever, completely reliable.

4. **Declarations are not statements.** `while (true) var foo;` *timed out* — my grammar
   let a bare `var` be a loop body, so the loop never terminated. jlox's grammar forbids
   it (`statement`, not `declaration`, in control-flow position). Splitting `Stmt`
   (declarations) from `BodyStmt` (statements) turned six infinite loops into the parse
   errors jlox produces. The oracle caught a grammar bug as a *hang*.

5. **Smaller spec-faithful repairs, each from one red vector:** dangling-else → `[avoid]`
   so `else` binds the nearest `if`; `init()` returns `this` even on explicit `return`
   (a return-mode on the call frame); functions/classes use *identity* equality
   (`foo.method == foo.method` is false because each binding is a fresh env); `-0` needs
   IEEE negation (`-1.0 *Float F`, not `0.0 -Float F`); Java/jlox number stringify
   reformatted out of K's scientific rendering; multiline strings + the non-ASCII
   round-trip fixed together by a custom `LoxStr` token (newlines in, no escapes).

Earlier `kompile`-level lessons from layer 1 still hold and recur: pure-computation rules
are **bare** (not `<k>…</k>` wrapped); import only the `domains` you use; read arity from
K's own `domains.md` (`Int2Float(I, 53, 11)`).

## The architecture, in one breath

`<k>` computation; `<store>` mapping env-id → `envFrame(Map, parentId)` and instance-id →
fields `Map`; `<env>` current env-id; `<fstack>` of `callFrame(continuation, callerEnv,
retMode)`; `<output>`; `<rerror>` carrying jlox's exact runtime-error string. Closures are
`closure(params, body, envId, name, isInit)`; instances are `instanceV(fieldsLoc, class)`
— *references*, so field mutation aliases (jlox `LoxInstance`). Methods bind `this` by
extending the captured env; `super` is the enclosing-scope binding jlox installs. It is a
near-mechanical transcription of the jlox classes onto K cells.

## The honest gap: the chapter-11 resolver

jlox adds a *resolver* pass that (a) binds each variable to a static lexical depth and
(b) reports semantic compile errors. klox models scoping with the **dynamic frame chain**
above — chosen precisely because it makes mutual recursion free. That choice is exact
except for **forward references** (a closure naming a variable declared *later* in an
enclosing scope), which is the exact case the resolver exists to fix. So three behavioral
tests (`variable/early_bound`, `closure/assign_to_shadowed_later`,
`function/local_mutual_recursion`) and 22 static-error tests are attributable to this one
unbuilt layer — recorded in `STATUS.md`, not half-implemented. The other 25 skips are
groups outside the language-conformance set (`scanning`, `expressions`, `benchmark`,
`limit` — VM implementation limits a formal semantics has none of), exactly as the book's
own runner excludes them. This is the method's discipline: **document the layer you
didn't build, with the tests it owns, instead of faking coverage.**

## The harness is the pyk best-practice, dogfooded

`klox` is the [pyk-harness](pyk-harness.md) reference made concrete: a
`[project.entry-points.kdist]` build target whose `Target.build` calls the in-process
`kompile()` API; the definition resolved with `kdist.get('klox-semantics.llvm')`; programs
parsed via the generated `parser_PGM` (the one legitimate non-K subprocess); the kompiled
LLVM interpreter run on a KORE `Pattern`; results read from the `<output>`/`<rerror>`
cells by walking the AST — **no `krun`/`kompile`/`kprove` CLI anywhere**. The oracle
runner adds a thread pool (the subprocesses release the GIL), a per-program wall-clock
timeout (so a runaway can't wedge the suite), and structured pass/fail/skip with honest
reasons. Project hygiene: PEP 621 + hatchling + `uv` + ruff + pyright, `kframework` pinned.

## Performance is part of "done"

`fib(25)` (242,785 calls) runs in ~5.8s on the LLVM backend — ~42,000 Lox-calls/s, with
no cost cliff (deep recursion completes, no quadratic blowup). The ~1.4s *per-program*
floor in the harness is Python/uv startup plus KORE text (de)serialization — the **same**
finding as the Bitcoin study, and a native caller would erase it.

## What this case study is for

It is the *small, legible* companion to the Bitcoin study: a reader who wants to see the
method's loop end-to-end — gate, layer, write-a-rule-from-the-spec, run-the-oracle, green,
advance — can read `klox` in one sitting, no crypto or consensus background needed. It
also proves the method is not Bitcoin-shaped: the same skill produced a Taproot-complete
stack-machine semantics and a closures-and-classes tree-walking semantics from the same
steps, and in both the oracle — not the author's confidence — was what said "done."
