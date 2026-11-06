# Case study: the spec as test oracle

A model proves things about a design. It says nothing about the code until something connects the two. **Model-based testing** (MBT) makes that connection: the specification generates executions, a driver replays each step against the real implementation, and the test compares the two states after every step. The spec becomes the test oracle, which is exactly what you want when an agent is writing the implementation.

This case study tests a Rust implementation of the [perp liquidation engine](perp-exploit.md) against its Quint model, using [quint-connect](https://crates.io/crates/quint-connect).

## The loop

1. Quint simulates the model and produces traces: sequences of actions with their nondeterministic choices.
2. A Rust **driver** maps each spec action to a call on the engine.
3. After every step, the test builds a snapshot of the engine's state in the spec's shape and compares it with the spec's state at that step.
4. Any mismatch, or any engine error on a step the spec allowed, fails the test with the trace that caused it.

```rust
impl Driver for PerpDriver {
    type State = Snapshot;

    // Every step in a trace was enabled in the spec, so the engine must accept it too.
    fn step(&mut self, step: &Step) -> Result {
        switch!(step {
            init => {
                self.engine = Some(Engine::new(
                    PARAMS, TRADERS, INITIAL_COLLATERAL, INITIAL_FUND, INITIAL_PRICE,
                ));
            },
            trade(t: i64, u: i64, q: i64) => self.engine().trade(t, u, q)?,
            // The spec moves the price by a delta; the engine takes an absolute oracle price.
            moveOracle(d: i64) => {
                let e = self.engine();
                e.update_price(e.price() + d)?;
            },
            liquidate(t: i64) => self.engine().liquidate(t)?,
            unwind(t: i64, q: i64) => self.engine().unwind(t, q)?,
            adl(t: i64) => self.engine().adl(t)?,
        })
    }
}

#[quint_run(spec = "perp_liquidation.qnt", main = "perp", max_samples = 200, max_steps = 40)]
fn simulation() -> impl Driver {
    PerpDriver::default()
}
```

The driver is mostly translation, and translation is where the useful mismatches show up: here the spec moves the price by a delta while the engine takes an absolute price, so the driver converts.

## Checking both directions

Comparing state after each step catches the engine doing the wrong thing. It does not catch the engine **accepting something the spec forbids**, because traces only contain actions the spec allowed. An engine that lets an undercollateralized trader open a position would pass every state comparison.

The fix is to make the set of enabled actions part of the compared state. The spec computes it as a variable, purely as instrumentation for testing:

```quint
// Which parameterized actions are enabled in `s`.
type Guards = {
  trade: Set[(int, int, int)],
  moveOracle: Set[int],
  liquidate: Set[int],
  unwind: Set[(int, int)],
  adl: Set[int],
}

pure def guardsOf(st: State): Guards = {
  trade: tuples(TRADERS, TRADERS, 1.to(MAX_TRADE)).filter(((b, sl, q)) => canTrade(st, b, sl, q)),
  liquidate: TRADERS.filter(t => canLiquidate(st, t)),
  ...
}
```

The driver computes the same thing from the engine by probing it: it tries every candidate action on a fresh clone and keeps the ones that succeed.

```rust
/// Every candidate in `domain` that the engine accepts, each tried on a fresh clone.
fn accepted<T: Ord + Copy, E>(
    engine: &Engine,
    domain: impl IntoIterator<Item = T>,
    mut action: impl FnMut(&mut Engine, T) -> std::result::Result<(), E>,
) -> BTreeSet<T> {
    domain.into_iter().filter(|&x| action(&mut engine.clone(), x).is_ok()).collect()
}
```

Now each step checks that **the engine accepts exactly what the spec allows**: no fewer actions (it rejects something legal) and no more (it accepts something illegal).

> [!TIP]
> **Compare guards, not just state.** Traces only exercise what the spec permits. To test what the implementation must refuse, put the enabled-action sets into the compared state and probe the implementation for them.

## Practical notes

- **Mirror the spec's types exactly.** The snapshot structs deserialize the spec's state from the trace, so field names and shapes must match the spec (`#[serde(rename = ...)]` bridges naming conventions). Constants must match the instance module the traces come from.
- **Generate traces at test time; don't commit them.** Committed traces go stale the moment the spec changes. Generating them in the test keeps spec and test in lockstep.
- **Keep the spec small enough to explore.** MBT samples the spec's executions (200 traces of up to 40 steps here), so the same rules about witnesses apply: an action that never fires in the spec is never tested in the code.
- **A failure has two possible culprits.** Either the code is wrong or the spec is. Decide which before changing anything, and never change the spec just to make the test pass.

## Why this matters for agent-written code

An agent implementing a money-moving component from a prose ticket has nothing precise to be checked against. Given a spec and an MBT harness, every implementation it writes is checked against every behavior the model allows, and every behavior it forbids. The human effort moves to where it belongs: getting the spec right, and reviewing the cases where the spec and the code disagree.
