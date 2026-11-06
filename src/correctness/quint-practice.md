# Quint in practice

What actually works when writing Quint specs, much of it learned the hard way. Commands are for Quint 0.33. Examples come from the [perpetual-futures liquidation model](perp-exploit.md).

## The build loop

**A spec is executable from its first line, so run it from its first line.** Never add more than one action before checking the previous one:

```sh
quint typecheck spec.qnt                     # after every edit
quint run spec.qnt --main=perp --max-steps=20   # after every new action
```

Typechecking is not enough. A spec can typecheck and still fail to run. The most common reason is a `const` with no value: it typechecks, and `quint run` fails with `Uninitialized const`. Give constants values in a small **instance module** and run that:

```quint
module perp {
  import perp_liquidation(
    TRADERS = Set(1, 2, 3),
    INITIAL_COLLATERAL = 100,
    INITIAL_FUND = 20,
    IM_PCT = 10,     // initial margin, percent
    MM_PCT = 5,      // maintenance margin, percent
    ...
  ).*
}
```

Instance modules double as **parameter sweeps**. The liquidation study defines `calmGame`, `crashGame`, `highIM` and `highIMandMM` as one-line instances of the same model with different margins and price gaps, then runs the same property against each.

## Three layers, and the guard/apply split

Separate every spec into three layers, and keep them separate:

| Layer | Quint | Contains |
|---|---|---|
| Functional logic | `pure def` | All business rules: guards, computations, next-state functions |
| State machine | `var`, `action` | Only wiring: a guard and an assignment |
| Properties | `val` (and `temporal`) | Invariants, witnesses, liveness |

Within the functional layer, split each operation into two functions: **can it happen?** and **what results?**

```quint
pure def canTrade(st: State, buyer: int, seller: int, q: int): bool = and {
  buyer != seller,
  q > 0,
  meetsInitial(withPos(st.traders.get(buyer), q), st.price),
  meetsInitial(withPos(st.traders.get(seller), -q), st.price),
}

pure def applyTrade(st: State, buyer: int, seller: int, q: int): State =
  { ...st, traders: st.traders.setBy(buyer, a => withPos(a, q))
                              .setBy(seller, a => withPos(a, -q)) }

action trade(buyer: int, seller: int, q: int): bool = all {
  canTrade(s, buyer, seller, q),
  commit(applyTrade(s, buyer, seller, q), Trade),
}
```

> [!WARNING]
> **Don't fuse the guard into the update.** A single function returning `{ success, newState }`, with the old state on failure, turns a disabled action into a no-op step. The model then contains a transition the real system can't take, and it hides deadlocks.

## Witnesses first, then invariants

Before checking that nothing bad happens, check that the interesting things *can* happen. A **witness** is a state you expect to be reachable. Add one per action, and run them before any invariant:

```quint
var lastAction: Action   // records which action fired last, so each one gets a witness

val liquidationFired: bool = lastAction == Liquidate
val adlFired: bool = lastAction == Adl
```

```sh
quint run spec.qnt --main=perp --witnesses liquidationFired adlFired --max-steps=40
# → adlFired was witnessed in N trace(s) out of 10000 explored (P%)
```

**A witness at 0% is a dead action**: its guard can never be true, so every invariant that depends on it is vacuously safe. Fix that before trusting anything else.

Then write invariants, starting with **conservation laws**. In a financial model they are the cheapest and most powerful sanity check, because almost every modeling bug breaks one:

```quint
// A perp moves money between accounts; it never creates or destroys it.
val conservation: bool =
  totalCollateral == TRADERS.size() * INITIAL_COLLATERAL + INITIAL_FUND
val balancedOpenInterest: bool = openInterestNet == 0
```

Only then add the protocol properties you actually care about (no account below zero, the insurance fund solvent, no stuck bad debt), and expect some of them to fail: that's the point.

## Encode assumptions as alternative steps, not edits

When a property only holds under an assumption, such as "keepers liquidate promptly", don't edit the model. Add a second step relation that encodes the assumption and select it on the command line:

```quint
// Prompt keepers: the oracle cannot move while any account is liquidatable.
action stepPrompt = {
  ...
  any {
    trade(t, u, q),
    all { not(anyLiquidatable), moveOracle(d) },
    liquidate(t),
    ...
  }
}
```

```sh
quint run spec.qnt --main=perp --step=stepPrompt --invariant=fundSolvent
```

Now the same model answers two questions: what goes wrong in general, and what goes wrong even when the assumption holds.

## `quint run` versus `quint verify`

| Use | Command | Notes |
|---|---|---|
| Every iteration while building | `quint run --invariant=X --max-steps=N` | Default 10,000 samples. Use `--seed` to reproduce a trace and `--verbosity=3` to see nondeterministic picks. |
| Reachability | `quint run --witnesses A B C` | Reports the percentage of traces reaching each |
| Deterministic scenarios | `quint test` | `run` definitions in a separate `*_test.qnt` file |
| Exhaustive check up to a bound | `quint verify --invariant=X --max-steps=N` | Apalache. Needs Java. Minutes to hours. Reports deadlocks. |
| Traces for other tools | `quint run --out-itf=trace.itf.json` | Informal Trace Format, for viewers and model-based testing |

Reach for `verify` deliberately: when the sampled runs are clean but the property is critical, or when the bug you're hunting needs a specific deep sequence. In the liquidation study, the risk-free attack sits in a corner random walks rarely enter, and Apalache found it exhaustively.

## Gotchas worth knowing in advance

- **`quint run` does not report deadlocks.** It stops early and prints a pass. A trace much shorter than `--max-steps` is the only hint.
- **`.get()` on a missing map key is undefined.** Pre-populate every map in `init` (`TRADERS.mapBy(_ => ...)`).
- **Every action must assign every variable.** A forgotten one is unconstrained in the next state. One `commit` helper that assigns everything avoids this entirely.
- **`assume` documents, it does not enforce.** To check that a premise holds (for example `2 * f < n`), write a `run` test that asserts it.
- **Names collide.** An action and a `pure def` cannot share a name, and some common words are built-ins (`to` is one, so use `dst`).
- **Integers only, by choice.** Model percentages as whole numbers and rearrange inequalities to avoid division: `collateral * 100 >= notional * IM_PCT`.
- **Keep ids opaque and messages in sets.** No string manipulation on ids, and no lists for messages unless ordering is the property under test.

## Modeling with a coding agent: quint-llm-kit

Quint is a niche language, so a general model will confidently invent syntax and standard-library operators that do not exist. Two things fix that: grounding the agent in real documentation (a docs MCP server, or repository-level instructions to look things up rather than guess), and a workflow that makes the agent run the tools after every step. [quint-llm-kit](https://github.com/quint-co/quint-llm-kit), from the Quint team (originally built at Informal Systems), packages both for Claude Code as three skills:

| Skill | Use it to | What it enforces |
|---|---|---|
| `quint-modeling` | Build a model from an idea, requirements, existing code, or a TLA+ spec; review an existing spec | A seven-step spine: separate concerns, shape the state, write pure functions, wire guarded actions, witnesses then invariants, compose `step` and simulate, self-review and hand off. `quint typecheck` and `quint run` after every step. |
| `quint-lang` | Look up syntax, operators, `basicSpells`, CLI flags; read counterexamples | A reference to check against instead of guessing |
| `quint-execute-spec` | Change code against an existing spec: refactors, features, closing gaps | Orient, research a compact gap analysis, plan, implement, verify, with Quint runs as gates between phases |

What I took from using it:

- **The human settles the design forks; the agent writes the model.** The most useful thing the agent produced was the list of questions it could not answer from the source: scope, who absorbs bad debt first, what the adversary can do. The liquidation model records them in its header as "design forks, settled with the user". Those decisions are the essential complexity; the Quint syntax is not.
- **Require the honesty note.** Ask for "what it models / what it does not" on every spec. It's the fastest way to review an agent's model, because the omissions are where it can be wrong.
- **Watch the witness percentages, not the green checkmarks.** An agent will happily report "no violation found" for a model whose key action never fires.
- **The spec is ground truth.** When implementation and spec disagree, the agent must stop and surface it, not edit the spec to match. `quint-execute-spec` makes that an explicit rule, and it is the right one.
- **Simulate by default; model-check on purpose.** The skills deliberately iterate with `quint run` and leave `quint verify` to an explicit decision, because a two-hour Apalache run inside an agent loop is mostly waste.
