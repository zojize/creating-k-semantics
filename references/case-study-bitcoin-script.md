# Case study: K-Bitcoin Script

The proven instance of this method. A two-person team (one human director + AI agents) built an executable K semantics for **modern** Bitcoin Script — legacy, SegWit v0, and Taproot/Tapscript — that passes all 1,222 of Bitcoin Core's script vectors and all but a documented 13 of its 214 transaction vectors, verifies 100,000+ mainnet blocks, and agrees with `libbitcoinconsensus` on **288,045 real inputs with zero mismatches**, at 2.1× Core's speed.

Read this for *how the abstract loop played out*, not for Bitcoin specifics. The transferable lesson is one sentence from the project retrospective:

> The design choices that make Bitcoin verifiable by every node on the network are the same choices that make it verifiable by an AI agent.

The workflow "works uncannily well when the underlying problem has a deterministic right answer the system itself can check, and much less well when the question is 'is this design good,' because there is nothing to verify against." That is the whole bet.

## The three oracles (Step 0)

| Oracle | Bitcoin instance | JS/Python analog |
| --- | --- | --- |
| Formal spec | The BIPs — BIP-16 (P2SH), BIP-66 (DERSIG), BIP-65 (CLTV), BIP-112 (CSV), BIP-141/143 (SegWit), BIP-340/341/342 (Taproot). Each BIP carries reference test vectors. | ECMA-262 / Python Language Reference |
| Conformance suite | Bitcoin Core's `script_tests.json` (1,222 cases) + `tx_valid.json`/`tx_invalid.json` (214) + per-BIP wallet vectors | test262 / CPython `Lib/test` |
| Reference implementation | `libbitcoinconsensus` (Bitcoin Core 27.2), plus mainnet itself as the final exam | V8/Node / CPython |

Bitcoin Script qualified easily: every consensus rule ships with vectors, the reference verifier is a linkable C library, and the entire blockchain is a labeled corpus. That abundance is *why* the agentic method fit.

## Chronological layering = consensus eras (Steps 2–3)

Script's history is a stack of soft-fork eras, each gated by a consensus **flag**. The team formalized them oldest-first, each era a version layer with its own test subset:

```text
pre-P2SH → P2SH → DERSIG/STRICTENC/LOW_S → CLTV → CSV
        → SegWit v0 (BIP-143) → Taproot key-path (BIP-341)
        → Tapscript (BIP-342) → legacy sighash
```

The conformance harness uses an **"excluded flags"** format — all flags active *minus* a listed few — so a new era is turned on incrementally without disturbing earlier ones. Flag ≈ era ≈ test subset ≈ version layer. The git history shows this literally: `feat(k_semantics): implement BIP-143…`, then `…BIP-341…`, then `…BIP-342…`, each a self-contained layer that left prior layers green.

Within the hardest layer (sighash), work was ordered by dependency, mirroring Step 3: tx wire-format readers → tagged-hash primitive → BIP-143 (SegWit) → BIP-341 (key-path) → BIP-342 (tapscript) → legacy. Each had a checklist in `docs/ROADMAP.md` and a smoke target (e.g. "5,575/5,575 tapscript inputs pass") before advancing.

## The verifiable-repair loop (Step 4)

No task was ever "write the semantics." Every task was a repair with a checkable acceptance criterion:

- *make this failing `script_tests.json` vector pass*
- *close this `tx_valid`/`tx_invalid` discrepancy*
- *match this BIP-341 wallet vector*
- *agree with `libbitcoinconsensus` on this mainnet input*

A failing vector **was** the specification for a small change; the full suite plus mainnet replay guarded regression. The AI agent was an implementation accelerator inside a deterministic verification loop — the ground truth was always the published tests and Core's behavior, never the model's confidence.

## The no-regression gate (Step 5)

The advance condition was a hard "all green": 1,672 tests (1,222 script + 201 tx + 22 proof specs + coverage) plus a clean mainnet replay before any era was declared done. Thirteen vectors are documented *expected* failures (`xfail`) with written justification — nine malformed `BADTX` cases that are transaction-level rather than script-level, one debatable `CODESEPARATOR` sighash edge case, and three whose invalidity depends on a deliberately excluded flag — the discipline being that every non-green case is explained, never ignored.

## Bootstrap by differential labeling (Step 1)

A few thousand static vectors became **288,190 checked inputs** by extracting real inputs from mainnet blocks and labeling each with `libbitcoinconsensus`. This is the technique that turns a thin static suite into massive coverage: harvest real programs, let the reference implementation supply the expected answer, diff. Zero mismatches over 288,045 paired rows is a far stronger correctness claim than the static suite alone.

## Custom hooks beyond K

K cannot natively compute SHA-256 or verify a Schnorr signature, and naive byte work is too slow. Two hook layers solved this — the **"custom hooks to exceed K's native capabilities"** pattern:

- A forked `blockchain-k-plugin` supplies primitive crypto (SHA/RIPEMD/ECDSA/Schnorr/secp256k1) as native hooks.
- An in-tree sighash plugin exposes **streaming SHA-256 walkers**, after a naive `+Bytes`-concatenation sighash turned out O(n²) and 500× too slow on 1 MB transactions.

The boundary is deliberate: hooks verify a predicate over bytes; **K still constructs the bytes and chooses the predicate.** The semantic structure stays in K; only the irreducible primitive leaves it. For JS/Python the analog is hooking native math, host I/O, or a perf-critical stdlib primitive — not the language's evaluation logic.

## Performance as a first-class result

The honest K-native benchmark went **149× → 87× → 2.1×** of Core across runs (v8 → v9 → merged main). The wins: binary KORE construction (skip text serialization), subprocess chunking (bound runtime memory), a concrete cleanup rule that empties large cells once execution reaches `done`, and replacing O(n²) concatenation. A cautionary note lives here too: an early run looked fast (2.4×) only because it used a precomputed-sighash *shortcut* for most inputs — the first **honest** full-K-native run was 149×. Measure the real path, not a shortcut.

## The formal-verification payoff

Because the semantics is in K, the same rules support proofs. The project carries **72 K reachability claims** — 70 discharged by the Haskell Kore prover, 2 HTLC hash-path claims checked through the LLVM execution path. Four are *historical-bug proof pairs*: a permissive claim (exploit succeeds without the flag) and a restrictive claim (flag blocks it) for MINIMALDATA, NULLDUMMY, CLEANSTACK, and DISCOURAGE_UPGRADABLE_NOPS — a machine-checked audit of four real consensus changes. This is the upside of executable semantics over a hand-written interpreter: testing and proving share one artifact.

## The human's job (oversight)

The director did not write most rules. They: chose the architecture (the configuration cells and the phase machine), set up the harnesses so the agent had something to iterate against, made the design calls a test suite cannot decide (e.g. leaving one debatable sighash case as a documented `xfail`), and reviewed and integrated what came back. That maps exactly to the three stop-for-review boundaries: **architecture, ambiguity, advancement.** The retrospective again: the same workflow "on a less-tested codebase would have collapsed in the first week."
