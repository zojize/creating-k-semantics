# creating-k-semantics

A [Claude Code](https://claude.com/claude-code) skill for **building an executable formal [K-Framework](https://kframework.org/) semantics of a real programming language** (JavaScript, Python, a bytecode, a DSL) — oracle-driven and test-first.

## The method

Building a semantics for a large language is tractable, and safe to run agentically, only because of one idea: **the oracle is the spec.** Every change is a *verifiable repair* against ground truth the system can check itself — a conformance vector flips green, a reference implementation agrees on a concrete input, a rule matches its cited spec clause. Two nested loops drive it:

- **Inner loop (per feature):** read the spec clause → write the minimal K rules → run the oracle → iterate red→green.
- **Outer loop (per version):** formalize language versions oldest-first (ES5 → ES2015 → …); a layer is *done* only when its whole conformance subset is green **and** a full re-run shows zero regression in every earlier layer.

The skill encodes the oracle gate, the per-language harness wiring (test262 + V8/Node + ECMA-262 for JS; CPython `Lib/test` + CPython for Python), the dependency-ordered no-regression layering, the human/judge review boundaries, and the anti-overfitting discipline. It is K-Framework-specific but language-agnostic; [Bitcoin Script](https://github.com/zojize/bitcoin-script) is the proven success story, kept in a reference file so the method stays decoupled from it.

## Layout

```text
SKILL.md                                  the method (loaded when the skill runs)
references/oracle-harness.md              concrete JS/Python oracle wiring
references/k-patterns.md                  transferable K idioms (verbatim from the K PL tutorial)
references/case-study-bitcoin-script.md   the decoupled worked example
```

## Install (symlink into Claude Code)

Clone anywhere, then symlink the repo into your skills directory so edits here stay live:

```sh
git clone https://github.com/zojize/creating-k-semantics ~/dev/creating-k-semantics
ln -s ~/dev/creating-k-semantics ~/.claude/skills/creating-k-semantics
```

Invoke it with `/creating-k-semantics`, or let it fire automatically when you start building a K semantics for a language.
