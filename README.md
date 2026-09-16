# para/tool

The function signature is the tool spec. Annotate an ordinary Noeta function with `#[Tool]` and this package finds it by whole-program reflection, derives its JSON Schema from the declared parameter types, coerces a caller's arguments to those types, and calls it by name. There is no tool-definition DSL and no codegen, because the language already answers every question one would ask.

```noeta
use para.tool.{Arg, Local, Tool, Toolbox}

#[Tool(about: "Current weather for a city")]
fn weather(
    #[Arg(help: "City name, e.g. 'Malmö'")] city: string,
    #[Arg(help: "`metric` for °C or `imperial` for °F")] units: string = "metric",
): Result<string, string> {
    return Ok("18°C and clear in ${city}")
}

box = Toolbox.of(Local.new())?
box.call("weather", "{\"city\": \"Lund\"}")   // Ok("18°C and clear in Lund")
```

## Why it is its own package

`#[Tool]` is an **agent** concept: a function a language model may call. Serving those functions over MCP is one thing you can do with them, and offering them to a model in a run loop is another, so two different packages want the same attribute.

Noeta reflection is keyed by attribute **type**. Two separately-declared `Tool` structs are mutually invisible to each other's `attributes_of::<Tool>()`, and nothing reports it — the second package's tools simply never appear. So the declaration sits below both consumers, together with everything that is purely about describing and invoking a tool. What stays above is what is shaped like its consumer: a run loop's dispatcher, a protocol's framing, an agent's structured-output door.

It depends on nothing, and that is the constraint rather than an accident. Noeta has no optional dependencies and no feature flags, so every edge is unconditional: one dependency on a protocol or a harness here would make every consumer of this layer carry it.

## What it provides

`para.tool`:

| symbol | kind | purpose |
| --- | --- | --- |
| `Tool` | attribute | marks a function reachable by a model. `about` is the description it reads; `name` overrides the model-facing name. Carries `@role(Semantic.TrustBoundary)` |
| `Arg` | attribute | per-parameter and per-field `help`, which becomes the `description` in the derived schema |
| `ToolSpec` | struct | one tool as a protocol-neutral description: `name`, `description`, and `parameters` as raw JSON Schema text |
| `ToolSource` | trait | where tools come from. `source_name`, `tool_type`, `specs`, `call` |
| `Local` | struct | reflection over `#[Tool]`, with selection: `new`, `under`, `only`, `except` |
| `Toolbox` | struct | every tool across every source, collision-checked at construction |
| `ToolError` | enum | `Tool` (a model can retry), `Source` (the machinery is down), `Selection` (the program is misconfigured) |
| `schema_of(target)` | fn | the JSON Schema for one **callable**, from its parameters |
| `schema_for(target)` | fn | the JSON Schema for one declared **type**, from its fields — the same walk |
| `json_schema(t)` | fn | one `Type` as a schema fragment, or the reason it has none |
| `coerce(t, v)` | fn | a decoded JSON value as a declared type |
| `bind(tool, params, supplied)` | fn | named arguments as the positional list `invoke` takes |
| `last_segment(target)` | fn | the model-facing name for a qualified reflection identity |
| `under_namespace(target, ns)` | fn | whether a qualified target lies under a namespace, at a segment boundary |

`para.tool.boundary`:

| symbol | kind | purpose |
| --- | --- | --- |
| `crossings()` | fn | every `#[Tool]` in the program, as a qualified target — the review surface |
| `declared_outside(roots)` | fn | every `#[Tool]` declared outside the namespaces this program owns |

## Installation

```sh
noeta add para/tool
```

That asks the registry for the current release and writes the caret requirement for it, so no version is pinned here to go stale. It adds:

```toml
[dependencies]
para = [{ version = "^X.Y", package = "para/tool" }]
```

The package is keyed `para`, so its modules address as `para.tool` and `para.tool.boundary`. The array form is what makes `para.*` an import root; the single-table form resolves the package but leaves `use para.tool` unresolvable. It is pure Noeta with no dependencies, so no `[trust]` entry is needed and nothing composes a toolchain.

## Selection: one program, two consumers

`Local.new()` answers every `#[Tool]` in the program. That is the right default with one consumer and wrong with two — an agent and an MCP server in one process would each register the identical full set, with no way to say which functions go to the model and which go over the wire.

The module path is the selector. A reflection target is already fully qualified (`app.ops.wipe_staging`), and the model-facing name is already its last segment, so a namespace is structure the compiler tracks and keeps correct across a rename or a move.

```noeta
for_the_model    = Local.new().under("app.assistant")
for_the_operator = Local.new().under("app.ops").except(["wipe_staging"])
```

Three rules make it safe to rely on:

- **Selection is routing, not display.** A tool outside a selection is not callable through it, so a client that guessed a name cannot reach a function the program did not offer it.
- **It is fixed at construction.** The selectors are immutable lists and never a predicate, so `specs()` is a pure function of the constructed value. That is what the MCP specification requires of a server's `tools/list`, which must not vary per connection.
- **A selector that matches nothing is an error**, naming it and listing what does exist. A selection that quietly matched nothing would surface as a model that never calls anything.

`examples/routing` is this in full, with an assistant and an operator console over one program.

## Tools are a trust boundary

`Tool` carries `@role(Semantic.TrustBoundary)`, which turns "which functions in this program can a language model reach?" into a query — `roles_of::<Semantic>()` in-language, and the architectural graph `noeta mcp` serves to an editor or an agent.

The corollary matters for anyone writing a library. `attributes_of::<Tool>()` is whole-program and closed-world, and that reach is what makes `#[Tool]` work across a package boundary: the query runs in this package and finds tools declared in your application. It reaches the other way too. A `#[Tool]` declared at module level inside a *library* is discovered from every application depending on it, and `Local` offers it to that application's model with nothing in the application mentioning it.

So a library keeps its own `#[Tool]` fixtures inside `@test` blocks, which are stripped before lowering on any build that is not that module's own `noeta test`. An application checks the property from its own side, which is the only side it is visible from:

```noeta
@test {
    fn no_dependency_offers_tools_to_this_program(): void {
        leaked = boundary.declared_outside(["app"])
        assert(leaked == [], "${leaked}")
    }
}
```

A returned target's first segment names the package to file against. This is a different door from `Local`'s selection on purpose: narrowing a selection until a leaked tool falls outside it hides the leak rather than closing it, and the next tool the dependency adds is hidden too.

## What a parameter may be

The `Type` → JSON Schema walk is total. Every declared type either derives a schema or is refused with a message naming the parameter, the type, and what to write instead — nothing emits `{}` for a shape it did not understand, because a schema that silently says "any value" is how a model learns to send arguments the function cannot accept.

| declared | schema |
| --- | --- |
| `int` / `float` / `bool` / `string` | `{"type": "integer" \| "number" \| "boolean" \| "string"}` |
| `dyn` | `{}` — declared to accept anything |
| `?T` | `T`'s schema, absent from `required` |
| `List<T>` / `Set<T>` | `{"type": "array", "items": …}`, with `uniqueItems` for a set |
| `Map<string, V>` | `{"type": "object", "additionalProperties": …}` |
| `A \| B` | `{"anyOf": […]}` |
| a declared struct or class | a closed nested object schema, recursively |
| a declared enum | `{"enum": […]}` — the *backings* of a backed enum, the case names otherwise |

Refused, each naming what to write instead: `bytes`, `Result`, a function type, a `dyn Trait`, `f32`/`f64`, a fixed-width integer, a `Map` not keyed by `string`, a recursive type, a generic instantiation, and a payload-carrying enum. An **enum-typed parameter** is refused separately, at the dispatch layer rather than the schema layer: its schema derives perfectly, but nothing at run time can build an enum value from JSON, so the message says it is a missing constructor and not a missing description.

A caller's arguments are coerced to the declared types before anything is invoked, and that layer *is* the type check — `invoke` validates a callable's name and arity and nothing else, so an `int` parameter handed a string would reach the function body and abort there. Coercion is lenient in one direction only: a scalar sent in the neighboring JSON type (`"5"` for an `int`, `2.0` for an `int`, `5` for a `string`) is accepted, because models do that constantly and refusing costs a round trip for nothing. A lossy conversion is refused.

## A failing tool is a turn, not an outage

An unknown name, a missing argument, an unknown argument, an argument that will not coerce, and a tool that returns `Err` all come back as a `ToolError` carrying a message the caller can hand to the model, which then gets to try something else.

```
tool `teleport` failed: no such tool — this toolbox has weather, distance_km
tool `weather` failed: missing required argument `city`
tool `weather` failed: unknown argument `mood` — this tool takes city, units
tool `add` failed: argument `a`: expected an integer, got `two`
tool `distance_km` failed: arguments: invalid JSON: key must be a string at line 1 column 2
tool `distance_km` failed: no road distance on file for Malmö to Kiruna
```

A panic *inside* a tool body is still a panic, and that is correct: `invoke` catches by-name resolution, not the callee.

## Examples

```sh
cd examples/tools   && noeta run main.noe && noeta test main.noe
cd examples/routing && noeta run main.noe && noeta test main.noe
```

- **`examples/tools`** — the headline: declare a `#[Tool]`, read the schema its signature derived, call it, and watch each failure come back as a message.
- **`examples/routing`** — two consumers over one program, with the namespace as the selector, plus the collision and boundary guards.

## License

MIT or Apache-2.0, at your option.
