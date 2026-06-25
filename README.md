# creating-k-semantics

An agent skill for **building an executable formal [K-Framework](https://kframework.org/) semantics of a real programming language** (JavaScript, Python, a bytecode, a DSL) — oracle-driven and test-first.

## The method

Building a semantics for a large language is tractable, and safe to run agentically, only because of one idea: **the oracle is the spec.** Every change is a *verifiable repair* against ground truth the system can check itself — a conformance vector flips green, a reference implementation agrees on a concrete input, a rule matches its cited spec clause. Two nested loops drive it:

- **Inner loop (per feature):** read the spec clause → write the minimal K rules → run the oracle → iterate red→green.
- **Outer loop (per version):** formalize language versions oldest-first (ES5 → ES2015 → …); a layer is *done* only when its whole conformance subset is green **and** a full re-run shows zero regression in every earlier layer.

The skill encodes the oracle gate, the per-language harness wiring (test262 + V8/Node + ECMA-262 for JS; CPython `Lib/test` + CPython for Python), the dependency-ordered no-regression layering, the human/judge review boundaries, and the anti-overfitting discipline. It is K-Framework-specific but language-agnostic; [Bitcoin Script](https://github.com/zojize/bitcoin-script) is the proven success story, kept in a reference file so the method stays decoupled from it.

## Layout

```text
SKILL.md                                  the method (loaded when the skill runs)
references/oracle-harness.md              oracle wiring (JS/Python), the pyk run-substrate, hermetic oracles
references/k-patterns.md                  transferable K idioms (K PL tutorial) + real-language front-end & scanner lessons
references/case-study-bitcoin-script.md   the decoupled worked example
```

## Install

This is a portable [agent skill](https://github.com/vercel-labs/skills) — a `SKILL.md` plus reference files, with no dependency on any one agent runtime. Install it into your coding agent with the [`skills`](https://github.com/vercel-labs/skills) CLI, which is harness-agnostic and auto-detects whichever agents you have (Claude Code, Cursor, Codex, and [70+ others](https://github.com/vercel-labs/skills)):

```sh
npx skills add zojize/creating-k-semantics
```

Pass `-a <agent>` to target a specific agent (e.g. `-a claude-code`), or `-g` to install globally instead of into the current project.

### Or symlink it manually

Clone anywhere, then symlink the repo into your agent's skills directory so edits here stay live:

```sh
git clone https://github.com/zojize/creating-k-semantics ~/dev/creating-k-semantics
ln -s ~/dev/creating-k-semantics <your-agent-skills-dir>/creating-k-semantics   # e.g. ~/.claude/skills/ for Claude Code
```

Invoke it by name (e.g. `/creating-k-semantics` if your agent supports slash commands), or let it fire automatically when you start building a K semantics for a language.
