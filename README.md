# Best practices

Personally curated engineering practices and study guides, written for code that is **semantically rich, agent-native and maintainable**.

**Read it as a site:** https://jinhongkuan.github.io/best-practices/

| Section | What's in it |
|---|---|
| [Maintainability](src/maintainability/README.md) | Practices by framework, one entry per framework in a fixed eleven-section layout. Currently: [Effect](src/maintainability/effect.md). |
| [Performance](src/performance/README.md) | Study guides, one per system, sequenced from common cases to rare ones. Currently: [Postgres access patterns](src/performance/postgres.md). |

The introduction, ethos and the layout of each kind of entry are in [`src/introduction.md`](src/introduction.md).

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
  maintainability/        practices reference entries
  performance/            study guides
theme/custom.css          figures, callouts, dimension cards
```

## License

[CC BY 4.0](LICENSE). You may share and adapt this material for any purpose, as long as you credit Jin Hong Kuan and link back to this repository.
