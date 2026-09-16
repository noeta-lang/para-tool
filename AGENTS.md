# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/tool** Noeta package: the `#[Tool]` attribute and everything that is purely about *describing and invoking* a tool. Toolchain issues (the language, the `noeta` binary, `std.*`) belong in the monorepo at github.com/noeta-lang/noeta, not here. Anything shaped like a *consumer* of tools — a run loop, a protocol server, structured output — belongs in the package that consumes them, not here.

**Every `.noe` file opens with a header comment carrying the reasoning for its own module.** Read that before its code, and keep it true.

## What this package is, and the line it must not cross

`#[Tool]` is an **agent** concept: a function a language model may call. Serving those functions over MCP is one thing you can do with them, and offering them to a model in a run loop is another, so two different packages want the same attribute. Noeta reflection is keyed by attribute **type**, so two separately-declared `Tool` structs are mutually invisible to each other's `attributes_of::<Tool>()`, with no diagnostic at all — the second package's tools would simply never appear. One declaration therefore has to sit below both consumers, and that is what this package is.

> [!IMPORTANT]
> **This package has no dependencies, and that is a design constraint rather than a fact about its current size.** Noeta has no optional dependencies and no feature flags, so every edge is unconditional: one dependency on a protocol or an agent harness would make every consumer of this layer carry it, which is exactly the coupling the package exists to prevent. Adding a `[dependencies]` entry here is a design change, not a convenience — raise it before writing it.
>
> A visible symptom to watch for: this package has **no root `noeta.lock`**, because there is nothing to lock. If one appears, a dependency was added. (The committed-lockfile rule still applies the day one legitimately exists: the generated file says in its own header that it is meant to be committed. `examples/*/noeta.lock` is not committed, and `.gitignore` says why.)

## Conventions

- Implement in full — no stubs or TODOs; new functionality lands with tests.

> [!IMPORTANT]
> **Never declare a `#[Tool]` at module level in this package.** Reflection is whole-program, so a module-level `#[Tool]` here is discovered from every consuming application and offered to that application's language model — a trust-boundary crossing a dependency has no business performing.
>
> Measured, not assumed. In a two-package probe where the library declared two module-level `#[Tool]` functions, the consuming application's `attributes_of::<Tool>()` answered `["probe.lib.lib_module_level_tool", "probe.lib.lib_second_module_level_tool", "probe_app.main.app_weather"]` — under `noeta run` and `noeta test` alike, and identically whether the query ran in the application or inside the library. A fixture inside a `@test` block did **not** appear.
>
> So the package's own fixtures live in `@test` blocks, carry `#[std.test.Skip]` so the runner does not call them as tests, and are asserted absent from a consumer by `examples/tools` and `examples/routing`. Do not "fix" a resulting failure by loosening the collision check in `Toolbox`, or by narrowing `Local` until the leaked tool falls outside the selection — both hide the leak rather than close it, and the next tool added leaks too.

- **Routing and leak-containment are different problems that touch the same surface, and they stay visibly separate.** `Local`'s `under`/`only`/`except` choose which of *your own* tools a given consumer gets, and speak model-facing names. `para.tool.boundary` answers "did something outside this program declare a tool in it?", and speaks fully qualified reflection targets. A value from one cannot be passed to the other, which is deliberate. Keep it that way, and keep the header comments that say so.
- **A selection is fixed at construction and is never a predicate.** The MCP specification requires that a server's `tools/list` not vary per connection, and a closure is an invitation to exactly that. `Local`'s three selectors are immutable lists of strings, so `specs()` is a pure function of the constructed value. A `fn(ToolSpec) -> bool` anywhere on this type is a regression however convenient it looks.
- **An unmatched selector is an error, not a no-op.** A selection that quietly matches nothing produces an empty `tools/list` and a model that never calls anything, three layers and a day from the typo that caused it.
- **American English** throughout — code, comments, and docs (`behavior`, not `behaviour`). Markdown never hard-wraps lines.
- **Conventional commits** for all commit titles. Commit each green slice as it completes, but **never `git push` without explicit authorization**. Never move a published `v*` tag — a release is a new tag.
- Keep `README.md` and this file up to date when layout or behavior changes.

## Build & test

Pure Noeta, no dependencies, no cargo. Nothing here composes a toolchain, and that is worth knowing as a diagnostic: **if a `noeta check` in this repo ever prints `composing the toolchain with native dependencies`, something has gone wrong with the dependency graph**, because this package has none.

- **Use a *released* `noeta`, never one built from a `lang` checkout.** A dev binary composes with `[patch]` entries pointing at the local `lang` crates, which is how a package with native dependencies ends up with two versions of an ABI crate in one graph. This package has no native dependencies so the failure mode does not apply directly, but the rule is the ecosystem's and a `lang`-built binary is not what consumers will run.
- **Run every module as its own entry.** `noeta test tool.noe` and `noeta test boundary.noe` are different runs, because a dev tier is activated for the **entry file only** — a sibling module's `@test` block is stripped when it is not the entry. `ci.yml` loops over `find . -name '*.noe'` for exactly this reason, and uses `find` rather than a glob because `*.noe` does not recurse.
- **Check from an example, not just from the package root.** Two real failures appear only from a consuming package, and both have bitten this repo already: an intra-package `use` that resolves from the module's own entry and not from a consumer, and anything this package leaks across the reflection boundary. `noeta check` + `noeta test` in `examples/tools` and `examples/routing` is the cheapest way to catch both, and a green package suite is not evidence either way.
- **Every test is hermetic.** This layer describes and invokes tools; it never opens a socket and never needs a key. A suite that needs one is a failed suite.

> [!IMPORTANT]
> **Ablate every test you write.** Commit it first, then break the thing it checks, watch it go red *for the reason it names*, and restore. `git checkout -- <file>` reverts to HEAD, so an uncommitted test vanishes with the ablation and the run that follows passes against a file no longer containing it.
>
> Two results are not verdicts. An ablation that **does not compile** proves nothing — respell it until the tree builds and the test is what fails. And an ablation that leaves everything **green** is a finding: it happened here when removing the `Type.Enum` arm of `undispatchable` changed nothing, because `params_of` reports a declared enum as `Type.Named` and no declaration reaches that arm at all. The fix was a test pinning which arm carries the refusal, not a deleted arm.

## Things the language makes you write a particular way

Collected because each one cost a debugging round in this repo.

- **`namespace` is a reserved word**, so no parameter or binding may be named that (E0046, which helpfully suggests `namespace_`). `Local.under` takes `ns`.
- **`@derive(Display)` and an `impl Display` block are E0027**, "implemented more than once". An enum with a hand-written `to_string` derives `Equatable` only.
- **A block-bodied `match` arm produces no value** in value position (E0055: "block arms are for side effects"). Either extract a helper, or write the `match` in statement position with a `mut` binding above it.
- **A whole-module import binds its last segment as a local name.** `use para.tool` binds `tool`; reaching a submodule through it does not work, so `para.tool.boundary` needs its own `use para.tool.boundary`, which binds `boundary`.
- **A consuming manifest needs the scope (array) form to make `para.*` an import root.** `para = { path = "…", package = "para/tool" }` resolves the package but leaves `use para.tool` as `E0019: no module para in this project`; `para = [{ path = "…", package = "para/tool" }]` is what works. Both example manifests use the array form even though each names one package.
- **Reflection keys are qualified by the derived module path — except a dev tier's.** A module-level function in `tool.noe` reports as `tool.refuses_bytes` from this package's own suite and as `para.tool.refuses_bytes` from a consumer, because a module's path is the *importing program's* prefix plus the file's path inside the package. A declaration lifted out of a `@test` block is keyed by its **bare** name instead (`shout`). That is why the refusal probes stay at module level: they are what keeps the schema walk under test against a *qualified* key, which is the shape every tool in a consumer has.
- **A dev tier is activated for the *entry file only*.** `noeta test other.noe` runs `other.noe`'s `@test` blocks and strips the blocks of every module it imports — a sibling in the same package and a dependency package alike. That is what makes the tier a real trust boundary rather than a build-mode convenience.
- **Every `fn` in a top-level `@test` block is a test root.** The runner calls each with no arguments, so a *helper* that takes one fails as a test rather than serving the tests. Helpers therefore live at module level below the block, while anything that must stay *inside* the tier carries `#[std.test.Skip]` — written inline and qualified, because a top-level `use std.test.{Skip}` would put a dev-only import in the module's shipping surface.
- **A declared enum reflects as `Type.Named`, never as `Type.Enum`.** `params_of` and `field_specs_of` report struct, class and enum kind-agnostically, which is why `nominal_schema` and `undispatchable_nominal` have to ask `variants_of` and `field_specs_of` as a **pair** — through `field_specs_of` alone an enum is indistinguishable from a field-less struct, and a walk that trusted it emits `{"type":"object","properties":{}}` for an enum and is silently wrong.
- **`json.parse` aborts on malformed input** and `json.try_parse::<T>` cannot build a `Map<string, dyn>`, so `parse_object` proves well-formedness with a field-less decodable witness and then walks the proven-good document dynamically. Never call `json.parse` on a blob that came from a model: an abort there is a crashed process where the whole design is a correction turn.
- **Reflection's `invoke` and `construct` validate far less than they look like they do.** `invoke` checks a callable's name and arity and *not* its argument types, so `invoke("f", ["seven"])` against `fn f(n: int)` returns `Ok` and aborts inside the body. It takes a positional `List<dyn>` and has no named-map form, so a *gap* (a defaulted parameter omitted while a later one is supplied) is inexpressible, and `bind` refuses it by name rather than guessing at a default reflection does not expose. `construct(name, fields)` checks a field's presence and its *scalar* type and nothing else. **The coercion layer is the type check, and nothing may reach `invoke`/`construct` around it.**
- **An enum value cannot be built from JSON at all**, so an enum-typed tool parameter is refused — at the dispatch layer, not the schema layer, and the message says which. `@derive(Deserialize<Json>)` on a struct with an enum-typed field is E0050 at check time, `construct` answers `not a constructible struct or class`, and `Enum.from` takes a *case name* and **aborts** on an unknown one. Do not "fix" it by emitting the schema and letting dispatch fail: that teaches a model to send arguments the function cannot accept, which is the one failure this package exists to prevent.

## Outstanding work

- **`ToolError.Source` is raised by no code in this package**, because the only source here is `Local` and reflection cannot fail as a transport. It exists because `ToolSource` is the seam a second consumer implements, and a dying MCP server's "exit status 3" has to survive the trip through this layer rather than being flattened into a tool's name. It is kept honest by a `BrokenSource` fixture in `tool.noe`'s suite that proves `Toolbox.with` propagates the variant unchanged. Delete neither without the other.
- **`para/ai` still carries its own copy** of everything extracted here. Its migration to depend on this package is a separate slice and needs this repo to have a remote first. Until then the duplication is expected; do not edit `para/ai` from this repo.
