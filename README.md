# Best practices

Personally curated best practices, one file per framework, written for code that is **semantically rich, agent-native and maintainable**.

These are my own opinions, collected from building with each framework, reading how well-regarded projects use it, and the framework's own guidance. They are not official recommendations from any framework's authors. Each entry cites the sources it draws on, so you can check the reasoning and disagree with it.

## The ethos

Every practice here earns its place by serving at least one of these:

- **Semantically rich.** Names, types and structure carry meaning. A symbol tells you what it is and where it lives, failures are typed values rather than strings, and domain concepts have their own types instead of hiding in primitives.
- **Agent-native.** A coding agent, or a person new to the code, can find things, change them and check the change using only the repository: predictable file locations, names you can grep for, seams that tests can swap, and conventions written down where agents read them.
- **Maintainable.** A change touches few files, logic can be tested without I/O, and choices are boring and documented.

When two practices conflict, the entry says which one wins and why.

## Entries

| Framework | Entry | Version covered | Last reviewed |
|---|---|---|---|
| [Effect](https://effect.website) (TypeScript) | [Effect.md](Effect.md) | `effect` 4.1 | 2026-10-31 |

## Layout of an entry

Every entry uses the same sections, in this order, so entries can be compared side by side. A section that doesn't apply to a framework says so rather than being left out.

| # | Section | What it covers |
|---|---|---|
| 1 | **Project directory** | Folder layout, where each kind of file goes, when to split into packages |
| 2 | **Modules and naming** | File naming, what one module holds, how modules are exported and imported |
| 3 | **Dependencies and I/O** | How code reaches the outside world (disk, network, processes) and how that is swapped in tests |
| 4 | **Errors** | How failures are modelled, reported and turned into exit codes or responses |
| 5 | **Writing code** | Idioms for everyday code in the framework |
| 6 | **Testing** | Test layout, tools, and what each kind of test is for |
| 7 | **Entry points and runtime** | How the program starts, shuts down and wires its parts together |
| 8 | **Build and publish** | Bundling, packaging and releasing |
| 9 | **Agent setup** | What to put in the repository so coding agents follow these practices |
| 10 | **Migration notes** | Changes between major versions that affect the practices above |
| 11 | **Sources** | Where the practices come from |

### Section coverage

| Entry | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Effect | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

## Adding an entry

1. Create `<Framework>.md` with the sections above.
2. State the version it covers at the top, and cite a source for every practice that isn't self-evident.
3. Add a row to **Entries** and to **Section coverage**.
