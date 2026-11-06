# Study guides

Many systems share a skeleton: a state machine, an environment that loses, duplicates, reorders and delays messages, and a property that must hold in every reachable state. A ledger must always sum to zero. A quoting engine must never hold two orders on one side. When such a property breaks, the bug usually lives in an interleaving nobody thought to test.

This part of the guide is about stating those properties precisely and checking them in *every* reachable state, not just the ones a test happened to visit.

## The verification ladder

Each rung checks more behaviors than the one below it, and costs more to set up.

| Rung | What it checks | Finds | Misses |
|---|---|---|---|
| Types | Shapes of values | Mixed-up units, missing cases, null | Anything about behavior over time |
| Example tests | The scenarios you thought of | Regressions in known paths | The interleaving you didn't imagine |
| Property-based tests | Many random inputs to one function | Edge values, broken algebraic laws | Bugs that need a specific *sequence* of events |
| Simulation of a model | Many random *executions* of a design | Shallow design bugs, fast | Rare interleavings deep in the state space |
| Model checking | *Every* execution of a design, up to a bound | Races, lost-message bugs, incentive exploits | Bugs outside the bound or outside the model |
| Model-based testing | The real implementation against the model's executions | Code that drifts from the design | Behavior the model doesn't describe |

The bottom three rungs are where money-moving systems need to live, and they share one artifact: an **executable specification**, a model of the design that a tool can run, explore and use as a test oracle. This guide uses [Quint](https://quint-lang.org), with one TLA+ case study, because Quint reads like a typed programming language while keeping the semantics of TLA+.

> [!TIP]
> **Test the code you have; model the design you're about to commit to.** A model is cheapest to change before code exists, and most valuable where the bug would be an interleaving nobody would think to test.

## Reading order

1. **[Specs that run: how Quint differs from code](quint-concepts.md).** The conceptual shift. A program is one execution; a specification is every allowed execution. Nondeterminism, actions as relations, properties as first-class citizens, and why abstraction is the actual design work.
2. **[Quint in practice](quint-practice.md).** The workflow that works: three layers, guards split from updates, witnesses before invariants, `quint run` versus `quint verify`, the gotchas, and how to drive modeling with a coding agent using quint-llm-kit.
3. **[Case study: a risk-free perp attack](perp-exploit.md).** A model checker finds an incentive exploit in a perpetual-futures liquidation mechanism unaided, by splitting "there exists a strategy such that for all market moves" between the checker and pure functions.
4. **[Case study: the spec as test oracle](model-based-testing.md).** Replaying the model's executions against a Rust engine, and checking that the engine accepts exactly the actions the spec allows: no more, no fewer.
5. **[Case study: proving the model can fail](broken-variants.md).** A TLA+ model of a market-making engine's order slot, with a deliberately broken variant per known bug, run as a suite where each variant must fail in the expected way. Plus the six-step race it found that would have filled past the position cap.

## What the case studies have in common

All three come from systems I built and modeled. Each is a small model: three traders, one order slot, a handful of messages. That is deliberate. Most design bugs show up with two or three participants and a few steps, and a model small enough to check exhaustively finds them in seconds to minutes. The case studies show both what that finds and where it stops.
