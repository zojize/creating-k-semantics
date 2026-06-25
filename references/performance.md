# Making a K semantics fast enough to run real inputs

An executable semantics that times out on real-world programs is not done. The good news: the gap to a production engine is mostly **boundary and algorithmic cost, not the rewrite engine.** A consensus-critical K semantics closed the gap from **149× a production C engine to ~2×** through the techniques below, with zero correctness regressions — a ~70× speedup of the K path itself.

For large inputs the time goes to four places, roughly in this order: (1) O(n²) buffer assembly inside rules, (2) per-rewrite-step overhead on long loops, (3) (de)serializing the term across the Python↔LLVM boundary, (4) host-side setup. The rewrite engine is rarely the floor. Find your floor by profiling end-to-end against a noop baseline — see the result-serialization and boundary traps already in [oracle-harness.md](oracle-harness.md) ("Driving the K definition from the harness"); this file is the deeper toolkit.

## 1. Never assemble a buffer with repeated `+Bytes`/`+List` — stream an accumulator

K's `+Bytes` (and `+List`) is **O(n) per call**, so building one buffer by right-recursive concatenation of N records is O(N × total_size) ≈ **O(n²)**. This is the single most common K performance cliff and it is invisible on small inputs.

Replace buffer assembly with a **streamed accumulator threaded through a tail-recursive walker** that consumes one record per step and never materializes the whole buffer. Carry a **byte offset**, advanced by each record's length — *not a record index*; an index forces `recordAt` to re-walk from the start every step and silently reintroduces the O(n²):

```k
  rule walk(St, _Buf, _Off, 0) => St
  rule walk(St, Buf, Off, N)
    => walk(update(St, recordAt(Buf, Off)), Buf, Off +Int recordLen(Buf, Off), N -Int 1)
    requires N >Int 0
```

If the accumulator is a hash or checksum, expose it as `Init`/`Update`/`Final` hooks and stream chunks through `Update`. *Measured: a 5,569-record stress input went 19.8 s → 508 ms (40×) from this change alone.*

## 2. Move a hot linear loop into a native bulk hook — but keep a pure-K twin

Even after removing the quadratic cost, a K-side walker pays the backend's **per-rewrite-step overhead (~tens of µs/step)**; a loop of thousands of steps is still hundreds of ms. Push the irreducible primitive loop into native code, but **keep the semantic structure in K so the prover can still reason**:

```k
  syntax Acc ::= bulkStream(Acc, Bytes, Int, Int)  [hook(EXT.bulkStream)]   // C does the tight loop
  rule walk(St, B, O, N) => bulkStream(St, B, O, N)                          // execution delegates
  rule bulkStream(St, B, O, N) => walkPure(St, B, O, N)  [simplification]    // prover unfolds to the twin
```

The original walker is renamed `walkPure` and kept. The `[simplification]` lemma makes the LLVM backend call the hook for speed while the Haskell prover rewrites the hook back into the inductive pure walker. **This equivalence is *asserted*, not derived — it is your trust boundary,** so a C/K divergence must surface as an oracle-checked output mismatch. That is exactly what the conformance suite guarantees: any disagreement fails a vector.

## 3. Decide the native-hook boundary up front

Primitives the rewrite engine cannot or should not execute rule-by-rule — crypto, big-integer kernels, host calls — belong in native hooks **from the start**, not as a late optimization. Declare them `[hook(NAMESPACE.op)]` backed by a plugin; K constructs the inputs and selects/checks the predicate, the hook computes the primitive. Keep the byte-level kernel out of the rewrite engine entirely. Layer technique 2 on top when you also want a pure-K twin for proofs.

## 4. Build the initial term with the binary KORE C API, not the text serializer

Feeding a large binary or string input into K through pyk's **text** KORE serializer is O(input size) with a large constant — every byte of a `Bytes`/`String` literal expands to a multi-character escape, and that serialization dominates wall-clock long before the rewrite engine does.

Construct the top cell through the **kllvm-c binary constructors** via ctypes instead: `kore_pattern_new_token_with_len(data, len, sort)` per domain value, `kore_pattern_new_injection(dv, valueSort, SortKItem)`, then `kore_composite_pattern_new(label)` + `add_argument` to assemble the config map and `LblinitGeneratedTopCell`. Hand raw bytes straight to the token constructor; stringify scalars. Keep the text path only as a debug/subprocess fallback. *Measured: a ~1 MB input went 365 ms → 216 ms (1.69×).* (This is the construction-side complement to reading results by matching the KORE `Pattern` AST instead of parsing text — see oracle-harness.md.)

## 5. Blank large invariant cells at the terminal phase with `[concrete]` cleanup

A run's cost includes serializing the **final** configuration back out. Any large cell that no rule reads after the terminal phase still rides into the result pattern and gets dumped/re-parsed every input — frequently the dominant per-input cost (one stress profile: 78% of wall-clock was parsing a 3.5 MB result). Reset those cells once execution is done:

```k
  syntax KItem ::= "#cleanup"
  rule <k> .K => #cleanup </k> <phase> done </phase> ...                 [concrete]   // trigger after k drains
  rule <k> #cleanup => .K </k> <bigCell> _ => .Bytes </bigCell> ...      [concrete]   // blank no-longer-read cells
```

The **`[concrete]` tag is essential**: it makes the rules execution-only, so the Haskell prover's symbolic execution never places `#cleanup` on the `k` cell — the cleared cells stay the fresh variables `kprove` already treats as unchanged, and every proof discharges identically. **Deliberately preserve any cell a proof spec asserts about.** *Measured: result pattern 3.5 MB → ~2 KB, per-input 227 ms → 23 ms (9.8×).* (oracle-harness.md names the "clear result-only cells" symptom; this is the proof-safe mechanism behind it.)

## 6. Bound native-backend memory in long runs

Running a whole conformance suite is a long loop over a native-backed interpreter, and **that backend leaks host memory the managed side cannot free** — the LLVM allocation arena grows unbounded across calls, and `ctypes.CDLL` never unloads the dylib, so static state persists. Past a few GB the OS kills the process; the run dies on RSS, not latency.

- **Moderate runs:** periodically tear down and recreate the interpreter (drop the reference, force a GC, re-instantiate) every N inputs.
- **Very long runs:** that is insufficient (the dylib never unloads). Split the workload into fixed-size chunks and run **each chunk as a separate subprocess that exits** to fully release the arena, then merge the chunk result files. Add a `--resume` flag that reuses completed chunks so an interrupted run recovers.

## 7. Measure the honest path

A shortcut that makes the benchmark fast by **not actually running the semantics** is a measurement lie. Feeding the engine a precomputed result for most inputs, or skipping the expensive phase for a subset, reports a fictitious ratio — one such run *looked* like 2.4× when the honest number was 149×. Route every input through the same full K-native path the production verifier uses; if you add a temporary shortcut to dodge a known-slow path, record it and **revert it in the same commit that lands the real fix.** Subtract a noop baseline (engine on a trivial input) so the timed number isolates rewrite cost from per-call boundary cost, and watch the host side too: the O(n²)-concat trap (technique 1) exists in the Python that feeds K — accumulate into a list and `join` once, never `+=` in a loop.

---

These are general K facts, illustrated with measured results from the [Bitcoin Script case study](case-study-bitcoin-script.md). None depends on the language being formalized: a JS or Python K semantics builds large source/heap terms, runs long suites, and serializes big result configurations, so it hits the same four cost centers.
