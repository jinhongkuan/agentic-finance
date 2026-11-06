# Specs that run: how Quint differs from code

[Quint](https://quint-lang.org) looks like a typed functional language, which is the point: engineers can read it. Underneath, it is the Temporal Logic of Actions (TLA), the same semantics as TLA+. That combination makes it easy to write Quint as if it were a program, and that habit is the main source of bad specs. This page covers the differences that matter.

## 1. A program is one execution; a spec is all of them

A program, given its inputs, takes one path. A specification describes the *set* of every path the system is allowed to take, and the tools explore that set.

```quint
action step = {
  nondet t = TRADERS.oneOf()
  nondet u = TRADERS.oneOf()
  nondet q = 1.to(MAX_TRADE).oneOf()
  nondet d = (-MAX_JUMP).to(MAX_JUMP).oneOf()
  any {
    trade(t, u, q),
    moveOracle(d),
    liquidate(t),
    unwind(t, q),
    adl(t),
  }
}
```

`nondet ... oneOf()` does not pick a random value at runtime. It says "any of these", and `any { ... }` says "any of these actions that is enabled". The simulator samples choices; the model checker considers all of them. In this model every trader, trade size, price move and liquidation order is explored, so **the adversary is the nondeterminism**. You don't write the attacker. You state what is possible, and the tools find the worst sequence.

> [!TIP]
> **Model the environment, not the strategy.** Lost messages, late keepers, price gaps, malicious counterparties: make each one a nondeterministic choice and let the checker play the adversary.

## 2. Actions are relations, not assignments

In code, `x = x + 1` changes `x`. In Quint, `x' = x + 1` is a boolean claim about the next state: it is true of a pair of states where the new `x` is one more than the old. An action is a predicate over (current state, next state). That has three consequences that trip up programmers:

- **A false guard disables the action.** It doesn't throw, return an error, or fall through to a default. In that state, the transition simply does not exist. "This can't happen here" is modeled by the action being disabled.
- **Every action must say what happens to every variable.** If an action forgets `y' = y`, the next value of `y` is unconstrained, which is a silent bug. Grouping state into one record with one `commit` helper avoids it.
- **There is no control flow across steps.** No loops over time, no callbacks, no "then". Ordering emerges from which actions are enabled in which states.

```quint
action liquidate(t: int): bool = all {
  canLiquidate(s, t),                         // guard: false here means "cannot happen"
  commit(applyLiquidate(s, t), Liquidate),    // assigns every variable
}
```

## 3. Logic lives in pure functions; the state machine only wires them

Quint has two layers. `pure def`s are ordinary functions over values, testable in the REPL with a hand-built state. Actions read state and assign next state. Good specs keep all business logic in the pure layer and keep actions to a guard and a commit. That split is what makes the [game-theoretic case study](perp-exploit.md) possible: the "for every market move" part of the question is a pure function folding over all moves, while the "there exists a strategy" part is the model checker searching action sequences.

## 4. Properties are first-class, and they quantify over all executions

A test asserts something about one run. A specification states properties the tools check against every reachable state:

| Kind | Question | Example |
|---|---|---|
| **Invariant** | Does this hold in every reachable state? | Money is conserved: the sum of all collateral never changes |
| **Witness** | Is this state reachable at all? | A liquidation actually happens in some trace |
| **Temporal (liveness)** | Does something good eventually happen? | A pending request is eventually resolved, if the exchange eventually answers |

```quint
val conservation: bool =
  totalCollateral == TRADERS.size() * INITIAL_COLLATERAL + INITIAL_FUND
```

Witnesses have no equivalent in ordinary testing, and they matter as much as invariants: an invariant that holds because the interesting action can never fire is a false sense of safety.

## 5. No I/O, no time, no performance: abstraction is the design work

A spec has no network, no clock, no database and no serialization. Whatever you need from them, you model: a set of in-flight messages that can be delivered in any order or dropped; a timeout as an action that becomes enabled; a database write as a state change. Deciding what to leave out is the real engineering decision, because every abstraction is a claim that the omitted detail cannot break the property. Good specs say so at the top:

```text
WHAT IT MODELS        N traders, isolated margin, continuous PnL settlement,
                      an adversarial oracle, late keepers, liquidation, ADL
WHAT IT DOES NOT      order book, slippage, fees, funding, partial liquidation,
                      cross margin, multiple markets
DESIGN FORKS SETTLED  bad debt: insurance fund first, then ADL
```

That honesty note is what stops a model from being over-trusted.

## 6. Sampling is not checking

Quint has two engines, and they answer different questions:

| | `quint run` (simulator) | `quint verify` (model checker, Apalache) |
|---|---|---|
| How | Random walks through the state space | Symbolic, exhaustive search up to a step bound, using an SMT solver |
| Speed | Seconds | Minutes to hours |
| A pass means | No violation in the traces sampled | No violation in *any* execution up to the bound |
| Good for | Building and iterating, witnesses, shallow bugs | Rare interleavings, final confidence, incentive exploits |

In the perp model, the simulator ran 20,000 traces without finding the attack at one parameter setting, while Apalache found an attack exhaustively in about two minutes at another. A clean simulation run is evidence, not proof.

## 7. The output is a counterexample, and it debugs the design

When an invariant fails, the tools return a trace: a numbered sequence of states and the action taken at each step. That trace is a story about the design ("the probe overtakes the order, the slot concludes the order never existed, places another, both fill"), and it is usually the most useful artifact the whole exercise produces. It is also where you learn whether the bug is in the design or in the model, which is the question that comes up most often.

## 8. Why this matters more with coding agents

Agents are fast at the accidental part of software and unreliable at the essential part. A precise, executable spec moves the essential decisions into an artifact that can be reviewed by a person and *checked by a machine*, and then [turned into a test oracle](model-based-testing.md) for whatever code an agent writes. The rule that keeps this honest: **the spec is the ground truth. Never edit the spec to make broken code pass.** If the spec is wrong, that is a design conversation, and it is the highest-leverage review in the whole loop.
