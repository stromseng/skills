# CLI Patterns

`@effect/cli` provides typed argument parsing, automatic help generation, and seamless integration with Effect services.

## Installation

```bash
bun add @effect/cli @effect/platform @effect/platform-bun
```

For Node.js, use `@effect/platform-node` instead.

## Minimal Example

A greeting CLI with a name argument and `--shout` flag:

```typescript
import { Args, Command, Options } from "@effect/cli"
import { BunContext, BunRuntime } from "@effect/platform-bun"
import { Console, Effect } from "effect"

const name = Args.text({ name: "name" }).pipe(Args.withDefault("World"))
const shout = Options.boolean("shout").pipe(Options.withAlias("s"))

const greet = Command.make("greet", { name, shout }, ({ name, shout }) => {
  const message = `Hello, ${name}!`
  return Console.log(shout ? message.toUpperCase() : message)
})

const cli = Command.run(greet, {
  name: "greet",
  version: "1.0.0"
})

cli(process.argv).pipe(
  Effect.provide(BunContext.layer),
  BunRuntime.runMain
)
```

Run it:

```bash
bun run greet.ts              # Hello, World!
bun run greet.ts Alice        # Hello, Alice!
bun run greet.ts --shout Bob  # HELLO, BOB!
bun run greet.ts --help       # Shows usage
```

Built-in flags (`--help`, `--version`) work automatically.

## Arguments and Options

**Arguments** are positional. **Options** are named flags. Options must come before arguments: `cmd --flag arg` works, `cmd arg --flag` doesn't.

### Argument Patterns

```typescript
// Required text
Args.text({ name: "file" })

// Optional argument
Args.text({ name: "output" }).pipe(Args.optional)

// With default
Args.text({ name: "format" }).pipe(Args.withDefault("json"))

// Repeated (zero or more)
Args.text({ name: "files" }).pipe(Args.repeated)

// At least one
Args.text({ name: "files" }).pipe(Args.atLeast(1))
```

### Option Patterns

```typescript
// Boolean flag
Options.boolean("verbose").pipe(Options.withAlias("v"))

// Text option
Options.text("output").pipe(Options.withAlias("o"))

// Optional text
Options.text("config").pipe(Options.optional)

// Choice from fixed values
Options.choice("format", ["json", "yaml", "toml"])

// Integer
Options.integer("count").pipe(Options.withDefault(10))
```

## Subcommands

Most CLIs have multiple commands. Use `Command.withSubcommands`:

```typescript
const task = Args.text({ name: "task" })

const add = Command.make("add", { task }, ({ task }) =>
  Console.log(`Adding: ${task}`)
)

const list = Command.make("list", {}, () =>
  Console.log("Listing tasks...")
)

const app = Command.make("tasks").pipe(
  Command.withSubcommands([add, list])
)
```

```bash
tasks add "Buy milk"   # Adding: Buy milk
tasks list             # Listing tasks...
tasks --help           # Shows available subcommands
```

## Descriptions

Add descriptions to commands, arguments, and options:

```typescript
const text = Args.text({ name: "task" }).pipe(
  Args.withDescription("The task description")
)

const all = Options.boolean("all").pipe(
  Options.withAlias("a"),
  Options.withDescription("Show all tasks including completed")
)

const addCommand = Command.make("add", { text }, ({ text }) =>
  Console.log(`Adding: ${text}`)
).pipe(Command.withDescription("Add a new task"))
```

## Service Integration

CLI commands can use Effect services just like any other Effect code:

```typescript
import { Args, Command } from "@effect/cli"
import { Context, Effect, Layer, Schema } from "effect"

const TaskId = Schema.Number.pipe(Schema.brand("TaskId"))
type TaskId = typeof TaskId.Type

class Task extends Schema.Class<Task>("Task")({
  id: TaskId,
  text: Schema.NonEmptyString,
  done: Schema.Boolean
}) {}

// Service definition
class TaskRepo extends Context.Tag("TaskRepo")<
  TaskRepo,
  {
    readonly list: (all?: boolean) => Effect.Effect<ReadonlyArray<Task>>
    readonly add: (text: string) => Effect.Effect<Task>
    readonly toggle: (id: TaskId) => Effect.Effect<Task | undefined>
    readonly clear: () => Effect.Effect<void>
  }
>() {}

// Commands using the service
const listCommand = Command.make("list", { all: Options.boolean("all") }, ({ all }) =>
  Effect.gen(function* () {
    const repo = yield* TaskRepo
    const tasks = yield* repo.list(all)

    if (tasks.length === 0) {
      yield* Console.log("No tasks.")
      return
    }

    for (const task of tasks) {
      const status = task.done ? "[x]" : "[ ]"
      yield* Console.log(`${status} #${task.id} ${task.text}`)
    }
  })
).pipe(Command.withDescription("List pending tasks"))

const addCommand = Command.make("add", { text: Args.text({ name: "task" }) }, ({ text }) =>
  Effect.gen(function* () {
    const repo = yield* TaskRepo
    const task = yield* repo.add(text)
    yield* Console.log(`Added task #${task.id}: ${task.text}`)
  })
).pipe(Command.withDescription("Add a new task"))

const app = Command.make("tasks", {}).pipe(
  Command.withDescription("A simple task manager"),
  Command.withSubcommands([addCommand, listCommand])
)
```

## Schema Validation

Use `Args.withSchema` to validate arguments with Schema:

```typescript
const TaskId = Schema.Number.pipe(Schema.brand("TaskId"))
type TaskId = typeof TaskId.Type

const id = Args.integer({ name: "id" }).pipe(
  Args.withSchema(TaskId),
  Args.withDescription("The task ID to toggle")
)

const toggleCommand = Command.make("toggle", { id }, ({ id }) =>
  Effect.gen(function* () {
    const repo = yield* TaskRepo
    const result = yield* repo.toggle(id)

    if (!result) {
      yield* Console.log(`Task #${id} not found`)
      return
    }

    yield* Console.log(`Toggled: ${result.text} (${result.done ? "done" : "pending"})`)
  })
).pipe(Command.withDescription("Toggle a task's done status"))
```

## Wiring It Together

```typescript
import { BunContext, BunRuntime } from "@effect/platform-bun"
import { Effect, Layer } from "effect"

const cli = Command.run(app, {
  name: "tasks",
  version: "1.0.0"
})

const MainLayer = Layer.provideMerge(TaskRepo.layer, BunContext.layer)

cli(process.argv).pipe(Effect.provide(MainLayer), BunRuntime.runMain)
```

## Summary

| Concept | API |
|---------|-----|
| Define command | `Command.make(name, config, handler)` |
| Positional args | `Args.text`, `Args.integer`, `Args.optional`, `Args.repeated` |
| Named options | `Options.boolean`, `Options.text`, `Options.choice` |
| Option alias | `Options.withAlias("v")` |
| Descriptions | `Args.withDescription`, `Options.withDescription`, `Command.withDescription` |
| Schema validation | `Args.withSchema(MySchema)` |
| Subcommands | `Command.withSubcommands([...])` |
| Run CLI | `Command.run(cmd, { name, version })` |
| Platform layer | `BunContext.layer` or `NodeContext.layer` |

For the full API, see the [@effect/cli documentation](https://effect-ts.github.io/effect/docs/cli).
