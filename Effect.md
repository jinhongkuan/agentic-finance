# Effect

Practices for TypeScript projects built on [Effect](https://effect.website).

**Version covered:** `effect` 4.1 (with `@effect/platform-node` 4.1 and `@effect/vitest` 4.1). Section 10 lists what changed from v3.

**The rule underneath everything:** plain functions for logic, Effect for effects. If a piece of code can't fail, wait, or need a dependency, it isn't an Effect. Most of what follows is this rule applied to folders, names and tests.

---

## 1. Project directory

A CLI or service in one package:

```
src/
  bin.ts                  entry point: provides platform layers, calls NodeRuntime.runMain
  Cli.ts                  the command tree (effect/cli); one Effect program per command
  Domain.ts               plain data types shared across modules
  Config.ts               the config Schema and its decoder
  Repo.ts                 a service: every file read/write and git call (layer, layerMemory)
  billing/                a feature area: lowercase folder, PascalCase modules
    Invoice.ts            pure logic
    InvoiceError.ts       the area's tagged errors
    PaymentGateway.ts     the area's service, with its layers as statics
  workflows/
    Commands.ts           programs that compose pure modules with services
  index.ts                export * as X from "./X.js", one line per module
test/
  billing/Invoice.test.ts mirrors src/
  Commands.test.ts
```

- **Group by feature once there is more than one domain area.** Use lowercase folders (`billing/`, `authority/`) holding PascalCase modules. A single-domain app can instead group by role: `domain/`, `services/`, `workflows/`.
  *Why:* a change to one concept stays inside one folder, and an agent finds a concept by its name rather than by guessing which layer it was filed under.
- **Keep pure modules free of Effect services.** Parsing, evaluation and normalization take and return plain data. Only `workflows/` and service modules yield services.
  *Why:* pure modules can be property-tested without layers, reused outside Effect (an ESLint rule, a browser bundle), and read without knowing the runtime.
- **Keep the entry point thin.** `bin.ts` provides platform services and runs the program; `Cli.ts` (or `Server.ts`) defines what the program does.
  *Why:* this follows the `create-effect-app` CLI template, and it keeps platform choice (Node, Bun) in one line.
- **Put tests in a sibling `test/` folder that mirrors `src/`.**
  *Why:* this is the convention across Effect's own packages, the templates and most application repos.
- **Split into several packages only when you publish or deploy more than one thing.** Effect's own monorepo and the official monorepo template (`domain` / `server` / `cli`) split because each part ships separately. A CLI that ships alone should publish as one package; internal workspaces can be bundled into it (section 8).
  *Why:* every package boundary adds build ordering, versioning and publishing work. Pay that cost only for a real deployment boundary.

## 2. Modules and naming

- **One concept per module, in a PascalCase file named after its main export.** `HttpClient.ts` exports `HttpClient`; `SourceError.ts` holds the source errors.
  *Why:* this is the near-universal convention in Effect code, and the file name tells you what's inside.
- **Export modules as namespaces from `index.ts`:**
  ```ts
  export * as Authority from "./authority/Authority.js"
  export * as Workspace from "./workflows/Workspace.js"
  ```
  Call sites read `Authority.expand(...)` and `Workspace.loadWorkspace`.
  *Why:* the namespace names the file, so a reader or agent can jump from any call site to its definition, and two modules can both export an `evaluate` without clashing. Effect's own packages export this way.
- **Make namespace names unique across the project.** Prefer `AuthorityFile` and `AuthoritySyntax` over two files both called `File.ts` or `Syntax.ts`.
  *Why:* a unique name is one grep away; a repeated one needs a path to disambiguate.
- **Avoid catch-all modules** (`utils.ts`, `helpers.ts`, `types.ts`). Name each module after the concept it holds.
  *Why:* a catch-all name says nothing about its contents, so every lookup becomes a search.
- **Give service identifiers the package and path:** `"myapp/billing/PaymentGateway"`.
  *Why:* the Effect docs recommend it, and it makes services traceable in logs and spans.

## 3. Dependencies and I/O

- **Make every effect boundary a service.** Disk, child processes, network and the clock (when determinism matters) go behind `Context.Service`:
  ```ts
  export class Repo extends Context.Service<Repo, {
    readText(file: string): Effect.Effect<string | undefined>
    git(args: ReadonlyArray<string>): Effect.Effect<string, RepoError>
  }>()("myapp/Repo") {
    static readonly layer = (root: string) => Layer.effect(Repo, /* FileSystem + ChildProcessSpawner */)
    static readonly layerMemory = (files: Record<string, string>) => Layer.sync(Repo, /* a Map */)
  }
  ```
  *Why:* workflows become testable without a disk or a git repository, and each kind of I/O lives in exactly one place.
- **Keep layers on the service as statics:** `layer` for production, plus `layerTest`, `layerMemory` or `layerFixture` for tests. A layer that needs configuration is a static function, such as `Sources.layer(config)`.
  *Why:* this is the v4 documented pattern; the service and every way to build it are found in one place.
- **Build services on the platform-agnostic services in `effect`** (`FileSystem`, `Path`, `ChildProcessSpawner` from `effect/process`), and provide `NodeServices.layer` only in `bin.ts`.
  *Why:* the core stays runtime-neutral, and swapping Node for Bun is a one-line change.
- **Provide layers at the edge, not inside library code.** Functions declare what they need in their `R` type; the entry point or test decides how it's built.
- **Bridge to callback APIs with the current context.** When a library calls your code back (an MCP SDK, a web framework):
  ```ts
  const run = Effect.runPromiseWith(yield* Effect.context<Repo>())
  server.registerTool("read", schema, (input) => run(readTool(input)))
  ```
  For a long-lived app, build one `ManagedRuntime` from the application layer instead.
  *Why:* callbacks run with the same services, and the same test layers, as the rest of the program.

## 4. Errors

- **Model errors as `Schema.TaggedError` classes:**
  ```ts
  export class NotFound extends Schema.TaggedError<NotFound>()("NotFound", { ref: Schema.String, message: Schema.String }) {}
  ```
  *Why:* the v4 docs prefer it. The `_tag` enables `Effect.catchTag`, and the schema makes errors serializable across RPC and MCP boundaries.
- **Group an area's errors in one `XError.ts` module** (`SourceError.ts`, `InvoiceError.ts`), not in one global `errors.ts`.
- **Have fallible pure functions return `Result`** rather than throwing. Parsers and decoders stay pure, and callers lift them with `Result.match`.
- **Give CLI exit codes to errors, not to `process.exit`:**
  ```ts
  class Exit extends Data.TaggedError("Exit")<{ readonly code: number }> {
    get [Runtime.errorExitCode]() { return this.code }
    readonly [Runtime.errorReported] = false // already printed
  }
  ```
  `NodeRuntime.runMain` reads the code when the program fails.
  *Why:* finalizers still run, and the exit code is visible in the program's type.

## 5. Writing code

- **Use `Effect.gen` inline, and `Effect.fn("Module.name")` for reusable functions** (it adds a span and better stack traces). Use `Effect.fnUntraced` in hot paths and library internals. Don't write functions that only wrap and return an `Effect.gen`.
- **Annotate return types with `Effect.fn.Return<A, E, R>`** on exported functions, so the error and requirement channels are part of the documented interface.
- **Always `return yield*` when failing** inside a generator, so TypeScript knows control stops there.
- **Parse untrusted input with `Schema`, not hand-written checks.** Use the `Predicate` module instead of writing your own `isRecord` and similar guards.
- **Keep the pure core free of `Effect` imports altogether** where you can: plain data in, plain data out (section 1).

## 6. Testing

- **Test Effect code with `@effect/vitest`:** `it.effect` for single tests, `layer(...)` to share a layer across a block, `it.effect.prop` for Schema-driven property tests.
- **Test workflows against in-memory layers** (`Repo.layerMemory`, `Transports.layerFixture`). These should be the bulk of your workflow tests: fast, deterministic, no disk.
- **Property-test the pure core** (with fast-check, or `it.effect.prop`). For stateful logic, check named invariants after every step of a random action sequence, against an independent, deliberately simple oracle.
- **Keep a few end-to-end tests against the built binary** in a throwaway directory, to cover wiring, argument parsing and exit codes, which in-memory tests can't see.
- **Prove the tests can fail:** hand-written mutants of key functions should each break a test.

## 7. Entry points and runtime

- **Use `NodeRuntime.runMain` as the only entry point:**
  ```ts
  run.pipe(Effect.provide(NodeServices.layer), NodeRuntime.runMain)
  ```
  *Why:* it installs SIGINT and SIGTERM handlers, interrupts fibers cleanly, and maps failures to exit codes (130 for interrupts).
- **Build CLIs with `effect/cli`.** Put flags every command needs on the root command with `Command.withSharedFlags`; they're accepted before or after the subcommand. Validate arguments with `Argument.withSchema` and `Flag.withSchema`.
- **Make long-running commands complete when their work does.** A server should resolve when its transport closes, using `Effect.callback`, rather than calling `Effect.never`. A process with nothing left to do should exit.

## 8. Build and publish

- **Publish a CLI as one package and bundle it** with esbuild or tsup. Keep the package's own dependencies external, and bundle private workspace packages in.
- **Mark internal workspace packages `private`.** Publish with `pnpm publish` so `workspace:*` versions are rewritten.
- **Give subpath exports to secondary entry points** (`"./eslint"`) rather than publishing a separate package for each.

## 9. Agent setup

- **Point agents at the docs that ship with the installed version.** Effect 4 includes `node_modules/effect/AGENTS.md` and `node_modules/effect/ai-docs/`, runnable examples that match the installed version. Tell agents to prefer them over web search, which mixes v3 and v4.
- **Write the conventions in the repository:** a short Conventions section in `CONTRIBUTING.md` or `AGENTS.md` covering module naming, how services and layers are declared, error style, and where in-memory layers live, with a link to one exemplary test.
- **Keep one canonical example of each pattern** (one service with production and test layers, one workflow test), and point to it from the conventions.
  *Why:* agents copy what they find nearby. A single clean example spreads; several inconsistent ones keep the inconsistency alive.

## 10. Migration notes (v3 → v4)

Verified while migrating a v3 codebase to `effect` 4.1.

| v3 | v4 |
|---|---|
| `Context.Tag("id")<Self, Shape>()` | `Context.Service<Self, Shape>()("id")`, with layers as statics |
| `Either`, `Effect.either` | `Result`, `Effect.result`; `.right` / `.left` become `.success` / `.failure` |
| `Either.right` / `left` / `isRight` | `Result.succeed` / `fail` / `isSuccess` |
| `Data.TaggedError` | still exists; `Schema.TaggedError` is preferred |
| `Effect.async` | `Effect.callback` |
| `Effect.catchAll` | `Effect.catch` |
| `Effect.timeoutFail({ onTimeout })` | `Effect.timeoutOrElse({ orElse: () => Effect.fail(...) })` |
| `Schedule.intersect(a, b)` | `Schedule.max([a, b])` |
| `Schema.Literal("a", "b")` | `Schema.Literals(["a", "b"])` |
| `Schema.Union(A, B)` | `Schema.Union([A, B])` |
| `Schema.Record({ key, value })` | `Schema.Record(key, value)` |
| `Schema.optionalWith(S, { default })` | `S.pipe(Schema.withDecodingDefaultKey(Effect.succeed(value)))` |
| `Schema.decodeUnknownEither` | `Schema.decodeUnknownResult` |
| `@effect/platform` `FileSystem`, `Command` | `FileSystem` from `effect`; `ChildProcess`, `ChildProcessSpawner` from `effect/process` |
| `@effect/cli` | `effect/cli` |
| `@effect/vitest` 0.x with vitest 3 | `@effect/vitest` 4.x requires vitest 5 |

## 11. Sources

- Effect 4 agent docs, shipped in the package: `node_modules/effect/AGENTS.md` and `node_modules/effect/ai-docs/` (services, layers, errors, `effect/cli`, `effect/process`, `@effect/vitest`).
- Effect's own packages, `v3` branch: flat PascalCase modules, `export * as` barrels, `test/` beside `src/`. <https://github.com/Effect-TS/effect/tree/v3/packages>
- `create-effect-app` templates (CLI: `bin.ts` + `Cli.ts`, bundled with tsup; monorepo: `domain` / `server` / `cli`). <https://github.com/Effect-TS/examples/tree/main/templates>
- Official HTTP server example (feature module plus same-named folder, `Domain/`, test layers as statics). <https://github.com/Effect-TS/examples/tree/main/examples/http-server>
- mikearnaldi/accountability (lowercase feature folders, per-feature `*Errors.ts`, `test/` mirroring `src/`). <https://github.com/mikearnaldi/accountability>
- lucas-barake/effect-monorepo (shared `domain` package, contracts with their schemas and errors). <https://github.com/lucas-barake/effect-monorepo>
- tim-smart/effect-mcp (an MCP server built on Effect, bundled with tsup). <https://github.com/tim-smart/effect-mcp>
- Effect docs on layer naming (`Live` / `Test` suffixes in v3, superseded by static layers in v4). <https://effect.website/docs/requirements-management/layers/>
