# Building the harness in Python: drive K through pyk, not the CLI

The harness runs on every loop iteration and *is* a real Python project — build it like one. The single rule that pays off most:

> **Drive K through the pyk library, never by shelling out to the `kompile` / `krun` / `kprove` CLI.** Subprocessing the CLI is brittle (you scrape pretty-printed text, juggle temp files, and re-implement the toolchain) and slow (a process spawn per program, over a suite of thousands). pyk does everything in-process and hands you a typed AST.

Verified across `kimp`/`imp-semantics`, KEVM (`kevm-pyk`), and the Bitcoin case study: all three register a kdist build target, kompile in-process, run via bindings, and read results from the KORE AST. This file is the build/run/prove API surface and the project hygiene; [oracle-harness.md](oracle-harness.md) covers which oracles to wire and the suite-running traps.

## Build — kdist targets, not a `kompile` subprocess

- Register the build in `pyproject.toml`: `[project.entry-points.kdist]` → `<name> = "<pkg>.<module>"`. pyk discovers it via `entry_points(group='kdist')`.
- In that module, subclass `pyk.kdist.api.Target` once per artifact (an `llvm` target, an `llvm-lib` for the C-linkable interpreter, a `haskell` target for proofs) and collect them in a module-level `__TARGETS__` dict.
- Each `Target.build` calls the **in-process** `kompile(output_dir=..., backend=PykBackend.LLVM, llvm_kompile_type=LLVMKompileType.C, hook_namespaces=..., ...)` from `pyk.ktool.kompile` — typed args, no CLI string assembly.
- Resolve the built directory at runtime with `kdist.get("<name>.<target>")` — never hardcode a build path. `kdist.build()` compiles in-process; `kdist.which()` locates without building. (Triggering a rebuild via the `kdist build` CLI is a tolerable convenience; assembling a `kompile` command yourself is not.)
- **The no-CLI rule governs the *shipped harness*, not your debugging.** `kdist`/`kompile()` swallow the K diagnostic — a grammar ambiguity or rule error comes back as a Python `CalledProcessError`/traceback, not the actual `[Error] … ` line. When a build fails, run `kompile <main>.k --syntax-module … --output-definition /tmp/scratch` directly to *read* the real diagnostic, then fix the `.k` and rebuild through `kdist`. Debugging at the CLI is fine; the durable harness is what stays library-driven.

## Run — pyk bindings, not a `krun` subprocess

- Execute concretely with `llvm_interpret(definition_dir=kdist.get(...), pattern=init, depth=...)` from `pyk.ktool.krun` — it returns a `pyk.kore.syntax.Pattern` in-process.
- **Build the initial configuration as a KORE AST**, not text: `top_cell_initializer({...})` with `inj` / `int_dv` / `bool_dv` / `map_pattern` from `pyk.kore.prelude`. (For very large embedded values, the binary KORE C API skips text serialization entirely — see [performance.md](performance.md).)
- **Read results from the Pattern AST**: `pyk.kore.match` combinators (`km.app`, `km.arg`, `km.kore_map_of`) or `KoreParser(text).pattern()` then match `App` / `DV` / `String` nodes. Reserve `kore_print` for golden-file diffs only — never regex the pretty-printed output.
- Hot loop: optionally bind the kompiled `interpreter` library directly via `ctypes` (`init_static_objects` + `take_steps`). Even pyk's `llvm_interpret` spawns the interpreter binary under the hood, so a `CDLL` binding is *strictly more* in-process (see performance.md).
- **`llvm_interpret` has no timeout, but a conformance run needs one** (a grammar bug or a missing recursion limit yields a non-terminating program). Since `llvm_interpret` just runs the kompiled `interpreter` binary on the KORE text, invoke that binary yourself with `subprocess.run(..., input=init.text, timeout=T)` and parse its stdout with `KoreParser` — same mechanism, now bounded, and still not the `krun` CLI. Surface a kill as a `timeout` outcome, not a hang.
- **Parallelize the runner.** Each program is an independent `parser_PGM` + `interpreter` subprocess, and subprocess waits release the GIL, so a `ThreadPoolExecutor` over the vectors gives near-linear speedup for free — warm the `kdist.get(...)` path once, then fan out. On a real suite this is the difference between a usable inner loop and a coffee break.

## Prove — the Kore RPC server, not a `kprove` subprocess

- Start/connect to `KoreServer` / `KoreClient` (or `BoosterServer`) from `pyk.kore.rpc`; wrap in `cterm_symbolic` → `KCFGExplore`.
- Drive reachability with `APRProof` / `APRProver(...).advance_proof(...)` from `pyk.proof`; `parallel_advance_proof` for many at once.
- Supply a `KCFGSemantics` subclass (`is_terminal` / `is_loop` / `same_loop`) so the prover detects your language's loops and termination.
- *Cautionary exception:* the Bitcoin case study subprocesses `kprove` and scrapes `#Top` from stdout because its hooks are LLVM-only and the Haskell prover can't evaluate them — a documented workaround, not the pattern to copy; it discharges fully-concrete claims by LLVM execution instead. The RPC + `APRProver` path (what kimp and KEVM use) is the norm.

## The only legitimate subprocesses

Drive K itself in-process; shell out only for genuinely **non-K** work: building a native dependency with `make` (a C/C++ hook library), or running the kompiled GLR **parser** binary (`parser_PGM`) to turn surface syntax into KORE — whose result is still an AST you parse with `KoreParser`. Everything `kompile`/`krun`/`kprove` would do, pyk does in-process.

## Project hygiene

- **PEP 621 `pyproject.toml` with the hatchling backend** (uniform across kimp/KEVM/the case study — not poetry), a **`uv` lockfile**, and run every tool through `uv run` so local and CI match. Ship a `py.typed` marker.
- **Pin the `kframework` distribution** (it ships pyk) — *exactly* (`kframework==7.1.x`) for reproducible builds, since pyk's kompile/kore/proof APIs move between releases; a `>=` floor is the weaker choice.
- **Strict typing + lint, gated in CI**, with `from __future__ import annotations` and `TYPE_CHECKING`-guarded imports throughout. Tool choice varies — newer projects use **ruff + pyright** (the case study, py3.14), the established RV stack uses flake8 + black + isort + mypy (kimp/KEVM, py3.10). Pick one and gate it; the durable practice is the *gate*, not the tool.
- One aggregated `check` target (lint + format-check + typecheck) run before every push.

## API quick reference

| Need | Use |
| --- | --- |
| Register a build | `[project.entry-points.kdist]` + `pyk.kdist.api.Target` subclasses in `__TARGETS__` |
| Compile | `pyk.ktool.kompile.kompile(...)`, `PykBackend`, `LLVMKompileType` |
| Locate/build artifacts | `pyk.kdist.kdist.get/which/build("<pkg>.<target>")` |
| Run concretely | `pyk.ktool.krun.llvm_interpret(...) -> Pattern`; subclass `KRun` for a driver |
| Build the input term | `pyk.kore.prelude.top_cell_initializer/inj/int_dv/bool_dv/map_pattern` |
| Inspect results | `pyk.kore.syntax.{Pattern,App,DV,String}`, `pyk.kore.parser.KoreParser`, `pyk.kore.match` |
| Prove | `pyk.kore.rpc.{KoreServer,KoreClient}`, `pyk.proof.{APRProof,APRProver}`, `pyk.kcfg.{KCFGExplore,KCFGSemantics}`, `pyk.ktool.kprove.KProve` |
| Non-K shell-outs only | `pyk.utils.run_process_2` for `make` / the `parser_PGM` binary |
