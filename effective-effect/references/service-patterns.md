# Service Patterns

## Effect.Service vs Context.Tag

Both `Effect.Service` and `Context.Tag` are ways to model services in Effect. They serve similar purposes but target different use-cases.

| Feature | Effect.Service | Context.Tag |
|---------|---------------|-------------|
| Tag creation | Generated for you (class name acts as tag) | You declare the tag manually |
| Default implementation | Required - supplied inline (`effect`, `sync`, etc.) | Optional - can be supplied later |
| Ready-made layers | Automatically generated (`.Default`, etc.) | You build layers yourself |
| Best suited for | Application code with clear runtime implementation | Library code or dynamically-scoped values |
| When no sensible default exists | Not ideal; you'd still have to invent one | Preferred |

**Key points:**

- **Less boilerplate:** `Effect.Service` is syntactic sugar over `Context.Tag` plus the accompanying layer and helpers.
- **Default required:** A class that extends `Effect.Service` must declare one of the built-in constructors (`effect`, `sync`, `succeed`, or `scoped`). This becomes `MyService.Default`, so code can run without providing extra layers.
- **The class is the tag:** When you create a class with `extends Effect.Service`, the class constructor itself acts as the tag.

## Effect.Service (Preferred for App Code)

Use `Effect.Service` when you have a sensible default implementation. This is the common case for application-level services like logging, HTTP clients, databases, etc.

```typescript
import { Effect, Layer } from "effect"

export class UserService extends Effect.Service<UserService>()("@app/UserService", {
  accessors: true,
  dependencies: [UserRepo.Default, CacheService.Default],
  effect: Effect.gen(function* () {
    const repo = yield* UserRepo
    const cache = yield* CacheService

    const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
      const cached = yield* cache.get(id)
      if (Option.isSome(cached)) return cached.value

      const user = yield* repo.findById(id)
      yield* cache.set(id, user)
      return user
    })

    const create = Effect.fn("UserService.create")(function* (data: CreateUserInput) {
      const user = yield* repo.create(data)
      yield* Effect.log("User created", { userId: user.id })
      return user
    })

    return { findById, create }
  }),
}) {}

// Usage - dependencies already wired
const program = Effect.gen(function* () {
  const user = yield* UserService.findById(userId) // accessors enabled
  return user
})

// At app root - simple composition
const MainLive = Layer.mergeAll(UserService.Default, OtherService.Default)
```

### Dependencies Array

**Always declare dependencies** in the `dependencies` array. This ensures:
- Dependencies are automatically provided when using `ServiceName.Default`
- Type errors if dependencies are missing
- No manual `Layer.provide` at usage sites

```typescript
export class OrderService extends Effect.Service<OrderService>()("@app/OrderService", {
  accessors: true,
  dependencies: [
    UserService.Default,
    ProductService.Default,
    InventoryService.Default,
  ],
  effect: Effect.gen(function* () {
    const users = yield* UserService
    const products = yield* ProductService
    const inventory = yield* InventoryService

    // Service implementation...
    return { create, cancel }
  }),
}) {}
```

### Generated Layers

`Effect.Service` automatically generates two layers:

| Layer | Description |
|-------|-------------|
| `Service.Default` | Provides the service with dependencies already included |
| `Service.DefaultWithoutDependencies` | Provides the service but requires dependencies separately |

Use `DefaultWithoutDependencies` when testing to inject mock dependencies instead of mocking the service itself:

```typescript
// Production: Use Default (dependencies included)
const runnable = program.pipe(Effect.provide(UserService.Default))

// Testing: Use DefaultWithoutDependencies + mock dependencies
const runnable = program.pipe(
  Effect.provide(UserService.DefaultWithoutDependencies),
  Effect.provide(MockUserRepo.layer),
  Effect.provide(MockCacheService.layer)
)
```

### Providing Alternate Implementations

The class is the tag, so you can provide mock implementations for testing:

```typescript
const mockUserService = new UserService({
  findById: (id) => Effect.succeed(mockUser),
  create: (data) => Effect.succeed({ ...mockUser, ...data }),
})

program.pipe(Effect.provideService(UserService, mockUserService))
```

## Context.Tag (For Libraries / No Default)

Use `Context.Tag` when:

1. **No sensible default exists** - e.g., a per-request database handle
2. **Writing library code** - you publish only the tag, callers supply the layer
3. **Dynamically-scoped values** - values that change per request/context
4. **Service-driven development** - sketching interfaces before implementations

```typescript
import { Context, Effect, Layer } from "effect"

// Per-request handle - no global default makes sense
class RequestContext extends Context.Tag("@app/RequestContext")<
  RequestContext,
  {
    readonly requestId: string
    readonly userId: UserId
  }
>() {}

// Library code - callers provide their own implementation
class PaymentGateway extends Context.Tag("@lib/PaymentGateway")<
  PaymentGateway,
  {
    readonly charge: (amount: number) => Effect.Effect<Receipt, PaymentError>
  }
>() {}

// Cloudflare Worker bindings - provided by runtime
class KVNamespace extends Context.Tag("@cf/KVNamespace")<
  KVNamespace,
  CloudflareKVNamespace
>() {}
```

### Layer Implementation with Context.Tag

Use `Layer.effect()` to implement services:

```typescript
import { HttpClient, HttpClientResponse } from "@effect/platform"
import { Context, Effect, Layer, Schema } from "effect"

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
      const http = yield* HttpClient.HttpClient
      const analytics = yield* Analytics

      const findById = Effect.fn("Users.findById")(function* (id: UserId) {
        yield* analytics.track("user.find", { id })
        const response = yield* http.get(`https://api.example.com/users/${id}`)
        return yield* HttpClientResponse.schemaBodyJson(User)(response)
      }).pipe(
        Effect.catchTag("ResponseError", (error) =>
          error.response.status === 404
            ? UserNotFoundError.make({ id })
            : Effect.die(error)
        )
      )

      const all = Effect.fn("Users.all")(function* () {
        const response = yield* http.get("https://api.example.com/users")
        return yield* HttpClientResponse.schemaBodyJson(Schema.Array(User))(response)
      })

      return Users.of({ findById, all })
    })
  )
}
```

## Service-Driven Development

Start by sketching service contracts (without implementations). This lets you write higher-level orchestration that type-checks even though leaf services aren't runnable yet.

```typescript
import { Clock, Context, Effect, Layer, Schema } from "effect"

// Leaf services: contracts only (no implementation yet)
class Users extends Context.Tag("@app/Users")<
  Users,
  { readonly findById: (id: UserId) => Effect.Effect<User> }
>() {}

class Tickets extends Context.Tag("@app/Tickets")<
  Tickets,
  { readonly issue: (eventId: EventId, userId: UserId) => Effect.Effect<Ticket> }
>() {}

class Emails extends Context.Tag("@app/Emails")<
  Emails,
  { readonly send: (to: string, subject: string, body: string) => Effect.Effect<void> }
>() {}

// Higher-level service: orchestrates leaf services
class Events extends Context.Tag("@app/Events")<
  Events,
  { readonly register: (eventId: EventId, userId: UserId) => Effect.Effect<Registration> }
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
            user.email,
            "Event Registration Confirmed",
            `Your ticket code: ${ticket.code}`
          )

          return registration
        }
      )

      return Events.of({ register })
    })
  )
}
```

**Benefits:**
- Type-checks immediately even without implementations
- Leaf service contracts are explicit
- Adding production implementations later doesn't change Events code

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
    const records: Record<string, unknown> = {
      "user-1": { id: "user-1", name: "Alice" },
      "user-2": { id: "user-2", name: "Bob" },
    }

    const query = (sql: string) => Effect.succeed(Object.values(records))
    const execute = (sql: string) => Console.log(`Test execute: ${sql}`)

    return Database.of({ query, execute })
  })
}
```

For `Effect.Service`, you can provide mocks directly:

```typescript
const mockDb = new Database({
  query: () => Effect.succeed([mockData]),
  execute: () => Effect.void,
})

program.pipe(Effect.provideService(Database, mockDb))
```

## Providing Layers to Effects

Use `Effect.provide` once at the top of your application:

```typescript
// Compose all layers
const appLayer = userServiceLayer.pipe(
  Layer.provideMerge(databaseLayer),
  Layer.provideMerge(loggerLayer)
)

// Provide once at entry point
const main = program.pipe(Effect.provide(appLayer))
Effect.runPromise(main)
```

**Why provide once at the top?**
- Clear dependency graph: all wiring in one place
- Easier testing: swap `appLayer` for `testLayer`
- No hidden dependencies: effects declare what they need via types

## Effect.fn for Tracing

**Always wrap service methods with `Effect.fn`**. This provides automatic tracing with meaningful span names.

```typescript
const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
  yield* Effect.annotateCurrentSpan("userId", id)
  // Implementation
})

const processPayment = Effect.fn("PaymentService.processPayment")(
  function* (orderId: OrderId, amount: number, currency: string) {
    yield* Effect.annotateCurrentSpan("orderId", orderId)
    yield* Effect.annotateCurrentSpan("amount", amount)
    yield* Effect.annotateCurrentSpan("currency", currency)
    // Implementation
  }
)
```

## Single Responsibility

Each service should have a focused responsibility:

```typescript
// CORRECT - Focused services
class UserService extends Effect.Service<UserService>()("@app/UserService", { /* ... */ }) {}
class AuthService extends Effect.Service<AuthService>()("@app/AuthService", { /* ... */ }) {}
class NotificationService extends Effect.Service<NotificationService>()("@app/NotificationService", { /* ... */ }) {}

// WRONG - God service doing everything
class AppService extends Effect.Service<AppService>()("@app/AppService", {
  effect: Effect.gen(function* () {
    return {
      createUser, deleteUser, login, logout,
      sendEmail, sendPush, processPayment,
      // ... 50 more methods
    }
  }),
}) {}
```

## Service Interface Patterns

### Return Types

Services should return `Effect` types, never `Promise`:

```typescript
// CORRECT
readonly findById: (id: UserId) => Effect.Effect<User, UserNotFoundError>

// WRONG - Promise in service interface
readonly findById: (id: UserId) => Promise<User>
```

### Use Option for Nullable Results

```typescript
// findById fails with error if not found
readonly findById: (id: UserId) => Effect.Effect<User, UserNotFoundError>

// findByIdOption returns Option for "may not exist" semantics
readonly findByIdOption: (id: UserId) => Effect.Effect<Option<User>>
```

## Client Wrapper Pattern

Wrap third-party SDK clients (Stripe, Resend, AWS, etc.) with Effect using an internal `use` helper, then re-expose domain operations as named service methods.

### Pattern Structure

```typescript
import { Context, Effect, Layer, Config, Redacted, Schema } from "effect"

// 1. Define tagged errors with Schema.Defect for cause
export class MyClientSyncError extends Schema.TaggedError<MyClientSyncError>()(
  "MyClientSyncError",
  { cause: Schema.Defect }
) {}

export class MyClientAsyncError extends Schema.TaggedError<MyClientAsyncError>()(
  "MyClientAsyncError",
  { cause: Schema.Defect }
) {}

export class MyClientInstantiationError extends Schema.TaggedError<MyClientInstantiationError>()(
  "MyClientInstantiationError",
  { cause: Schema.Defect }
) {}

// Union type for convenience
export type MyClientError =
  | MyClientSyncError
  | MyClientAsyncError
  | MyClientInstantiationError

// 2. Define service interface with named operations (easy to mock)
export type IMyClient = Readonly<{
  someAsyncMethod: (
    params: { param: string }
  ) => Effect.Effect<AsyncResult, MyClientSyncError | MyClientAsyncError>
  someSyncMethod: () => Effect.Effect<SyncResult, MyClientSyncError | MyClientAsyncError>
}>

// 3. Create the service implementation
const make = Effect.gen(function* () {
  const apiKey = yield* Config.redacted("MY_CLIENT_API_KEY")

  const client = yield* Effect.try({
    try: () => new ThirdPartyClient(Redacted.value(apiKey)),
    catch: (cause) => new MyClientInstantiationError({ cause }),
  })

  // Internal helper for sync + async SDK methods
  const use = <T>(operation: string, fn: (client: ThirdPartyClient) => T) =>
    Effect.gen(function* () {
      const result = yield* Effect.try({
        try: () => fn(client),
        catch: (cause) => new MyClientSyncError({ cause }),
      })

      if (result instanceof Promise) {
        return yield* Effect.tryPromise({
          try: () => result,
          catch: (cause) => new MyClientAsyncError({ cause }),
        })
      }

      return result as Awaited<T>
    }).pipe(Effect.withSpan(`my_client.${operation}`))

  const someAsyncMethod = (params: { param: string }) =>
    use("someAsyncMethod", (client) => client.someAsyncMethod(params))

  const someSyncMethod = () =>
    use("someSyncMethod", (client) => client.someSyncMethod())

  return { someAsyncMethod, someSyncMethod }
})

// 4. Export as Context.Tag with Default layer
export class MyClient extends Context.Tag("MyClient")<MyClient, IMyClient>() {
  static Default = Layer.effect(this, make).pipe(
    Layer.annotateSpans({ module: "MyClient" })
  )
}
```

### Usage

```typescript
const program = Effect.gen(function* () {
  const myClient = yield* MyClient

  const asyncResult = yield* myClient.someAsyncMethod({ param: "value" })
  const syncResult = yield* myClient.someSyncMethod()

  return { asyncResult, syncResult }
})

// Run with layer
program.pipe(Effect.provide(MyClient.Default))
```

### Key Benefits

1. **Centralized error handling** - All client errors wrapped in typed errors
2. **Automatic tracing** - Internal `use` creates consistent spans for every SDK call
3. **Config-based secrets** - API keys loaded via `Config.redacted`
4. **Clean DI** - Consumers inject via `yield* MyClient`
5. **Encapsulation** - Raw client and `use` helper stay internal to the service
6. **Stack trace preservation** - `Schema.Defect` preserves the original error for debugging
7. **Sync/async agnostic** - Handles both sync and async client methods via `instanceof Promise` check
8. **Mock-friendly API** - Tests mock named methods instead of callback-based `use`

### Variations

#### Domain-Specific Error Types

Extend the base pattern with domain-specific errors:

```typescript
export class MyClientNetworkError extends Schema.TaggedError<MyClientNetworkError>()(
  "MyClientNetworkError",
  { cause: Schema.Defect }
) {}

export class MyClientValidationError extends Schema.TaggedError<MyClientValidationError>()(
  "MyClientValidationError",
  { message: Schema.String }
) {}

const mapSyncError = (cause: unknown) => {
  if (cause instanceof ValidationError) {
    return new MyClientValidationError({ message: cause.message })
  }
  return new MyClientSyncError({ cause })
}

const mapAsyncError = (cause: unknown) => {
  if (cause instanceof NetworkError) {
    return new MyClientNetworkError({ cause })
  }
  return new MyClientAsyncError({ cause })
}

const use = <T>(operation: string, fn: (client: ThirdPartyClient) => T) =>
  Effect.gen(function* () {
    const result = yield* Effect.try({
      try: () => fn(client),
      catch: mapSyncError,
    })

    if (result instanceof Promise) {
      return yield* Effect.tryPromise({
        try: () => result,
        catch: mapAsyncError,
      })
    }

    return result as Awaited<T>
  }).pipe(Effect.withSpan(`my_client.${operation}`))
```

#### Testing with a Mock Service

Provide only the named operations your app uses:

```typescript
export type IEmailClient = Readonly<{
  sendEmail: (params: SendEmailParams) => Effect.Effect<EmailResult, EmailError>
  getEmail: (id: string) => Effect.Effect<Email, EmailError>
}>

const mockEmailClient = EmailClient.of({
  sendEmail: (_params) => Effect.succeed({ id: "test-id", status: "queued" }),
  getEmail: (_id) => Effect.succeed({ id: "test-id", subject: "Hello" }),
})

program.pipe(Effect.provideService(EmailClient, mockEmailClient))
```

#### With Retry Policy

```typescript
import { Schedule } from "effect"

const retryPolicy = Schedule.exponential(100).pipe(
  Schedule.intersect(Schedule.recurs(3)),
  Schedule.jittered
)

const use = <T>(operation: string, fn: (client: ThirdPartyClient) => T) =>
  Effect.gen(function* () {
    const result = yield* Effect.try({
      try: () => fn(client),
      catch: (cause) => new MyClientSyncError({ cause }),
    })

    if (result instanceof Promise) {
      return yield* Effect.tryPromise({
        try: () => result,
        catch: (cause) => new MyClientAsyncError({ cause }),
      }).pipe(Effect.retry(retryPolicy))
    }

    return result as Awaited<T>
  }).pipe(Effect.withSpan(`my_client.${operation}`))
```

### Real-World Example: Stripe

```typescript
import Stripe from "stripe"
import { Context, Effect, Layer, Config, Redacted, Schema } from "effect"

export class StripeSyncError extends Schema.TaggedError<StripeSyncError>()(
  "StripeSyncError",
  { cause: Schema.Defect }
) {}

export class StripeAsyncError extends Schema.TaggedError<StripeAsyncError>()(
  "StripeAsyncError",
  { cause: Schema.Defect }
) {}

export type StripeError = StripeSyncError | StripeAsyncError

export type IStripeClient = Readonly<{
  createCustomer: (
    params: Stripe.CustomerCreateParams
  ) => Effect.Effect<Stripe.Customer, StripeError>
  retrieveCustomer: (id: string) => Effect.Effect<Stripe.Customer, StripeError>
}>

const make = Effect.gen(function* () {
  const secretKey = yield* Config.redacted("STRIPE_SECRET_KEY")

  const client = new Stripe(Redacted.value(secretKey))

  const use = <T>(operation: string, fn: (stripe: Stripe) => T) =>
    Effect.gen(function* () {
      const result = yield* Effect.try({
        try: () => fn(client),
        catch: (cause) => new StripeSyncError({ cause }),
      })

      if (result instanceof Promise) {
        return yield* Effect.tryPromise({
          try: () => result,
          catch: (cause) => new StripeAsyncError({ cause }),
        })
      }

      return result as Awaited<T>
    }).pipe(Effect.withSpan(`stripe.${operation}`))

  const createCustomer = (params: Stripe.CustomerCreateParams) =>
    use("customers.create", (stripe) => stripe.customers.create(params))

  const retrieveCustomer = (id: string) =>
    use("customers.retrieve", (stripe) => stripe.customers.retrieve(id)).pipe(
      Effect.flatMap((customer) =>
        customer.deleted
          ? Effect.fail(
              new StripeSyncError({
                cause: new Error("Customer is deleted"),
              })
            )
          : Effect.succeed(customer)
      )
    )

  return { createCustomer, retrieveCustomer }
})

export class StripeClient extends Context.Tag("StripeClient")<
  StripeClient,
  IStripeClient
>() {
  static Default = Layer.effect(this, make).pipe(
    Layer.annotateSpans({ module: "StripeClient" })
  )
}

// Usage
const createCustomer = Effect.gen(function* () {
  const stripe = yield* StripeClient

  const customer = yield* stripe.createCustomer({
    email: "user@example.com",
  })

  const existing = yield* stripe.retrieveCustomer(customer.id)

  return existing
})

// Testing
const mockStripeClient = StripeClient.of({
  createCustomer: (params) =>
    Effect.succeed({ id: "cus_test", email: params.email ?? null } as Stripe.Customer),
  retrieveCustomer: (_id) =>
    Effect.succeed({ id: "cus_test", email: "user@example.com" } as Stripe.Customer),
})

const testProgram = createCustomer.pipe(
  Effect.provideService(StripeClient, mockStripeClient)
)
```

### Guidance

1. Keep `use` private to the service module.
2. Expose only business-relevant operations (for example `createCustomer`, not `use`).
3. Keep span names and error mapping centralized inside `use`.
4. Prefer service-level mocks in tests over mocking SDK internals.
5. Add methods as needed; avoid exposing the raw SDK client.
