# Case study: proving the model can fail

A model that never reports a violation proves nothing until you know it *can* report one. This case study is a TLA+ model of a market-making engine's order slot, verified with the TLC model checker, where every known way to break the design is kept in the model as a switch, and the whole set runs as a suite in which **each broken variant must fail in exactly the expected way**.

It is written in TLA+ rather than Quint. The ideas carry over directly: Quint has the same semantics, and the same structure works with Quint's `--main` instances.

## The system

A quoting engine keeps one resting order per side of the book, per asset, and replaces it as its target price moves. Each side is a **slot** with five states: Empty, Pending (a request is in flight), Live, Probing (asking the exchange what happened to an order whose answer was lost) and Settling (waiting for fills to arrive before placing again). Between the slot and the exchange is a network that **delays and reorders every message and can lose answers**. Fills are never lost.

The properties, each a sentence a trader would recognize:

| Property | Kind | Plain sentence |
|---|---|---|
| `OneOrderPerSide` | Safety | A side never has more than one order open on the exchange |
| `CapHolds` | Safety | The position never exceeds the cap |
| `NoUntrackedOrder` | Safety | Every open order is the slot's current order, and the slot knows it may be open |
| `Progress` | Liveness | No slot stays in Pending, Probing or Settling forever, if the exchange eventually answers |
| `HaltedClears` | Liveness | While an asset is halted, its orders go away, unless it resumes first |

## Broken variants as constants

Each known failure mode of the design is a boolean constant that, when set, removes one design decision:

```tla
CONSTANTS
    Expiry,       \* a timed-out request can no longer take effect. FALSE only in a broken variant
    BreakD2,      \* broken variant: may place while a request is in flight
    BreakSettle,  \* broken variant: skips Settling
    BreakProbe,   \* broken variant: never probes again after a lost answer
    BreakHalt     \* broken variant: places while halted and never withdraws
```

Each configuration file sets the constants and lists the properties to check. The suite is a script that runs every configuration and **compares the outcome with an expected result**, failing if anything differs:

```bash
# config                 expected  (pass, or the exact TLC error line fragment)
CHECKS=(
  "design-safety         pass"
  "design-liveness       pass"
  "design-halt-liveness  Temporal properties were violated"   # accepted risk, documented
  "brokenNoExpiry-cap    CapHolds is violated"                # the design before the fix
  "brokenD2-one-order    OneOrderPerSide is violated"
  "brokenD2-safety       NoUntrackedOrder is violated"
  "brokenSettle-safety   CapHolds is violated"
  "brokenProbe-liveness  Temporal properties were violated"
  "brokenHalt-liveness   Temporal properties were violated"
)
```

Three things this buys:

- **Each property is shown to have teeth.** If `brokenSettle-safety` passed, either `CapHolds` is too weak or the model never reaches Settling. Either way you'd never know from the design runs alone.
- **Each design decision is justified by a counterexample.** "Why do we wait in Settling?" has a precise answer: without it, `CapHolds` breaks, and here is the trace.
- **Accepted risks are recorded as expected failures.** One liveness violation is a known, accepted gap: while an asset is halted, every cancel can expire before it lands. Instead of disappearing from the suite, it stays as an expected result, so it is visible, and the suite fails if the gap ever closes or changes shape.

> [!TIP]
> **Mutation-test your spec.** For every design decision, keep a variant that removes it and assert the property that decision protects now fails. A suite of expected failures is what makes the passing runs mean something.

## The race it found

The first run checked the design as originally written, and `CapHolds` failed after only 1,101 distinct states:

1. The slot places order 1. The request is slow.
2. The slot times out and probes order 1.
3. The probe reaches the exchange first. The exchange answers that it doesn't know the order.
4. The slot concludes order 1 was never placed, goes Empty, and places order 2.
5. Order 1 lands, then order 2. Both rest on the book.
6. Both fill. The position is 4; the cap is 3.

Every step is individually reasonable, and the bug only exists in the ordering of messages, which is why no unit test found it. The fix is a design decision, not a patch: **requests expire before they are probed**, so a timed-out request can no longer take effect by the time the slot acts on the probe's answer. The old design then became the broken variant `brokenNoExpiry-cap`, and the fixed design passed every safety property across more than six million distinct states.

Writing the model also surfaced a gap before any checking ran: the exchange's error for a failed cancel ("never placed, already canceled, or filled") had no transition in the design at all. Modeling forces every response to go somewhere.

## Lessons

- **Model the network you have, not the one you wish you had.** Delay, reordering and loss bounded at one lost answer were enough to find the race. Most such bugs need only two or three messages in flight.
- **Keep the model about the design, not the code.** This model follows the design documents, and its traces are the evidence for each decision. The code is then checked against the design separately; [model-based testing](model-based-testing.md) is one way.
- **Keep the suite runnable by anyone.** One script, exit code 0 only if every configuration does what it should. It belongs in CI next to the unit tests.
