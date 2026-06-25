---
name: creating-k-semantics
description: Build or extend an executable K-Framework semantics for a real language (JavaScript, Python, a bytecode, a DSL), oracle-driven and test-first. Triggers — starting a K semantics for a language; adding the next language version or feature layer to an existing K definition; an autonomous loop or harness driving oracle-checked semantics development. Also use when another skill needs the oracle-driven, version-layered method for formalizing a language.
---

# Creating K Semantics for a Programming Language

You are writing an **executable** formal semantics: a K definition that runs real programs, not prose. The method that makes this tractable for a large language — and safe to run agentically or in a loop — has one root idea:

> **The oracle is the spec.** Every change is a *verifiable repair* against ground truth the system can check itself: a conformance vector flips green, the reference implementation agrees on a concrete input, the rule matches the cited spec clause. Where there is no oracle, there is nothing to verify, and a generated semantics is unfalsifiable — do not build there.

Two nested loops run this:

- **Inner loop (per feature):** read the spec clause → write the minimal K rules → run the oracle → iterate red→green.
- **Outer loop (per version):** walk language versions oldest-first; each is **done** only when its whole conformance subset is green *and a full re-run shows zero regression in every earlier layer*.

The failure mode to fear is **overfitting**: rules contorted to pass a test instead of modeling the spec. The spec document is the guard against it; the no-regression gate and the reference implementation are the net that catches it.

This skill is K-Framework-specific but language-agnostic. Bitcoin Script (a consensus stack machine) and Lox (a tree-walking imperative/OO language) are the worked examples, kept entirely in their case-study references so nothing here couples to either.

## The loop at a glance

```text
Step 0  Gate: does the language qualify? (3 oracles)        ── once
Step 1  Build the oracle harness                            ── once
Step 2  Pick the oldest unfinished version layer            ──┐
Step 3  Order its features by dependency → backlog          ──│ outer
Step 4  Per feature: read spec → write rules → run → green  ──│ loop
Step 5  Advance gate: full re-run, zero regression          ──┘
```

Stop-for-review at three design boundaries (below). End when every targeted version layer is green.

## Step 0 — Gate: qualify the language

A language is buildable this way only if you can name and **run** all three oracles. Find them before writing a single rule.

1. **Formal spec** — the algorithmic ground truth you mirror in rules (ECMA-262 for JS; the Python Language Reference for Python).
2. **Conformance suite** — machine-checkable, tagged by version/feature; this is what flips green (test262 for JS; CPython's `Lib/test/` for Python).
3. **Reference implementation** — a differential oracle for behavior the suite misses (V8/Node for JS; the CPython binary for Python).

[references/oracle-harness.md](references/oracle-harness.md) has the concrete wiring for each.

**Hard gate:** no conformance suite *and* no runnable reference implementation → STOP. You would be generating unverifiable semantics — the one situation this method cannot protect you from. (A formal spec alone is not enough; you need something executable to check against.)

**Completion criterion:** you have all three in hand, can run the suite and the reference impl locally, and have located the spec section for your first feature. If the suite is thin for some features, that is fine — Step 1 bootstraps coverage.

## Step 1 — Build the oracle harness (once)

Wire the three oracles into a single command that, given your current (possibly near-empty) K definition, reports structured pass/fail per vector and flags any disagreement with the reference implementation. See [references/oracle-harness.md](references/oracle-harness.md) for concrete JS and Python wiring. Build it as a real Python project that drives K through the **pyk library** — kdist build targets, in-process `llvm_interpret`, KORE-AST results — and **never the `kompile`/`krun`/`kprove` CLI via subprocess**; [references/pyk-harness.md](references/pyk-harness.md) is the build/run/prove API surface and project setup.

The harness must:

- **Select** a vector subset by version/feature tag (so the loop can target one feature).
- **Run** the K definition on each program and compare result/error to expected.
- **Differential-check** inputs the static suite does not cover by running the reference implementation and diffing — any disagreement is a bug in your semantics (or your harness), never a "result."
- **Bootstrap** coverage where the static suite is thin: harvest real programs (open-source corpora, the reference impl's own test files, fuzzed inputs) and *label them with the reference implementation*. This is how a few thousand hand-written vectors become hundreds of thousands of checked inputs (see the case study).

**Completion criterion:** one command runs a tagged subset end-to-end and prints structured pass/fail plus any differential mismatch; a trivial known-good program (e.g. `1 + 1`) is green through the full pipeline.

## Step 2 — Pick the oldest unfinished version layer

The outer loop is **chronological**: formalize the oldest version first and get it fully green before the next exists. For JS that is ES5 (or ES3) before ES2015 before later editions; for Python, the oldest dialect you target before newer ones. Earlier features are the substrate later ones execute on, so this order means you never build on unmodeled ground.

**Completion criterion:** the current layer is named, and its conformance subset is selectable in the harness by tag.

## Step 3 — Order the layer's features by dependency

Within a version, order sub-goals by **execution dependency**, not spec page order and not failing-test count:

```text
lexing / values / types  →  expressions  →  statements & control flow
  →  functions & scope  →  objects / prototypes / classes  →  builtins & stdlib
```

Each sub-goal binds one spec section to one conformance subset.

**At the builtins/stdlib stage, choose an architecture deliberately** (it is load-bearing enough to be a stop-for-review call). Two approaches work: write each builtin as native K rules, or **self-host the library in the source language itself** — a prelude written in the language under study, layered over a small core of K-level abstract operations. Real references lean on the latter: much of the JavaScript standard library is specified that way and KJS implements it in JavaScript; much of CPython's stdlib is Python. Self-hosting trades K-rule volume for a source prelude and keeps the K core small and spec-shaped; native rules keep everything in one formalism.

**Completion criterion:** an ordered, written backlog of sub-goals for this layer; each item names its spec section and its test-subset tag.

## When K's grammar can't parse the language

The first sub-goal, lexing, can hit a wall the tutorial languages never expose: **K's GLR parser is error-driven and discards whitespace and newlines as layout, so a context-sensitive lexing rule the spec mandates cannot be written as a grammar production.** JavaScript Automatic Semicolon Insertion, Python and Haskell significant indentation, and the JavaScript `/`-division-versus-regex split are all in this class — clean-grammar toy languages have none of them, a real language hits one on day one.

The fix is architectural, so treat it as a **stop-for-review** decision: model the irregular part as a *separate K definition that rewrites source text into normalised source text* (an in-K lexer plus the transformation), then let the evaluator's grammar parse the normalised output, where the irregularity is gone — explicit statement terminators, explicit block delimiters. Two kompiled definitions, glued by a thin harness that moves one string between them. Decide this before writing the evaluator grammar; it shapes the whole front end. The scanner-level mechanics (custom tokens, non-ASCII, projection-reserved braces) are in [references/k-patterns.md](references/k-patterns.md).

## Step 4 — The inner loop, per feature: read → write → verify → green

Take the next sub-goal and run the inner loop:

1. **Read the spec clause first** — and mine any **existing formalization or the reference implementation's own source** alongside it. A prior K semantics (KJS, KEVM), a mechanized spec, or the engine's own C/Python source often pins down a step the prose leaves implicit. Mirror the spec's structure (for ECMA-262, mirror the abstract operations); adapt a borrowed algorithm to *your* configuration rather than copying its structure across, since architectures differ. See [references/k-patterns.md](references/k-patterns.md) for the K idioms: the `<k>` computation cell, `strict`/`seqstrict` for evaluation order, the env+store split, control operators, and using a type system as a *separate* semantics over the same grammar.
2. **Write the minimal rules** for this feature only.
3. **Run the harness** on this sub-goal's subset. Iterate red→green.
4. **No overfitting.** Every rule cites a spec clause. A vector that goes green only via a special-case with no spec basis means the real bug is elsewhere — usually a missing foundational rule. Fix the cause; the no-regression gate will later expose the debt.

**Completion criterion (exhaustive):** 100% of this sub-goal's vectors are green, AND an anti-overfitting judge pass confirms every new or changed rule cites a spec clause with no orphan special-cases.

**Anti-overfitting judge pass:** an independent agent diffs every new or changed rule against its cited spec section and flags any rule with no spec basis. "The Coming Loop" warns unattended loops breed defensive, opaque code; for a semantics that means overfit rules, and this pass plus the spec-citation rule is the defense.

## Step 5 — Advance gate: zero regression

Before starting the next sub-goal or layer, re-run the **full** suite across every prior layer plus this one. This is the operational meaning of "passes ALL tests."

**Completion criterion:** the full re-run shows zero regressions and the current layer's entire conformance subset is green. Record the green snapshot (passing count + commit) before advancing. A drop in any earlier layer's count blocks advancement — fix it first.

## Stop-for-review boundaries

The inner loop runs autonomously. **Stop** and get a human (or judge agent) decision at the three points tests cannot settle:

- **Configuration / architecture design** — before adding or restructuring cells, sorts, or phases. Config shape is load-bearing and not test-verifiable; a wrong shape is expensive to undo later.
- **Spec ambiguity** — when the spec is unclear, or the suite and reference implementation disagree. Record the chosen interpretation and its justification.
- **Layer advancement** — before opening a new version layer, review the green snapshot and the layer's design.

These three are *design* calls; the per-sub-goal **anti-overfitting judge pass** (Step 4) is the routine check that runs without stopping the loop.

## Running it as a loop

The inner loop (Step 4) is fully agentic. A harness can drive Steps 3–5 within one version layer unattended — pick next sub-goal, write rules, run oracle, advance on green — pausing only at the stop-for-review boundaries. Keep the loop's stopping condition explicit: *advance on green, halt the layer when the subset is exhausted and the full re-run is clean.*

## Performance is part of "done"

An executable semantics that cannot run real-world-sized programs is not finished — KJS and KEVM both treated speed as a first-class result. Benchmark against the reference implementation and watch for K-specific cost cliffs (quadratic term concatenation, oversized result patterns).

**Completion criterion:** the semantics runs the reference implementation's own real-world corpus to completion within a stated factor of it (record the ratio), with no input timing out — or each remaining cliff is documented with its cause named. [references/performance.md](references/performance.md) is the K optimization toolkit (the O(n²)-concat trap → streamed accumulator, native bulk hooks with a pure-K twin, `[concrete]` result cleanup, binary-KORE construction, long-run memory); [references/case-study-bitcoin-script.md](references/case-study-bitcoin-script.md) is the worked 149×→2.1× path.

## References

- [references/k-patterns.md](references/k-patterns.md) — transferable K idioms for PL semantics, with verbatim examples from the K PL tutorial (LAMBDA, IMP, IMP++, SIMPLE) mapped to JS/Python; plus the front-end lessons real languages need beyond the tutorial's clean grammars (source→source transforms, scanner/token gotchas, unordered-`Map` enumeration).
- [references/oracle-harness.md](references/oracle-harness.md) — concrete oracle wiring for JS (test262 + V8/Node + ECMA-262) and Python (Lib/test + CPython), the bootstrap-by-differential-labeling technique, the pyk substrate for invoking K, and keeping the reference oracle hermetic.
- [references/pyk-harness.md](references/pyk-harness.md) — building the harness as a proper Python project: drive K through the pyk library (kdist `Target` builds, in-process `kompile`/`llvm_interpret`, KORE-AST results, Kore-RPC `APRProof`) instead of subprocessing the K CLI, plus pyproject/uv/typing hygiene. Verified against kimp, KEVM, and bitcoin-script.
- [references/performance.md](references/performance.md) — the K optimization toolkit when real inputs are too slow: the O(n²) `+Bytes`/`+List` trap and streamed accumulators, native bulk hooks with a pure-K twin and `[simplification]` equivalence, `[concrete]` result-cell cleanup, binary-KORE construction, and bounding native-backend memory over long suite runs.
- [references/case-study-bitcoin-script.md](references/case-study-bitcoin-script.md) — the proven success story: how this exact method built a Taproot-complete Bitcoin Script semantics, era by era, with zero mismatches over 288K real inputs.
- [references/case-study-lox.md](references/case-study-lox.md) — the simpler, non-stack companion: building a Lox (Crafting Interpreters) semantics chapter by chapter against its reference test suite; a small, legible end-to-end run of the loop with no crypto background needed.
