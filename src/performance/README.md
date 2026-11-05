# Study guides

One guide per system. Unlike the maintainability entries, a study guide is not a checklist. It is sequenced to teach: the common cases come first and the rare ones last, each case names the standard structure and the standard technique, and a **rule of thumb** is called out wherever one exists.

## Guides

| System | Guide | Covers |
|---|---|---|
| Postgres | [Postgres access patterns](postgres.md) | The 7 dimensions of a query, the 15 recurring query families, and the index, layout and query-technique menus each picks from |

## Conventions

- **Rule of thumb** callouts (`> [!TIP]`) are the compressed version of the section above them. If you remember nothing else, remember those.
- **Watch out** callouts (`> [!WARNING]`) name the mistake that the section exists to prevent.
- Figures are inline SVG drawn to show a mechanism, not to decorate. If a sentence would say it faster, there is a sentence instead.

## Adding a guide

1. Create `src/performance/<system>.md`.
2. Order the material from most common to rarest. Put a rule of thumb after every section that has one.
3. Add a row to **Guides** and a line under Performance in `src/SUMMARY.md`.
