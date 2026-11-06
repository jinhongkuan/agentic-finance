# Agentic Finance

> "Software systems have orders of magnitude more states than computers do."
>
> Fred Brooks, [*No Silver Bullet: Essence and Accidents of Software Engineering*](https://www.cs.unc.edu/techreports/86-020.pdf) (1986)

**When should you write the code yourself, when should you reach for a formal checker, and when should you hand the work to an agent?** That is the question this guide works on.

Brooks called this complexity *essential*: it comes from deciding what the software must do in every one of those states, not from typing it in. As I argued in [Why agentic coding makes the spec problem worse](https://www.bicameral-ai.com/blog/why-agentic-coding-makes-the-spec-problem-worse), coding agents shrink the accidental part and leave the essential part as large as ever.

Study guides and practices for agentic software engineering, in three parts: correctness, maintainability and performance. Agentic finance is my domain of interest and where agentic engineering calls for the most judgment, so most examples come from it, but what is documented here applies to agentic software engineering in general.

**Read it as a site:** https://jinhongkuan.github.io/agentic-finance/

| Part | What's in it |
|---|---|
| [Introduction](src/introduction.md) | The motivation, where the examples come from, and the ethos behind every entry |
| [Correctness](src/correctness/README.md) | Study guides on executable specifications with Quint: how they differ from code, a practical workflow (including agent-assisted modeling with quint-llm-kit), and three case studies built on real models |
| [Maintainability](src/maintainability/README.md) | Practices by framework, one entry per framework in a fixed eleven-section layout. Currently: [Effect](src/maintainability/effect.md) |
| [Performance](src/performance/README.md) | Study guides, one per system, sequenced from common cases to rare ones. Currently: [Postgres access patterns](src/performance/postgres.md) |

## Running locally

The site is built with [mdBook](https://rust-lang.github.io/mdBook/).

```sh
brew install mdbook      # or: cargo install mdbook
mdbook serve --open      # live preview at http://localhost:3000
mdbook build             # static site in book/
```

Pushing to `main` builds and deploys the site to GitHub Pages through `.github/workflows/deploy.yml`.

## Layout

```
book.toml                 mdBook config
src/
  SUMMARY.md              sidebar / table of contents
  introduction.md
  correctness/            study guides: Quint concepts, practice, case studies
  maintainability/        practices by framework
  performance/            study guides
theme/custom.css          figures, callouts, dimension cards
```

## License

[CC BY 4.0](LICENSE). You may share and adapt this material for any purpose, as long as you credit Jin Hong Kuan and link back to this repository.
