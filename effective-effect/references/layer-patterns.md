# Layer Patterns

## What is a Layer?

A Layer is an implementation of a service. Layers handle:

1. **Setup/initialization**: Connecting to databases, reading config, etc.
2. **Dependency resolution**: Acquiring other services they need
3. **Resource lifecycle**: Cleanup happens automatically

## Layer Implementation

Use `Layer.effect()` to implement services:

```typescript
import { Context, Effect, Layer } from "effect"

class Users extends Context.Tag("@app/Users")<
  Users,
  {
    readonly findById: (id: UserId) => Effect.Effect<User, UserNotFoundError>
    readonly all: () => Effect.Effect<readonly User[]>
  }
>() {
  static readonly layer = Layer.effect(
    Users,
    Effect.gen(function* () {
      // 1. yield* services you depend on
      const http = yield* HttpClient.HttpClient
      const analytics = yield* Analytics

      // 2. define methods with Effect.fn for call-site tracing
      const findById = Effect.fn("Users.findById")(function* (id: UserId) {
        yield* analytics.track("user.find", { id })
        const response = yield* http.get(`/users/${id}`)
        return yield* HttpClientResponse.schemaBodyJson(User)(response)
      })

      const all = Effect.fn("Users.all")(function* () {
        const response = yield* http.get("/users")
        return yield* HttpClientResponse.schemaBodyJson(Schema.Array(User))(response)
      })

      // 3. return the service
      return Users.of({ findById, all })
    })
  )
}
```

**Layer naming:** camelCase with descriptive suffix: `layer`, `testLayer`, `postgresLayer`, `sqliteLayer`

## Providing Layers to Effects

Use `Effect.provide` once at the top of your application. Avoid scattering `provide` calls throughout your codebase.

```typescript
import { Context, Effect, Layer } from "effect"

// Compose all layers into a single app layer
const appLayer = userServiceLayer.pipe(
  Layer.provideMerge(databaseLayer),
  Layer.provideMerge(loggerLayer),
  Layer.provideMerge(configLayer)
)

// Your program uses services freely
const program = Effect.gen(function* () {
  const users = yield* UserService
  const logger = yield* Logger
  yield* logger.info("Starting...")
  yield* users.getUser()
})

// Provide once at the entry point
const main = program.pipe(Effect.provide(appLayer))

Effect.runPromise(main)
```

**Why provide once at the top?**
- Clear dependency graph: all wiring in one place
- Easier testing: swap `appLayer` for `testLayer`
- No hidden dependencies: effects declare what they need via types
- Simpler refactoring: change wiring without touching business logic

## Layer Memoization

Effect automatically memoizes layers by reference identity. When the same layer instance appears multiple times in your dependency graph, it's constructed only once.

This matters especially for resource-intensive layers like database connection pools:

```typescript
import { Layer } from "effect"

// BAD: calling the constructor twice creates two connection pools
const badAppLayer = Layer.merge(
  UserRepo.layer.pipe(
    Layer.provide(Postgres.layer({ url: "postgres://localhost/mydb", poolSize: 10 }))
  ),
  OrderRepo.layer.pipe(
    Layer.provide(Postgres.layer({ url: "postgres://localhost/mydb", poolSize: 10 })) // Different reference!
  )
)
// Creates TWO connection pools (20 connections total). Could hit server limits.
```

**The fix:** Store the layer in a constant first:

```typescript
import { Layer } from "effect"

// GOOD: store the layer in a constant
const postgresLayer = Postgres.layer({ url: "postgres://localhost/mydb", poolSize: 10 })

const goodAppLayer = Layer.merge(
  UserRepo.layer.pipe(Layer.provide(postgresLayer)),
  OrderRepo.layer.pipe(Layer.provide(postgresLayer)) // Same reference!
)
// Single connection pool (10 connections) shared by both repos
```

**The rule:** When using parameterized layer constructors, always store the result in a module-level constant before using it in multiple places.

## Test Layers

Create lightweight test implementations with `Layer.sync` or `Layer.succeed`:

```typescript
import { Console, Context, Effect, Layer } from "effect"

class Database extends Context.Tag("@app/Database")<
  Database,
  {
    readonly query: (sql: string) => Effect.Effect<unknown[]>
    readonly execute: (sql: string) => Effect.Effect<void>
  }
>() {
  static readonly testLayer = Layer.sync(Database, () => {
    let records: Record<string, unknown> = {
      "user-1": { id: "user-1", name: "Alice" },
      "user-2": { id: "user-2", name: "Bob" },
    }

    const query = (sql: string) => Effect.succeed(Object.values(records))
    const execute = (sql: string) => Console.log(`Test execute: ${sql}`)

    return Database.of({ query, execute })
  })
}

class Cache extends Context.Tag("@app/Cache")<
  Cache,
  {
    readonly get: (key: string) => Effect.Effect<string | null>
    readonly set: (key: string, value: string) => Effect.Effect<void>
  }
>() {
  static readonly testLayer = Layer.sync(Cache, () => {
    const store = new Map<string, string>()

    const get = (key: string) => Effect.succeed(store.get(key) ?? null)
    const set = (key: string, value: string) => Effect.sync(() => void store.set(key, value))

    return Cache.of({ get, set })
  })
}
```

## Layer.mergeAll Over Nested Provides

**Use `Layer.mergeAll`** for composing layers at the same level:

```typescript
// CORRECT - Flat composition
const ServicesLive = Layer.mergeAll(
  UserService.layer,
  OrderService.layer,
  ProductService.layer,
  NotificationService.layer
)

const InfrastructureLive = Layer.mergeAll(
  DatabaseLive,
  RedisLive,
  HttpClientLive
)

const AppLive = ServicesLive.pipe(
  Layer.provide(InfrastructureLive)
)
```

```typescript
// WRONG - Deeply nested, hard to read
const AppLive = UserService.layer.pipe(
  Layer.provide(
    OrderService.layer.pipe(
      Layer.provide(
        ProductService.layer.pipe(
          Layer.provide(DatabaseLive)
        )
      )
    )
  )
)
```

## Layer Naming Conventions

Use static properties on the Tag class:

- `layer` - Production implementation
- `testLayer` - Test/mock implementation
- `postgresLayer`, `sqliteLayer` - Variant implementations

```typescript
class Users extends Context.Tag("@app/Users")<Users, { /* ... */ }>() {
  // Production
  static readonly layer = Layer.effect(Users, /* ... */)

  // Test with mocks
  static readonly testLayer = Layer.sync(Users, () => {
    const store = new Map<UserId, User>()
    // ...
    return Users.of({ findById, create })
  })
}
```

## Layer.effect vs Layer.succeed vs Layer.sync

```typescript
// Layer.succeed - for static values (no effects, no side effects)
const ConfigLive = Layer.succeed(AppConfig, {
  port: 3000,
  env: "development"
})

// Layer.sync - when construction has synchronous side effects
const CacheLive = Layer.sync(Cache, () => {
  const store = new Map<string, string>() // Side effect: creates state
  return Cache.of({
    get: (key) => Effect.succeed(store.get(key) ?? null),
    set: (key, value) => Effect.sync(() => void store.set(key, value))
  })
})

// Layer.effect - when construction needs async effects
const LoggerLive = Layer.effect(
  Logger,
  Effect.gen(function* () {
    const config = yield* AppConfig
    const transport = config.env === "production"
      ? yield* createCloudTransport()
      : createConsoleTransport()
    return Logger.of({ log: (msg) => transport.log(msg) })
  })
)
```

## Layer.unwrapEffect for Config-Dependent Layers

When a layer needs async configuration:

```typescript
import { Config, Effect, Layer } from "effect"

// Layer that depends on config
const ApiClientLive = Layer.unwrapEffect(
  Effect.gen(function* () {
    const apiKey = yield* Config.string("API_KEY")
    const baseUrl = yield* Config.string("API_BASE_URL")
    const timeout = yield* Config.integer("API_TIMEOUT").pipe(
      Config.withDefault(5000)
    )

    return Layer.succeed(
      ApiClient,
      ApiClient.of({ apiKey, baseUrl, timeout })
    )
  })
)
```

## Scoped Layers

For resources that need cleanup:

```typescript
import { Effect, Layer, Scope } from "effect"

// Resource that needs cleanup
const DatabaseConnectionLive = Layer.scoped(
  DatabaseConnection,
  Effect.acquireRelease(
    Effect.gen(function* () {
      const pool = yield* createPool(config)
      yield* Effect.log("Database pool created")
      return pool
    }),
    (pool) =>
      Effect.gen(function* () {
        yield* pool.end()
        yield* Effect.log("Database pool closed")
      }).pipe(Effect.orDie)
  )
)
```

## Testing Layer Composition

Use `Layer.provideMerge` to expose leaf services for test setup and assertions:

```typescript
import { Effect, Layer } from "effect"
import { describe, expect, it } from "@effect/vitest"

// provideMerge exposes leaf services in tests for setup/assertions
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

      // Arrange: create a user (accessing leaf service)
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
})
```

## Infrastructure Layers

Infrastructure layers (Database, Redis, HTTP clients) are typically provided once at the application root:

```typescript
import { PgClient } from "@effect/sql-pg"

const DatabaseLive = PgClient.layer({
  host: Config.string("DB_HOST"),
  port: Config.integer("DB_PORT"),
  database: Config.string("DB_NAME"),
  username: Config.string("DB_USER"),
  password: Config.secret("DB_PASSWORD")
})

// Services use database
class UserRepo extends Context.Tag("@app/UserRepo")<
  UserRepo,
  { readonly findById: (id: UserId) => Effect.Effect<User | undefined> }
>() {
  static readonly layer = Layer.effect(
    UserRepo,
    Effect.gen(function* () {
      const sql = yield* PgClient.PgClient

      const findById = Effect.fn("UserRepo.findById")(function* (id: UserId) {
        const rows = yield* sql`SELECT * FROM users WHERE id = ${id}`.pipe(Effect.orDie)
        return rows[0] as User | undefined
      })

      return UserRepo.of({ findById })
    })
  )
}

// App root provides infrastructure once
const AppLive = Layer.mergeAll(
  OrderService.layer,
  UserService.layer
).pipe(
  Layer.provide(DatabaseLive),
  Layer.provide(RedisLive)
)
```

## Lazy Layers

For expensive initialization that should be deferred:

```typescript
const ExpensiveServiceLive = Layer.lazy(() => {
  // This code runs only when the layer is first used
  return Layer.effect(
    ExpensiveService,
    Effect.gen(function* () {
      yield* Effect.log("Initializing expensive service...")
      const client = yield* createExpensiveClient()
      return ExpensiveService.of({ client })
    })
  )
})
```
