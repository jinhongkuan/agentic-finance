# Introduction

> "Software systems have orders of magnitude more states than computers do."
>
> Fred Brooks, [*No Silver Bullet*](https://www.cs.unc.edu/techreports/86-020.pdf) (1986), §2.1

**When should you write the code yourself, when should you reach for a formal checker, and when should you hand the work to an agent?** That is the question this guide works on.

Agents now write most of the code. That has not made the job easier; it has moved it. I made the argument at length in [Why agentic coding makes the spec problem worse](https://www.bicameral-ai.com/blog/why-agentic-coding-makes-the-spec-problem-worse), and two of its conclusions motivate this collection:

1. **AI accelerated the writing part of coding, not the thinking part.** Deciding what to build is the essential complexity of software, in Fred Brooks's sense, and a spec detailed enough to remove ambiguity is itself code. Swapping the language for English only disguises where the labor lives. So the engineer's job is now mostly judgment: which structure, which trade-off, which invariant.
2. **The default agentic workflow erodes exactly the things that are hard to demo.** Large diffs get rubber-stamped, context is opaque, and a vibecoded app favors visible features over security, maintainability and performance. The leverage is in making human decisions explicit and putting them where agents and reviewers will read them.

## Where the examples come from

Agentic finance is my domain of interest, and it is where agentic engineering calls for the most judgment and carries the highest stakes: the code moves real money, so an invariant that silently breaks has a cost measured in dollars and regulators, while the agents writing that code are prone to confident fabrication. So most examples here come from finance. What is documented applies to agentic software engineering in general; finance is simply where a practice gets pushed to its limit.

## How this guide is organized

| Part | Kind | Shape |
|---|---|---|
| [Correctness](correctness/) | Study guides | Executable specifications with Quint: concepts, practice, and case studies built on real models |
| [Maintainability](maintainability/) | Practices reference, one per framework | A fixed set of eleven sections, so entries can be compared side by side |
| [Performance](performance/) | Study guide, one per system | Sequenced to teach, from common cases to rare ones, with a rule of thumb wherever one exists |

## The ethos

Every practice here earns its place by serving at least one of these:

- **Semantically rich.** Names, types and structure carry meaning. A symbol tells you what it is and where it lives, failures are typed values rather than strings, and domain concepts have their own types instead of hiding in primitives.
- **Agent-native.** A coding agent, or a person new to the code, can find things, change them and check the change using only the repository: predictable file locations, names you can grep for, seams that tests can swap, and conventions written down where agents read them.
- **Maintainable.** A change touches few files, logic can be tested without I/O, and choices are boring and documented.

When two practices conflict, the entry says which one wins and why.

These are my own opinions, collected from building with each tool, reading how well-regarded projects use it, and the tool's own guidance. They are not official recommendations from any project's authors. Each entry cites its sources, so you can check the reasoning and disagree with it.

## Reading locally

The site is built with [mdBook](https://rust-lang.github.io/mdBook/):

```sh
brew install mdbook      # or cargo install mdbook
mdbook serve --open      # live-reloading preview at http://localhost:3000
```

## License

[CC BY 4.0](https://github.com/jinhongkuan/agentic-finance/blob/main/LICENSE). You may share and adapt this material for any purpose, as long as you credit Jin Hong Kuan and link back to the repository.
