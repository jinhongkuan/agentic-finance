# Introduction

Personally curated engineering practices, written for code that is **semantically rich, agent-native and maintainable**.

These are my own opinions, collected from building with each tool, reading how well-regarded projects use it, and the tool's own guidance. They are not official recommendations from any project's authors. Each entry cites the sources it draws on, so you can check the reasoning and disagree with it.

## The ethos

Every practice here earns its place by serving at least one of these:

- **Semantically rich.** Names, types and structure carry meaning. A symbol tells you what it is and where it lives, failures are typed values rather than strings, and domain concepts have their own types instead of hiding in primitives.
- **Agent-native.** A coding agent, or a person new to the code, can find things, change them and check the change using only the repository: predictable file locations, names you can grep for, seams that tests can swap, and conventions written down where agents read them.
- **Maintainable.** A change touches few files, logic can be tested without I/O, and choices are boring and documented.

When two practices conflict, the entry says which one wins and why.

## Two kinds of entry

| Section | Kind | Shape |
|---|---|---|
| [Maintainability](maintainability/README.md) | Practices reference, one per framework | A fixed set of eleven sections, so entries can be compared side by side |
| [Performance](performance/README.md) | Study guide, one per system | Sequenced to teach, from common cases to rare ones, with a rule of thumb wherever one exists |

## Reading locally

The site is built with [mdBook](https://rust-lang.github.io/mdBook/):

```sh
brew install mdbook      # or cargo install mdbook
mdbook serve --open      # live-reloading preview at http://localhost:3000
```

## License

[CC BY 4.0](https://github.com/jinhongkuan/best-practices/blob/main/LICENSE). You may share and adapt this material for any purpose, as long as you credit Jin Hong Kuan and link back to the repository.
