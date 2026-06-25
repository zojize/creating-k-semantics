# K patterns for programming-language semantics

Transferable K idioms for formalizing a real language, each anchored to a **verbatim** snippet from the K PL tutorial (LAMBDA → IMP → IMP++ → SIMPLE) and mapped to JS/Python. Source: `github.com/runtimeverification/pl-tutorial`; provenance is `path:lines` in that repo. Snippets are exact K, lightly trimmed of literate-doc artifacts only where noted.

These are the *language* idioms. The Bitcoin case study applies the same idioms in a stack-machine shape (one big config, a centralized execution guard, structured failure, native hooks) — see [case-study-bitcoin-script.md](case-study-bitcoin-script.md).

## Two meta-lessons before the patterns

**Build up in order, one concern at a time.** The tutorial is deliberately additive — LAMBDA (pure expressions) → IMP (imperative state) → LAMBDA++ (control) → IMP++ (threads + I/O) → SIMPLE (full language), and its README says *"do them in order."* Within IMP, each lesson lands one concern and stays `kompile`/`krun`-able: lesson 1 is pure syntax, lesson 2 adds only the configuration, lesson 3 adds the first rule, lesson 4 the rest. This is the same chronological, stays-green discipline the outer loop demands — layer by version/feature, keep each layer runnable before adding the next.

**The configuration grows one cell per concern.** IMP starts at two cells (`<k>` + `<state>`). Aliasing/closures force splitting `<state>` into `<env>`+`<store>`. I/O adds stream cells. Concurrency wraps the thread-locals in `<thread multiplicity="*">`. By SIMPLE the machine has `<control>` (nesting `<fstack>` + `<xstack>`), `<genv>`, `<store>`, `<nextLoc>`, `<input>`, `<output>`, and more. Because of **configuration abstraction** (rules cite only the cells they touch), cell count tracks *feature* count, not *rule* count — adding a cell rarely disturbs existing rules. Corollary, from IMP++ lesson 2's own README: if you *know* closures/exceptions/threads are coming, adopt `<env>`+`<store>` and a control stack from the start rather than paying the refactor tax mid-stream.

---

## Front end: tokens and transforms the grammar can't express

The tutorial languages parse straight from `$PGM:Pgm` using builtin `Int`/`Id`/`String` tokens. A real language breaks both halves of that assumption, in ways that are non-obvious facts about K's scanner and cost real time to rediscover.

**A parse-time transform the grammar can't express → a separate source→source pass.** K's GLR parser is error-driven and discards whitespace and newlines as layout, so context-sensitive lexing the spec mandates is not expressible as a production: JavaScript Automatic Semicolon Insertion, Python/Haskell significant indentation, the JavaScript `/`-division-versus-regex disambiguation. Model it as a *second K definition* that rewrites source text into normalised text (an in-K lexer plus the transformation), then let the evaluator grammar parse the normalised output, where the irregularity is gone (explicit terminators, explicit block delimiters). Two kompiled definitions, a thin harness moving one string between them — the front-end shape KJS uses for ASI, and the one any layout-sensitive language needs.

**Custom-token gotchas — a real lexer hits all of these:**
- A `$`-bearing identifier token collides with the builtin `KConfigVar` token used for `$PGM` in the configuration. Tag the token `[token, prec(-1)]` so `KConfigVar` and keyword terminals win the scanner tie *inside the definition*, while the identifier still lexes in a *program* (which contains no `KConfigVar`).
- A custom numeric or string token collides with builtin `Int`/`String` and can corrupt K's *own* library rules. Keep custom number tokens disjoint from `Int`; transcode source strings into the builtin `String` sort rather than defining a string token.
- **K's scanner is ASCII-only and byte-based.** It rejects non-ASCII in a token char-class and does not interpret `\uXXXX` in a token regex; K `String`s are byte sequences (`lengthString("café") == 5`). A Unicode identifier therefore cannot be a grammar token — recognise it in the source→source pass (test `ordChar(c) >= 128`) and carry it as a wrapped builtin `String` (e.g. an `@id("<utf-8 bytes>")` node), not a token.
- `{ }` is reserved for K projections, so a List-sort-in-braces (`{ Ss:Stmts }`) never matches in a rule — use explicit block productions (`Block ::= "{" "}" | "{" Stmts "}"`).
- Overlapping function or rule cases are nondeterministic unless ordered with `[priority(N)]` (lower tried first) or `[owise]`.

## Foundations

**Syntax is semantics.** A K definition opens as a BNF grammar; the AST *is* the term rules rewrite. Sorts double as static categories and the runtime term universe.
```k
  syntax AExp  ::= Int | Id
                 | AExp "/" AExp              [left, strict]
                 | "(" AExp ")"               [bracket]
                 > AExp "+" AExp              [left, strict]
  syntax BExp  ::= Bool
                 | AExp "<=" AExp             [seqstrict]
                 > BExp "&&" BExp             [left, strict(1)]
```
`1_k/2_imp/lesson_1/imp.k:5-14` → these productions are your ESTree / Python `ast` node kinds; a `>` block encodes the operator-precedence table; `[bracket]` is the no-semantics grouping node.

**Configuration = the machine state**, one declarative term of named cells. Minimal shape: a `<k>` computation cell seeded with `$PGM`, plus state.
```k
  configuration <T color="yellow">
                  <k color="green"> $PGM:Pgm </k>
                  <state color="red"> .Map </state>
                </T>
```
`1_k/2_imp/lesson_2/imp.k:33-36` → `<k>` is the interpreter's continuation/agenda; `<state>` is the variable-environment Map; `$PGM` is the parsed module body handed to `krun`.

**Rules with cell-frame matching.** `rule LHS => RHS`; `...` matches the irrelevant cell remainder so a rule names only what it touches; `requires` adds a side condition; `_ => v` mutates in place.
```k
  rule <k> X:Id => I ...</k> <state>... X |-> I ...</state>
```
`1_k/2_imp/lesson_3/imp.k:38` → small-step reduction for an identifier-reference node: read `X` from scope, replace the node with its value. The `...` frames are what make per-node rules composable.

**Evaluation order via strictness**, declared not hand-coded. `strict` = any order, `seqstrict` = left-to-right, `strict(1)` = first arg only (short-circuit). This is the single biggest leverage point over a hand-written interpreter.
```k
  syntax Exp ::= Val | Exp Exp  [strict, left]
  syntax BExp ::= AExp "<=" AExp  [seqstrict]
                | BExp "&&" BExp  [left, strict(1)]
```
`1_k/1_lambda/lesson_3/lambda.k:8-10`, `1_k/2_imp/lesson_1/imp.k:5-14` → `seqstrict` is left-to-right operand evaluation (`a + b`); `strict(1)` is `&&`/`||`/`if`-test short-circuiting. Encode the spec's evaluation-order clauses here.

**Heating/cooling** is what `strict` desugars to: a heating rule lifts a subterm onto `<k>` with a `[]` hole, a cooling rule plugs the result back. You rarely write these, but knowing them explains "stuck."
```k
    A1<=A2 => A1 ~> []<=A2
    A1 ~> []<=A2 => A1<=A2
    I1<=A2 => A2 ~> I1<=[]
    A2 ~> I1<=[] => I1<=A2
```
`1_k/2_imp/lesson_3/README.md:69-82` → the framework auto-generating "evaluate this subexpression, then resume." A rule "stuck" on an un-reduced argument almost always means a missing `strict`.

**KResult marks finished values.** Declaring `syntax KResult ::= Val` tells K which terms are done; heating stops at KResults, cooling only plugs KResults back. This is what makes strictness terminate.
```k
  syntax KResult ::= Val
  rule (lambda X:Id . E:Exp) V:Val => E[V / X]
```
`1_k/1_lambda/lesson_6/lambda.k:60-62` → KResult is your value universe (number, string, boolean, object ref, closure). A node is "done" iff it is a KResult.

**Sequencing with `~>` and `.K`.** `<k>` is a list of tasks joined by `~>` ("then"); `.K` is empty. Sequencing, block dissolution, and "do this then resume that" are all rewrites into `~>`.
```k
  rule {} => .K
  rule {S} => S
  rule S1:Stmt S2:Stmt => S1 ~> S2
```
`1_k/2_imp/lesson_4/imp.k` (block + sequencing rules, excerpted) → `~>` is the continuation queue; pushing an auxiliary task after a body (`S ~> setEnv(Env)`) is how you schedule `finally`/cleanup without a separate stack object.

**Macros desugar derived constructs** at parse time, so the core never sees the sugar.
```k
  syntax Exp ::= "let" KVar "=" Exp "in" Exp [macro]
  rule let X = E in E':Exp => (lambda X . E') E
```
`1_k/3_lambda++/lesson_1/lambda.k:24-25` → use for spec-defined desugarings: JS `a ||= b`, default params, Python comprehensions, annotated assignment. Keeps the core rule count small.

## State, scope, and values

**Env + store indirection.** Split the single state Map into `<env>` (name → location) and `<store>` (location → value). Two-hop lookup is the prerequisite for aliasing, mutation, closures, and threads.
```k
  configuration <T color="yellow">
                  <k color="green"> $PGM:Stmt </k>
                  <env color="LightSkyBlue"> .Map </env>
                  <store color="red"> .Map </store>
                </T>
```
`1_k/4_imp++/lesson_2/imp.k:41-45` → `<env>` is the lexical scope object, `<store>` is the heap of cells; this is JS object-reference semantics and Python's name→PyObject indirection, and why two names can alias one mutable cell.

**Fresh allocation with `!N`.** `!N:Int` mints a globally fresh value each time the rule fires — no manual counter threaded through rules.
```k
  rule <k> int (X,Xs => Xs); ...</k>
       <env> Rho => Rho[X <- !N:Int] </env>
       <store>... .Map => !N |-> 0 ...</store>
```
`1_k/4_imp++/lesson_2/imp.k:71-73` → `!N` is the allocator handing out a new heap address / object identity on each declaration or `new` — the K equivalent of a fresh reference / CPython `id()`.

**Declaration = bind + allocate + initialize**, and seed the slot with a distinguished non-value so use-before-init gets stuck instead of silently succeeding.
```k
  syntax KItem ::= "undefined"
  rule <k> var X:Id; => .K ...</k>
       <env> Env => Env[X <- L] </env>
       <store>... .Map => L |-> undefined ...</store>
       <nextLoc> L => L +Int 1 </nextLoc>
```
`2_languages/1_simple/1_untyped/simple-untyped.md:441-447` → JS `let x;` (TDZ vs `undefined`) / Python local creation. The non-KResult `undefined` is what makes "ReferenceError on use-before-assignment" conformance cases fail to progress, matching the reference impl.

**Block scope via save/restore.** Snapshot `<env>` on entry, schedule its restoration after the body via the `~>` agenda. No extra scope-stack cell needed.
```k
  rule <k> { S } => S ~> setEnv(Env) ...</k>  <env> Env </env>
```
`2_languages/1_simple/1_untyped/simple-untyped.md:778` → JS block scoping / Python scope entry-exit; the saved-env-as-continuation is a scope frame push/pop reusing `~>`.

**Closures capture the defining env.** A function value freezes the `<env>` present at definition; applying it switches to the captured env (plus the parameter at a fresh location) and schedules caller-env restoration.
```k
  syntax Val ::= closure(Map,Id,Exp)
  rule <k> lambda X:Id . E => closure(Rho,X,E) ...</k>
       <env> Rho </env>
  rule <k> closure(Rho,X,E) V:Val => E ~> Rho' ...</k>
       <env> Rho' => Rho[X <- !N] </env>
       <store>... .Map => (!N:Int |-> V) ...</store>
```
`1_k/3_lambda++/lesson_6/lambda.md:110-116` → `closure(Rho,X,E)` *is* a JS function's `[[Environment]]` / Python's `__closure__` — lexical (not dynamic) capture, which JS/Python require.

**K `Map` is unordered — keep a parallel key `List` when enumeration order is observable.** `keys(M)` yields no guaranteed order. When the language exposes property/key order — JavaScript `for-in` and `Object.keys`, Python `dict` insertion order — store an insertion-ordered `List` of keys beside the `Map` and iterate *that*, updating both together on insert and delete. Relying on `Map` iteration order passes locally and then diverges from the reference impl under a different key set — a bug the no-regression gate catches late but the differential oracle catches early.

## Control: calls, exceptions, concurrency, I/O

**Calls/returns via a control stack.** A `<control>` cell holds an `<fstack>`; calling pushes a `(Env, K, ControlContext)` frame and switches env; `return` pops it and reinstalls the caller's continuation.
```k
  rule <k> lambda(Xs,S)(Vs:Vals) ~> K => mkDecls(Xs,Vs) S return; </k>
       <control>
         <fstack> .List => ListItem((Env,K,C)) ...</fstack>
         C
       </control>
       <env> Env => GEnv </env>
       <genv> GEnv </genv>
```
`2_languages/1_simple/1_untyped/simple-untyped.md:699-705` → `<fstack>` is the activation-record stack; the saved triple is a stack frame (return address = continuation `K`, saved scope = `Env`); switching to `GEnv` encodes static scoping for top-level functions.

**Exceptions via a saved handler stack.** `try` pushes a handler onto `<xstack>`; `throw V; ~> _` discards the pending computation, pops the nearest handler, restores its env/control, and runs the catch body.
```k
  rule <k> throw V:Val; ~> _ => { var X = V; S2 } ~> K </k>
       <control>
         <xstack> ListItem((X, S2, K, Env, C)) => .List ...</xstack>
         (_ => C)
       </control>
       <env> _ => Env </env>
```
`2_languages/1_simple/1_untyped/simple-untyped.md:914-920` → `~> _` discarding the continuation is non-local unwinding (JS `throw` / Python `raise`); restoring `(Env,K,C)` unwinds the call/scope stack to the handler frame. Conformance suites exercise this heavily.

**Concurrency via multiplicity cells.** Give thread-local cells `multiplicity="*"`; create a thread by rewriting `.Bag` into a fresh `<thread>`; a finished thread GCs back to `.Bag`.
```k
  rule <thread>...
         <k> spawn S => !T:Int ...</k>
         <env> Env </env>
       ...</thread>
       (.Bag => <thread>...
               <k> S </k>
               <env> Env </env>
               <id> !T </id>
             ...</thread>)
```
`2_languages/1_simple/1_untyped/simple-untyped.md:947-955` → multiplicity-`*` cells are the set of live agents (Web Workers / event-loop tasks / Python threads); modeling them as a multiset gives you the interleaving non-determinism a concurrency-aware semantics must expose.

**I/O as stream-attributed list cells.** `<input stream="stdin">` and `<output stream="stdout">`; `read()` consumes the head, `print` appends, and a `context` declaration hand-specifies evaluation order for the variadic builtin.
```k
  rule <k> read() => I ...</k>
       <input> ListItem(I:Int) => .List ...</input>
  context print(HOLE:AExp, _);
  rule <k> print(P:Printable,AEs => AEs); ...</k>
       <output>... .List => ListItem(P) </output>
```
`1_k/4_imp++/lesson_4/imp.k:56-57, 88-90` → the stream cells are stdin/stdout (`input()`/`print`, `process.stdin`/`console.log`); `context … HOLE …` is the escape hatch when declarative `strict` is too coarse for a builtin's bespoke argument order.

## Types are a second semantics over the same grammar

**A type checker reuses the grammar but defines its own configuration and rules** — rewriting expressions to *types* instead of values. One source can thus carry an execution semantics, a static-typing semantics, and a dynamic-typing semantics.
```k
  configuration <T color="yellow">
                  <tasks color="orange">
                    <task multiplicity="*" color="yellow" type="Set">
                      <k color="green"> $PGM:Stmt </k>
                      <tenv multiplicity="?" color="cyan"> .Map </tenv>
                      <returnType multiplicity="?" color="black"> void </returnType>
                    </task>
                  </tasks>
                  <gtenv color="blue"> .Map </gtenv>
                </T>
```
`2_languages/1_simple/2_typed/1_static/simple-typed-static.md:336-345` → this is how you add a TypeScript-style checker or a mypy-style Python checker over the same AST: same nodes, a typing configuration (`<tenv>`, `<returnType>`), typing rules. One conformance corpus can drive both the runtime and the type checker.

## Anti-patterns

- **Starting with one monolithic state Map when closures/exceptions/threads are coming.** You will be forced to split into `<env>`+`<store>` later, and that split revisits every rule that touched `<state>`. Adopt env+store and a control stack up front for a real language.
- **Hand-writing argument-ordering rules instead of declaring strictness.** Reserve hand-written `context` only for genuinely irregular cases (variadic `print`).
- **Forgetting `syntax KResult ::= …` for a value sort.** Strictness never sees it as "done," so subterms never cool back and rules silently get stuck. Every runtime value sort must be a KResult.
- **Over-specifying cells in rules** instead of using `...` frames. Brittle: breaks the moment a cell is nested deeper or added. Match only what you touch.
- **Initializing fresh slots with a real value** instead of `undefined`. Use-before-init then silently succeeds and diverges from the reference impl.
- **Building closures/exceptions/threads on substitution or a single global scope.** Substitution (LAMBDA) does not scale to mutation, aliasing, or shared state; switch to closure values over env+store and saved-continuation control stacks.
- **Bolting type checking into the execution rules.** Keep it a separate rewrite theory over the same grammar.
- **Threading a manual global counter** for allocation/thread-ids instead of `!N`/`!T` freshness — a frequent source of accidental collisions.
- **Relying on `Map` iteration order** where the language exposes enumeration order (JS `for-in`, Python `dict`). K `Map`s are unordered; keep a parallel insertion-ordered key `List`.

## How this maps back to the case study

Bitcoin Script is a stack machine, so it has no `<env>`/`<store>`/closures — but the *same idioms* are visible: one `<T>` configuration with a cell per concern (`<stack>`, `<exec>`, `<phase>`, `<flags>`, transaction cells); a centralized execution guard (`#guardExec` over the `<exec>` list) that plays the cross-cutting role strictness/`context` play here; structured `#fail("ERR")` versus silently getting stuck (your harness must distinguish the two, exactly as it must here); and native hooks for what K cannot do natively. When you formalize JS/Python, you will reach for the env+store, control-stack, and types-as-second-semantics patterns above that a stack language never needs.
