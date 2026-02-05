# Testing Patterns

`@effect/vitest` provides enhanced testing support for Effect code. It handles Effect execution, scoped resources, layers, and provides detailed fiber failure reporting.

## Why @effect/vitest?

- **Native Effect support**: Run Effect programs directly in tests with `it.effect()`
- **Automatic cleanup**: `it.scoped()` manages resource lifecycles
- **Test services**: Use TestClock, TestRandom for deterministic tests
- **Better errors**: Full fiber dumps with causes, spans, and logs
- **Layer support**: Provide dependencies to tests with `Effect.provide()`

## Install

```bash
bun add -D vitest @effect/vitest
```

## Setup

Update your test script to use vitest (not `bun test`):

```json
// package.json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

Create a `vitest.config.ts`:

```typescript
import { defineConfig } from "vitest/config"

export default defineConfig({
  test: {
    include: ["tests/**/*.test.ts"],
  },
})
```

## Basic Testing

Import test functions and assertions from `@effect/vitest`:

```typescript
import { Effect } from "effect"
import { describe, expect, it } from "@effect/vitest"

describe("Calculator", () => {
  // Sync test - regular function
  it("creates instances", () => {
    const result = 1 + 1
    expect(result).toBe(2)
  })

  // Effect test - returns Effect
  it.effect("adds numbers", () =>
    Effect.gen(function* () {
      const result = yield* Effect.succeed(1 + 1)
      expect(result).toBe(2)
    })
  )
})
```

## Test Function Variants

### it.effect()

For tests that return Effect values (most common):

```typescript
it.effect("processes data", () =>
  Effect.gen(function* () {
    const result = yield* processData("input")
    expect(result).toBe("expected")
  })
)
```

### it.scoped()

For tests using scoped resources. The scope closes automatically when the test ends, triggering cleanup finalizers:

```typescript
import { FileSystem } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect } from "effect"

it.scoped("temp directory is cleaned up", () =>
  Effect.gen(function* () {
    const fs = yield* FileSystem.FileSystem

    // makeTempDirectoryScoped creates a directory that's deleted when scope closes
    const tempDir = yield* fs.makeTempDirectoryScoped()

    // Use the temp directory
    yield* fs.writeFileString(`${tempDir}/test.txt`, "hello")
    const exists = yield* fs.exists(`${tempDir}/test.txt`)
    expect(exists).toBe(true)

    // When test ends, scope closes and tempDir is deleted
  }).pipe(Effect.provide(NodeFileSystem.layer))
)
```

### it.live()

For tests using real time (no TestClock). Use when you need actual delays or real clock behavior:

```typescript
import { Clock, Effect } from "effect"

// it.effect provides TestContext - clock starts at 0
it.effect("test clock starts at zero", () =>
  Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    expect(now).toBe(0)
  })
)

// it.live uses real system clock
it.live("real clock", () =>
  Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    expect(now).toBeGreaterThan(0) // Actual system time
  })
)
```

### it.scopedLive()

For tests that need both scoped resources and real time (live environment):

```typescript
import { Effect } from "effect"

it.scopedLive("live test with resources", () =>
  Effect.gen(function* () {
    const resource = yield* Effect.acquireRelease(
      acquireRealResource,
      () => releaseRealResource
    )
    // Uses real clock + automatic resource cleanup
  })
)
```

### Using TestClock

`it.effect` automatically provides TestContext with TestClock. Use `TestClock.adjust` to simulate time:

```typescript
import { Effect, Fiber, TestClock } from "effect"

it.effect("time-based test", () =>
  Effect.gen(function* () {
    const fiber = yield* Effect.delay(Effect.succeed("done"), "10 seconds").pipe(
      Effect.fork
    )
    yield* TestClock.adjust("10 seconds")
    const result = yield* Fiber.join(fiber)
    expect(result).toBe("done")
  })
)
```

## Providing Layers

Use `Effect.provide()` inline for test-specific layers:

```typescript
import { Context, Effect, Layer } from "effect"

class Database extends Context.Tag("Database")<
  Database,
  { query: (sql: string) => Effect.Effect<string[]> }
>() {}

const testDatabase = Layer.succeed(Database, {
  query: (_sql) => Effect.succeed(["mock", "data"])
})

it.effect("queries database", () =>
  Effect.gen(function* () {
    const db = yield* Database
    const results = yield* db.query("SELECT * FROM users")
    expect(results.length).toBe(2)
  }).pipe(Effect.provide(testDatabase))
)
```

## Test Modifiers

### Skipping Tests

Use `it.effect.skip` to temporarily disable a test:

```typescript
it.effect.skip("temporarily disabled", () =>
  Effect.gen(function* () {
    // This test won't run
  })
)
```

### Running a Single Test

Use `it.effect.only` to run just one test:

```typescript
it.effect.only("focus on this test", () =>
  Effect.gen(function* () {
    // Only this test runs
  })
)
```

### Expecting Tests to Fail

Use `it.effect.fails` to assert that a test should fail:

```typescript
it.effect.fails("known bug", () =>
  Effect.gen(function* () {
    // This test is expected to fail
    expect(1 + 1).toBe(3)
  })
)
```

## Effect-Specific Utilities

`@effect/vitest` provides additional assertion utilities in `utils`:

```typescript
import { it } from "@effect/vitest"
import {
  assertEquals,       // Uses Effect's Equal.equals
  assertTrue,
  assertFalse,
  assertSome,        // For Option.Some
  assertNone,        // For Option.None
  assertRight,       // For Either.Right
  assertLeft,        // For Either.Left
  assertSuccess,     // For Exit.Success
  assertFailure      // For Exit.Failure
} from "@effect/vitest/utils"
import { Effect, Option, Either } from "effect"

it.effect("with effect assertions", () =>
  Effect.gen(function* () {
    const option = yield* someOptionalEffect
    assertSome(option, expectedValue)

    const either = yield* someEitherEffect
    assertRight(either, expectedValue)
  })
)
```

## Testing Expected Failures with Effect.flip

Use `Effect.flip` to convert failures to successes for simpler assertions:

```typescript
import { it, expect } from "@effect/vitest"
import { Effect, Data } from "effect"

class UserNotFoundError extends Data.TaggedError("UserNotFoundError")<{
  userId: string
}> {}

it.effect("should fail with specific error", () =>
  Effect.gen(function* () {
    const error = yield* Effect.flip(failingOperation())
    expect(error).toBeInstanceOf(UserNotFoundError)
    expect(error.userId).toBe("123")
  })
)
```

## Logging

By default, `it.effect` suppresses log output. To enable logging:

```typescript
import { Logger } from "effect"

// Option 1: Provide a logger
it.effect("with logging", () =>
  Effect.gen(function* () {
    yield* Effect.log("This will be shown")
  }).pipe(Effect.provide(Logger.pretty))
)

// Option 2: Use it.live (logging enabled by default)
it.live("live with logging", () =>
  Effect.gen(function* () {
    yield* Effect.log("This will be shown")
  })
)
```

## Stateful Mock Layers

For tests that need to track state across multiple operations:

```typescript
import { Context, Effect, Layer, Ref, Option } from "effect"

interface User {
  id: string
  name: string
  email: string
}

interface UserRepository {
  findById: (id: string) => Effect.Effect<Option.Option<User>>
  save: (user: User) => Effect.Effect<User>
}

const UserRepository = Context.GenericTag<UserRepository>("UserRepository")

// Mock with state using Ref
const UserRepositoryStateful = Layer.effect(
  UserRepository,
  Effect.gen(function* () {
    const storage = yield* Ref.make<Map<string, User>>(new Map([
      ["1", { id: "1", name: "Alice", email: "alice@example.com" }]
    ]))

    return {
      findById: (id: string) =>
        storage.get.pipe(
          Effect.map((map) => Option.fromNullable(map.get(id)))
        ),

      save: (user: User) =>
        storage.update((map) => map.set(user.id, user)).pipe(
          Effect.as(user)
        )
    }
  })
)

// Usage in tests
it.effect("should save and retrieve user", () =>
  Effect.gen(function* () {
    const repo = yield* UserRepository

    const newUser = { id: "2", name: "Bob", email: "bob@example.com" }
    yield* repo.save(newUser)

    const result = yield* repo.findById("2")

    expect(Option.isSome(result)).toBe(true)
    if (Option.isSome(result)) {
      expect(result.value.name).toBe("Bob")
    }
  }).pipe(Effect.provide(UserRepositoryStateful))
)
```

## Test Layers

Create in-memory test implementations with `Layer.sync`:

```typescript
import { Context, Effect, Layer, Schema } from "effect"

const UserId = Schema.String.pipe(Schema.brand("UserId"))
type UserId = typeof UserId.Type

class User extends Schema.Class<User>("User")({
  id: UserId,
  name: Schema.String,
  email: Schema.String,
}) {}

class UserNotFound extends Schema.TaggedError<UserNotFound>()("UserNotFound", {
  id: UserId,
}) {}

// Users service with test layer that has create + findById
class Users extends Context.Tag("@app/Users")<
  Users,
  {
    readonly create: (user: User) => Effect.Effect<void>
    readonly findById: (id: UserId) => Effect.Effect<User, UserNotFound>
  }
>() {
  // Mutable state is fine in tests - JS is single-threaded
  static readonly testLayer = Layer.sync(Users, () => {
    const store = new Map<UserId, User>()

    const create = (user: User) => Effect.sync(() => void store.set(user.id, user))

    const findById = (id: UserId) =>
      Effect.fromNullable(store.get(id)).pipe(
        Effect.orElseFail(() => UserNotFound.make({ id }))
      )

    return Users.of({ create, findById })
  })
}
```

## Worked Example: Testing a Service

Here's a complete example testing an `Events` service that orchestrates `Users`, `Tickets`, and `Emails`:

```typescript
import { Clock, Context, Effect, Layer, Schema } from "effect"
import { describe, expect, it } from "@effect/vitest"

// Domain types
const RegistrationId = Schema.String.pipe(Schema.brand("RegistrationId"))
type RegistrationId = typeof RegistrationId.Type

const EventId = Schema.String.pipe(Schema.brand("EventId"))
type EventId = typeof EventId.Type

const UserId = Schema.String.pipe(Schema.brand("UserId"))
type UserId = typeof UserId.Type

const TicketId = Schema.String.pipe(Schema.brand("TicketId"))
type TicketId = typeof TicketId.Type

class User extends Schema.Class<User>("User")({
  id: UserId,
  name: Schema.String,
  email: Schema.String,
}) {}

class Registration extends Schema.Class<Registration>("Registration")({
  id: RegistrationId,
  eventId: EventId,
  userId: UserId,
  ticketId: TicketId,
  registeredAt: Schema.Date,
}) {}

class Ticket extends Schema.Class<Ticket>("Ticket")({
  id: TicketId,
  eventId: EventId,
  code: Schema.String,
}) {}

class Email extends Schema.Class<Email>("Email")({
  to: Schema.String,
  subject: Schema.String,
  body: Schema.String,
}) {}

class UserNotFound extends Schema.TaggedError<UserNotFound>()("UserNotFound", {
  id: UserId,
}) {}

// Leaf services with test layers
class Users extends Context.Tag("@app/Users")<
  Users,
  {
    readonly create: (user: User) => Effect.Effect<void>
    readonly findById: (id: UserId) => Effect.Effect<User, UserNotFound>
  }
>() {
  static readonly testLayer = Layer.sync(Users, () => {
    const store = new Map<UserId, User>()
    const create = (user: User) => Effect.sync(() => void store.set(user.id, user))
    const findById = (id: UserId) =>
      Effect.fromNullable(store.get(id)).pipe(
        Effect.orElseFail(() => UserNotFound.make({ id }))
      )
    return Users.of({ create, findById })
  })
}

class Tickets extends Context.Tag("@app/Tickets")<
  Tickets,
  { readonly issue: (eventId: EventId, userId: UserId) => Effect.Effect<Ticket> }
>() {
  static readonly testLayer = Layer.sync(Tickets, () => {
    let counter = 0
    const issue = (eventId: EventId, _userId: UserId) =>
      Effect.sync(() =>
        Ticket.make({
          id: TicketId.make(`ticket-${counter++}`),
          eventId,
          code: `CODE-${counter}`,
        })
      )
    return Tickets.of({ issue })
  })
}

class Emails extends Context.Tag("@app/Emails")<
  Emails,
  {
    readonly send: (email: Email) => Effect.Effect<void>
    readonly sent: Effect.Effect<ReadonlyArray<Email>>
  }
>() {
  static readonly testLayer = Layer.sync(Emails, () => {
    const emails: Array<Email> = []
    const send = (email: Email) => Effect.sync(() => void emails.push(email))
    const sent = Effect.sync(() => emails)
    return Emails.of({ send, sent })
  })
}

// Events service orchestrates leaf services
class Events extends Context.Tag("@app/Events")<
  Events,
  { readonly register: (eventId: EventId, userId: UserId) => Effect.Effect<Registration, UserNotFound> }
>() {
  static readonly layer = Layer.effect(
    Events,
    Effect.gen(function* () {
      const users = yield* Users
      const tickets = yield* Tickets
      const emails = yield* Emails

      const register = Effect.fn("Events.register")(
        function* (eventId: EventId, userId: UserId) {
          const user = yield* users.findById(userId)
          const ticket = yield* tickets.issue(eventId, userId)
          const now = yield* Clock.currentTimeMillis

          const registration = Registration.make({
            id: RegistrationId.make(crypto.randomUUID()),
            eventId,
            userId,
            ticketId: ticket.id,
            registeredAt: new Date(now),
          })

          yield* emails.send(
            Email.make({
              to: user.email,
              subject: "Event Registration Confirmed",
              body: `Your ticket code: ${ticket.code}`,
            })
          )

          return registration
        }
      )

      return Events.of({ register })
    })
  )
}

// Compose test layers - provideMerge exposes leaf services for setup/assertions
const testLayer = Events.layer.pipe(
  Layer.provideMerge(Users.testLayer),
  Layer.provideMerge(Tickets.testLayer),
  Layer.provideMerge(Emails.testLayer)
)

describe("Events.register", () => {
  it.effect("creates registration with correct data", () =>
    Effect.gen(function* () {
      const users = yield* Users
      const events = yield* Events

      // Arrange: create a user
      const user = User.make({
        id: UserId.make("user-123"),
        name: "Alice",
        email: "alice@example.com",
      })
      yield* users.create(user)

      // Act
      const eventId = EventId.make("event-789")
      const registration = yield* events.register(eventId, user.id)

      // Assert
      expect(registration.eventId).toBe(eventId)
      expect(registration.userId).toBe(user.id)
    }).pipe(Effect.provide(testLayer))
  )

  it.effect("sends confirmation email with ticket code", () =>
    Effect.gen(function* () {
      const users = yield* Users
      const events = yield* Events
      const emails = yield* Emails

      // Arrange
      const user = User.make({
        id: UserId.make("user-456"),
        name: "Bob",
        email: "bob@example.com",
      })
      yield* users.create(user)

      // Act
      yield* events.register(EventId.make("event-789"), user.id)

      // Assert: check sent emails
      const sentEmails = yield* emails.sent
      expect(sentEmails).toHaveLength(1)
      expect(sentEmails[0].to).toBe("bob@example.com")
      expect(sentEmails[0].subject).toBe("Event Registration Confirmed")
      expect(sentEmails[0].body).toContain("CODE-")
    }).pipe(Effect.provide(testLayer))
  )
})
```

## Running Tests

Run tests with vitest:

```bash
# Run all tests
bun run test

# Watch mode
bun run test:watch

# Run specific file
bunx vitest run tests/user.test.ts

# Run tests matching pattern
bunx vitest run -t "UserService"
```

## Property-Based Testing

### Using it.prop for Pure Properties

```typescript
import { FastCheck } from "effect"
import { it } from "@effect/vitest"

it.prop(
  "addition is commutative",
  [FastCheck.integer(), FastCheck.integer()],
  ([a, b]) => a + b === b + a
)

// With object syntax
it.prop(
  "multiplication distributes",
  { a: FastCheck.integer(), b: FastCheck.integer(), c: FastCheck.integer() },
  ({ a, b, c }) => a * (b + c) === a * b + a * c
)
```

### Using it.effect.prop for Effect Properties

```typescript
import { it } from "@effect/vitest"
import { Effect, Context, FastCheck } from "effect"

class Database extends Context.Tag("Database")<Database, {
  set: (key: string, value: number) => Effect.Effect<void>
  get: (key: string) => Effect.Effect<number>
}>() {}

it.effect.prop(
  "database operations are idempotent",
  [FastCheck.string(), FastCheck.integer()],
  ([key, value]) =>
    Effect.gen(function* () {
      const db = yield* Database

      yield* db.set(key, value)
      const result1 = yield* db.get(key)

      yield* db.set(key, value)
      const result2 = yield* db.get(key)

      return result1 === result2
    })
)
```

### With Schema Arbitraries

```typescript
import { it, expect } from "@effect/vitest"
import { Effect, Schema } from "effect"

const User = Schema.Struct({
  id: Schema.String,
  age: Schema.Number.pipe(Schema.between(0, 120))
})

it.effect.prop(
  "user validation works",
  { user: User },
  ({ user }) =>
    Effect.gen(function* () {
      expect(user.age).toBeGreaterThanOrEqual(0)
      expect(user.age).toBeLessThanOrEqual(120)
      return true
    })
)
```

### Configuring FastCheck

```typescript
import { it } from "@effect/vitest"
import { Effect, FastCheck } from "effect"

it.effect.prop(
  "property test",
  [FastCheck.integer()],
  ([n]) => Effect.succeed(n >= 0 || n < 0),
  {
    timeout: 10000,
    fastCheck: {
      numRuns: 1000,
      seed: 42,
      verbose: true
    }
  }
)
```

## Common Pitfalls

1. **Not Providing Layers**: Forgetting to provide required services causes runtime errors.

2. **Shared State Between Tests**: Tests interfering with each other via shared mutable state. Use fresh layers per test or reset state in beforeEach.

3. **Only Testing Happy Paths**: Not testing error scenarios. Use `Effect.flip` and `Effect.exit` to test failures.

4. **Missing Cleanup Tests**: Not verifying finalizers execute properly. Check that `Effect.addFinalizer` callbacks run.

5. **Ignoring Concurrency**: Not testing concurrent behavior. Use `Effect.fork` and `Fiber.join` to test parallel operations.

6. **Flaky Tests from Race Conditions**: Tests that sometimes pass, sometimes fail. Use `TestClock` for deterministic timing.

7. **Over-Mocking**: Mocking too much loses integration value. Only mock external services, not internal logic.

8. **Not Testing Interruption**: Missing interruption scenarios. Test that operations handle `Fiber.interrupt` gracefully.

9. **Hardcoded Timing**: Tests that depend on specific wall-clock timing. Use `TestClock.adjust` instead of `Effect.sleep` with real time.

10. **Missing Exit Checks**: Not verifying Exit values properly. Use `Exit.isSuccess`, `Exit.isFailure`, and `Cause.failureOption`.

## Next Steps

- Use [TestClock](https://effect.website/docs/guides/testing/test-clock) for time-dependent tests
- Use [TestRandom](https://effect.website/docs/guides/testing/test-random) for deterministic randomness
- See [Testing documentation](https://effect.website/docs/guides/testing/introduction) for advanced patterns
