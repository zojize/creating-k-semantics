# Oracle harness wiring

Goal: one command that runs a **tagged subset** of your conformance suite against the current K definition, reports structured pass/fail, and **differential-checks** anything the static suite misses against the reference implementation. Build this before writing rules — it is the loop's heartbeat.

## What the harness must produce

For each vector, a structured record — never a bare boolean. Distinguish the failure kinds, the way the Bitcoin semantics distinguishes `#fail(ERR)` from a stuck term:

```text
{ id, tag, expected, got, status }
status ∈ { pass, wrong-value, wrong-error-class, stuck/crash, harness-error, differential-mismatch }
```

`wrong-value` vs `wrong-error-class` vs `stuck` tells you *where* the bug is before you open a trace. `differential-mismatch` (your semantics disagrees with the reference impl on an input the suite never covered) is always a real bug — in your rules or your harness — never a result.

The harness must let the loop **select by layer/feature tag** so Step 4 can target one sub-goal, and **re-run everything** so Step 5 can prove no regression.

---

## Driving the K definition from the harness

How you *invoke* K matters as much as what you check — the harness runs on every loop iteration, so build it on solid ground. Two substrates:

- **Hand-rolled `subprocess` over `kompile`/`krun`.** Quickest to stand up, but brittle: you scrape K's textual output, juggle temp files, and re-implement what the toolchain already does.
- **The pyk toolchain (recommended beyond a toy).** Declare each definition as a `kdist` build target and resolve it with `kdist.get(...)`; run programs in-process via `llvm_interpret`; read results by matching the **KORE `Pattern` AST** — find the cell you want and project out its `DV` — instead of parsing text. This is the idiomatic K-in-Python substrate (kimp, kevm, KJS use it). It is orthogonal to a vector-assembly runner like `test262-harness`: that builds the program text, this runs the K definition on it.

Three traps surface once you run a *suite of many small programs*, independent of language:

- **K `String`s are byte sequences, not Unicode.** Moving non-ASCII across the host↔KORE boundary needs a byte-exact encoding (treat the K String as a Latin-1 view of its UTF-8 bytes), or identifiers and string literals mojibake. Same byte-truth as why a Unicode token can't exist (k-patterns.md, front-end section).
- **Deep terms overflow.** A large program's right-recursive cons-list (and any deep stuck residual) nests deeper than the host language's default recursion limit *and* the parser stack — parse and serialise KORE on a worker thread with a raised limit and a roomy stack, or big inputs error instead of being measured.
- **The result configuration is huge.** It carries the whole store/heap, and (de)serialising it per test dominates cost — clear result-only cells at the terminal step. For thousands of tiny programs the per-invocation overhead (process spawn + KORE (de)serialisation) routinely dwarfs the rewrite engine itself, so profile end-to-end, not just the rewrite. This is the "oversized result patterns" cliff the method warns about.

---

## JavaScript

**Spec — ECMA-262** (tc39.es/ecma262). Its *abstract operations* are written as numbered pseudocode; mirror their structure directly in K rules (one rule per step, or per algorithm). Track the edition: ES5 is the 5th edition (test262 references the 5.1 revision), ES2015 the 6th, then yearly (ES2016…).

**Conformance suite — test262** (github.com/tc39/test262), the official TC39 suite. Each test file carries YAML frontmatter in a `/*--- … ---*/` block:

- `esid` (or legacy `es5id`/`es6id`) — the spec clause the test pins. **`es5id` is your ES5-layer selector** — a legacy field, but exactly the ES5-era marker you want.
- `flags` — `onlyStrict`, `noStrict`, `raw`, `module`, `async`, `generated`, `CanBlockIsFalse`…
- `includes` — harness helpers to prepend (from `harness/`: `assert.js`, `sta.js`, `propertyHelper.js`, `compareArray.js`, `doneprintHandle.js`).
- `features` — feature tags (`let`, `class`, `async-functions`, `BigInt`, `optional-chaining`, `generators`…). **Use these to gate layers:** the ES5 subset is `es5id`-tagged tests with no post-ES5 `features` and not `module`.
- `negative` — `{ phase: parse|resolution|runtime, type: <ErrorName> }` for tests that must fail.

Runner contract (write your own thin runner, or lean on `test262-harness`/`eshost`): unless `raw`, prepend `assert.js` + `sta.js`, then each `includes`, then the test source; for `onlyStrict` prepend `"use strict";`; for `negative` assert the named error in the named phase; otherwise success = no thrown error. Select by tag for the current sub-goal.

**Reference implementation — V8/Node** (`node`), SpiderMonkey (`js`), or JavaScriptCore; `eshost` runs one program across several engines at once. Differential check: run the program in your K semantics and in `node`, compare the completion value / thrown error name / printed output.

```text
# sketch
for vec in test262 where es5id and not features(ES6+) and tag == current-subgoal:
    src = assemble(vec)                         # includes + strict wrap per flags
    got = run_k_semantics(src)                  # value | error-name | stuck
    record(vec.id, expected_from(vec.negative), got)
for src in harvested_corpus(feature):           # bootstrap, below
    record(src, expected = run_node(src), got = run_k_semantics(src))
```

---

## Python

**Spec — the Python Language Reference** (docs.python.org/3/reference) + the data-model chapter. Less algorithmic than ECMA-262, so weight CPython behavior and tests more heavily as ground truth and use the reference for *intent*. The grammar file is the syntactic authority — `Grammar/python.gram` (PEG) on Python 3.9+, or `Grammar/Grammar` (LL(1)) on Python 2.7 and early 3.x.

**Conformance suite — CPython's `Lib/test/`** (stdlib `unittest`). **The version layer is a CPython git tag**: check out `v2.7.18` for "Python 2", a `v3.x` tag for a Python 3 dialect; that tag's `Lib/test/` *is* that version's conformance set. For a language-core semantics, target the core tests first — `test_grammar.py`, `test_syntax.py`, `test_scope.py`, `test_class.py`, `test_generators.py`, `test_unpack.py`, the numeric/`test_*ops` tests — and defer stdlib-heavy ones. Run via `python -m test <name>` (regrtest) or `python -m unittest`.

**Reference implementation — the CPython binary at the matching tag.** Differential check: run the program in your K semantics and in `python`, compare stdout / exception type / exit status.

```text
# sketch
checkout cpython @ v2.7.18                       # the version layer
for test in Lib/test/{test_grammar,test_syntax,...}.py:
    run each case in k_semantics; compare to expected (unittest assertions)
for src in harvested_corpus(feature):
    record(src, expected = run_cpython(src), got = run_k_semantics(src))
```

---

## Keep the reference oracle hermetic

The differential oracle is ground truth only if running the program under test does not perturb the oracle itself. A wrapper that executes the program inside the harness's own scope can leak the harness's bindings into the program and manufacture a failure the real engine never produces. Concretely: running a program via `(0, eval)(src)` in a Node wrapper executes it in global scope, so a program declaring a global with the same name as a wrapper local throws a spurious "already declared" error; a Python `exec(src, ns)` oracle that reuses a populated namespace leaks names the same way. Isolate the wrapper — run the program inside a fresh function scope / IIFE, give it a clean namespace, and use deliberately obscure internal names as a belt. A leaking oracle does not fail loudly; it silently relabels good vectors as failures and poisons every comparison drawn from it.

---

## Bootstrap by differential labeling

When the static suite is thin for a feature (common outside the well-trodden core), manufacture coverage the way the Bitcoin project turned a few thousand vectors into 288K checked inputs:

1. **Harvest** real programs exercising the feature — open-source corpora (npm/PyPI sources), the reference implementation's own example/test files, or a fuzzer (Fuzzilli for JS, Atheris/Hypothesis for Python).
2. **Label** each by running the **reference implementation** and capturing its result (value | exception | output).
3. **Diff** your semantics against that label. Any disagreement is a bug to fix or a documented, justified `xfail`.

This is the single highest-leverage move for pushing past the static suite's blind spots, and it scales coverage without hand-writing vectors. Keep the harvested-and-labeled set in the regression gate alongside the static suite.

## Tagging the no-regression gate

Persist a green snapshot per layer (passing count + commit). Step 5 re-runs the full tagged set; a drop in any earlier layer's count blocks advancement. The tag taxonomy (`layer:es5`, `feature:closures`, `source:test262`, `source:harvested`) is what lets one command serve both the targeted inner loop and the exhaustive outer gate.
