# Practices by framework

One entry per framework, each using the same sections in the same order, so entries can be compared side by side. A section that doesn't apply to a framework says so rather than being left out.

## Entries

| Framework | Entry | Version covered | Last reviewed |
|---|---|---|---|
| [Effect](https://effect.website) (TypeScript) | [Effect](effect.md) | `effect` 4.1 | 2026-10-31 |

## Layout of an entry

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

1. Create `src/maintainability/<framework>.md` with the sections above.
2. State the version it covers at the top, and cite a source for every practice that isn't self-evident.
3. Add a row to **Entries** and to **Section coverage**, and a line under Maintainability in `src/SUMMARY.md`.
