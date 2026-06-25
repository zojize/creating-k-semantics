# Case study: klox — Lox in K (the simpler, non-stack example)

A second worked instance of the method, deliberately simpler than the Bitcoin one and **non-stack-based**: an executable K semantics of **Lox** (the language from *Crafting Interpreters*), built test-first against Lox's own reference suite. Repo: [`klox`](https://github.com/zojize/klox). Where the Bitcoin study shows the method at consensus scale, this one shows it on an ordinary imperative + OO language — variables, lexical scope, closures, control flow, functions, classes — so it exercises the [k-patterns](k-patterns.md) (env+store, closures, control stack, resolver-as-second-semantics) rather than a stack machine.

This study is an honest in-progress dogfood: **layer 1 (expressions + print) is green against the real suite**; later chapters are the remaining loop. The point is that every claim below was *run*, not asserted.

## The three oracles (Step 0) — Lox qualifies cleanly

| Oracle | Lox instance |
| --- | --- |
| Formal spec | the book (free online), with an explicit grammar; features land **chapter by chapter** |
| Conformance suite | `craftinginterpreters/test/` — **265** `.lox` programs with `// expect:` / `// expect runtime error:` annotations, grouped by feature |
| Reference implementation | `jlox` (Java) and `clox` (C) |

The gate passed on the first look; no bootstrapping needed (265 annotated tests is plenty for the core).

## Chapters are the version layers (Steps 2–3)

Lox's book chapters *are* the dependency-ordered layering the method prescribes, and the test groups map one-to-one:

```text
expressions/print (ch7-8) → variables/assignment/block (ch8) → if/while/for/logical (ch9)
  → function/call/return (ch10) → closure (ch11) → class/field/method/this/constructor (ch12)
  → inheritance/super (ch13)
```

"Run the suite as of chapter N" is the no-regression gate in miniature: the harness counts a `class`/`var`/`fun` appearing in an early test group as a parse failure until *its* layer lands — the ledger working, not silent skipping.

## The loop, as actually run (Step 4)

Layer 1 was built by the inner loop against real `kompile` feedback — a faithful demonstration of read → write → run → green:

1. **Over-wrapped rules.** First cut wrote every rule as `<k> … …</k>`; kompile rejected the trailing `...`. The known-good template (`expr.k`) writes pure-computation rules **bare** (K applies them at the top of `<k>`); only state-touching rules (print → `<output>`) need the cell. Fixed by following the template.
2. **`DOMAINS` ambiguity.** Importing all of `DOMAINS` pulled in `List` and made the rule parser report "Expected: List" on `Int2Float(I)`. Narrowing to `INT`/`FLOAT`/`BOOL`/`STRING` (import only what you use) cleared it.
3. **Arity from the source, not memory.** `Int2Float` failed to parse; reading the K `domains.md` showed the real signature `Int2Float(Int, precision, exponentBits)` — doubles are `Int2Float(I, 53, 11)`. (This is the method's "read the spec/source, don't guess" rule applied to K's own library.)
4. **Number stringify.** `1.5` printed as K's `"0.15000000000000000e1"`; jlox prints `"1.5"`. Wrote a K formatter that reformats K's scientific rendering to Java's decimal style (integer-valued doubles drop the decimal). The oracle caught this immediately — a `print 3/2;` vector flipped it red.
5. **The Unicode trap, on cue.** A non-ASCII string literal came back mojibake — exactly the "[K Strings are UTF-8 bytes](oracle-harness.md)" trap the method warns about. The host-side decode is in; the 3-byte input alignment is a documented remaining gap.

**Result: 13/13 pure expression+print tests green**, with four documented layer-1 xfails (negative zero, 3-byte UTF-8 round-trip, multiline strings, empty/comment-only programs) — each *explained*, none silently skipped.

## The harness is the pyk best-practice, dogfooded

`klox` is the [pyk-harness](pyk-harness.md) reference made concrete: a `[project.entry-points.kdist]` build target whose `Target.build` calls the in-process `kompile()` API; the definition resolved with `kdist.get('klox-semantics.llvm')`; programs parsed via the generated `parser_PGM` (the one legitimate non-K subprocess), run via `llvm_interpret` on a KORE `Pattern`, results read from the `<output>` cell by walking the AST — **no `krun`/`kompile`/`kprove` CLI anywhere**. Project hygiene: PEP 621 + hatchling + `uv` + pyright, `kframework` pinned.

## What this case study is for

It is the *small, legible* companion to the Bitcoin study: a reader who wants to see the method's loop end-to-end — gate, layer, write-a-rule, run-the-oracle, green, advance — can read `klox` in one sitting, with no crypto or consensus background. It also proves the method is not Bitcoin-shaped: the same skill produced a stack-machine semantics and a tree-walking imperative/OO semantics from the same steps.
