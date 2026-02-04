# Error Patterns

## Schema.TaggedError for All Errors

**Always use `Schema.TaggedError`** for defining errors. This provides:

1. **Serialization** - Errors can be sent over RPC/network
2. **Type safety** - `_tag` discriminator enables `catchTag`
3. **Consistent structure** - All errors have predictable shape
4. **Yieldable** - Can be used directly without `Effect.fail()`
5. **HTTP status mapping** - Via `HttpApiSchema.annotations`

### Basic Definition

```typescript
import { Schema } from "effect"

class ValidationError extends Schema.TaggedError<ValidationError>()(
  "ValidationError",
  {
    field: Schema.String,
    reason: Schema.String,
  }
) {}

class NotFoundError extends Schema.TaggedError<NotFoundError>()(
  "NotFoundError",
  {
    resource: Schema.String,
    id: Schema.String,
  }
) {}

const AppError = Schema.Union(ValidationError, NotFoundError)
type AppError = typeof AppError.Type

// Usage
const error = ValidationError.make({
  field: "email",
  reason: "Invalid format",
})
```

**Benefits:**
- Serializable (can send over network/save to DB)
- Type-safe
- Built-in `_tag` for pattern matching
- Custom methods via class
- Sensible default `message` when you don't declare one

## Yieldable Errors

`Schema.TaggedError` creates yieldable errors that can be used directly without `Effect.fail()`:

```typescript
import { Effect, Schema } from "effect"

class UserNotFoundError extends Schema.TaggedError<UserNotFoundError>()(
  "UserNotFoundError",
  { id: Schema.String }
) {}

// CORRECT - Yieldable errors can be used directly
const findUser = Effect.fn("findUser")(function* (id: string) {
  const user = yield* repo.findById(id)
  if (!user) {
    return yield* UserNotFoundError.make({ id }) // No Effect.fail needed!
  }
  return user
})

// Also works in catchTag handlers
const handled = effect.pipe(
  Effect.catchTag("ResponseError", (error) =>
    error.response.status === 404
      ? UserNotFoundError.make({ id }) // Yieldable!
      : Effect.die(error)
  )
)

// REDUNDANT - Effect.fail is not needed
const redundant = yield* Effect.fail(UserNotFoundError.make({ id }))
```

## Expected Errors vs Defects

Effect tracks errors in the type system (`Effect<A, E, R>`) so callers know what can go wrong and can recover. But tracking only matters if recovery is possible. When there's no sensible way to recover, use a defect instead.

**Use typed errors** for domain failures the caller can handle:
- Validation errors
- "Not found" errors
- Permission denied
- Rate limits

**Use defects** for unrecoverable situations:
- Bugs and invariant violations
- Configuration that's required for the app to function
- Corrupted data that shouldn't exist

```typescript
import { Effect } from "effect"

// At app entry: if config fails, nothing can proceed
const main = Effect.gen(function* () {
  const config = yield* loadConfig.pipe(Effect.orDie) // Convert to defect
  yield* Effect.log(`Starting on port ${config.port}`)
})
```

**When to catch defects:** Almost never. Only at system boundaries for logging/diagnostics. Use `Effect.exit` to inspect or `Effect.catchAllDefect` if you must recover (e.g., plugin sandboxing).

## Schema.Defect for Unknown Errors

Use `Schema.Defect` to wrap unknown errors from external libraries:

```typescript
import { Schema, Effect } from "effect"

class ApiError extends Schema.TaggedError<ApiError>()(
  "ApiError",
  {
    endpoint: Schema.String,
    statusCode: Schema.Number,
    // Wrap the underlying error from fetch/axios/etc
    error: Schema.Defect,
  }
) {}

// Usage - catching errors from external libraries
const fetchUser = (id: string) =>
  Effect.tryPromise({
    try: () => fetch(`/api/users/${id}`).then((r: Response) => r.json()),
    catch: (error) => ApiError.make({
      endpoint: `/api/users/${id}`,
      statusCode: 500,
      error
    })
  })
```

**Schema.Defect handles:**
- JavaScript `Error` instances -> `{ name, message }` objects
- Any unknown value -> string representation
- Serializable for network/storage

**Use for:**
- Wrapping errors from external libraries (fetch, axios, etc)
- Network boundaries (API errors)
- Persisting errors to DB
- Logging systems

## Error Recovery

Effect provides several functions for recovering from errors.

### catchAll

Handle all errors by providing a fallback effect:

```typescript
import { Effect, Schema } from "effect"

class HttpError extends Schema.TaggedError<HttpError>()(
  "HttpError",
  { statusCode: Schema.Number, reason: Schema.String }
) {}

class ValidationError extends Schema.TaggedError<ValidationError>()(
  "ValidationError",
  { field: Schema.String, reason: Schema.String }
) {}

declare const program: Effect.Effect<string, HttpError | ValidationError>

const recovered: Effect.Effect<string, never> = program.pipe(
  Effect.catchAll((error) =>
    Effect.gen(function* () {
      yield* Effect.logError("Error occurred", error)
      return `Recovered from ${error._tag}: ${error.reason}`
    })
  )
)
```

### catchTag

Handle specific errors by their `_tag`:

```typescript
const recovered: Effect.Effect<string, ValidationError> = program.pipe(
  Effect.catchTag("HttpError", (error) =>
    Effect.gen(function* () {
      yield* Effect.logWarning(`HTTP ${error.statusCode}: ${error.reason}`)
      return "Recovered from HttpError"
    })
  )
)
```

### catchTags

Handle multiple error types at once:

```typescript
const recovered: Effect.Effect<string, never> = program.pipe(
  Effect.catchTags({
    HttpError: () => Effect.succeed("Recovered from HttpError"),
    ValidationError: () => Effect.succeed("Recovered from ValidationError")
  })
)
```

## Backend Error Definitions

Business-logic errors should be explicit and rich in context:

```typescript
import { Schema } from "effect"

export class InsufficientBalanceError extends Schema.TaggedError<InsufficientBalanceError>()(
  "InsufficientBalanceError",
  {
    reason: Schema.String,
    currentBalance: Schema.Number,
    requiredAmount: Schema.Number,
    accountId: Schema.String,
  }
) {}
```

When wrapping other APIs or libraries, use `cause: Schema.Defect` to capture the original error:

```typescript
import { Schema } from "effect"

export class DatabaseUpsertError extends Schema.TaggedError<DatabaseUpsertError>()(
  "DatabaseUpsertError",
  {
    reason: Schema.String,
    cause: Schema.Defect,
  }
) {}

export class UserLookupError extends Schema.TaggedError<UserLookupError>()(
  "UserLookupError",
  {
    userId: Schema.String,
    reason: Schema.String,
    cause: Schema.Defect,
  }
) {}
```

## Boundary/API Errors

Frontend/API errors should be remapped at the boundary to avoid leaking internal details:

```typescript
import { Schema } from "effect"
import { HttpApiSchema } from "@effect/platform"

export class UserNotFoundError extends Schema.TaggedError<UserNotFoundError>()(
  "UserNotFoundError",
  {
    userId: UserId,
    userMessage: Schema.String,
  },
  HttpApiSchema.annotations({ status: 404 })
) {}

export class UserCreateError extends Schema.TaggedError<UserCreateError>()(
  "UserCreateError",
  {
    userMessage: Schema.String,
    userCause: Schema.optional(Schema.String),
  },
  HttpApiSchema.annotations({ status: 400 })
) {}

export class SessionExpiredError extends Schema.TaggedError<SessionExpiredError>()(
  "SessionExpiredError",
  {
    sessionId: SessionId,
    expiredAt: Schema.DateTimeUtc,
    userMessage: Schema.String,
  },
  HttpApiSchema.annotations({ status: 401 })
) {}
```

## Error Fields

### Backend Fields

- `reason: Schema.String` - Human readable explanation of why something went wrong
- `cause: Schema.Defect` - Wraps nested errors, `Effect.log` prints as stack trace
- `userMessage: Schema.optional(Schema.String)` - For errors that may reach users
- Relevant context fields (IDs, etc.)

> **⚠️ Warning:** Never use `message` as a field name. When present, `Effect.log` will only print the `message` value and **omit all other fields and contextual information**. This hides important diagnostics and makes debugging significantly harder. Use `reason` for backend errors and `userMessage` for boundary errors.

### Frontend Fields

- `userMessage: Schema.String` - Safe to display to users
- `userCause: Schema.optional(Schema.String)` - Additional context for users
- `retryable: Schema.Boolean` - Whether the operation can be retried
- Relevant context fields (IDs, etc.) for UI handling

## Why Explicit Error Types?

Generic errors like `BadRequestError` or `NotFoundError` seem convenient but create problems:

| Generic Error         | Problems                                                |
| --------------------- | ------------------------------------------------------- |
| `NotFoundError`       | Which resource? How should frontend recover?            |
| `BadRequestError`     | What's invalid? Can user fix it?                        |
| `UnauthorizedError`   | Session expired? Wrong credentials? Missing permission? |
| `InternalServerError` | Retryable? User action needed?                          |

**Explicit errors enable:**

1. **Specific UI messages** - "Your session expired" vs generic "Unauthorized"
2. **Targeted recovery** - Refresh token vs show login page
3. **Better observability** - Group errors by specific type in dashboards
4. **Type-safe handling** - `catchTag("SessionExpiredError")` vs generic catch

### Anti-Pattern: Generic Error Mapping

```typescript
// WRONG - Collapsing to generic HTTP errors
export class NotFoundError extends Schema.TaggedError<NotFoundError>()(
  "NotFoundError",
  { userMessage: Schema.String },
  HttpApiSchema.annotations({ status: 404 })
) {}

// At API boundaries:
Effect.catchTags({
  UserNotFoundError: (err) => Effect.fail(new NotFoundError({ userMessage: "Not found" })),
  ChannelNotFoundError: (err) => Effect.fail(new NotFoundError({ userMessage: "Not found" })),
  MessageNotFoundError: (err) => Effect.fail(new NotFoundError({ userMessage: "Not found" })),
})

// Frontend receives: { _tag: "NotFoundError", userMessage: "Not found" }
// - Can't show specific message
// - Can't take specific action
// - Debugging is harder
```

```typescript
// CORRECT - Keep explicit errors all the way to frontend
export class UserNotFoundError extends Schema.TaggedError<UserNotFoundError>()(
  "UserNotFoundError",
  { userId: UserId, userMessage: Schema.String },
  HttpApiSchema.annotations({ status: 404 })
) {}

export class ChannelNotFoundError extends Schema.TaggedError<ChannelNotFoundError>()(
  "ChannelNotFoundError",
  { channelId: ChannelId, userMessage: Schema.String },
  HttpApiSchema.annotations({ status: 404 })
) {}

// Frontend can handle each case:
Result.builder(result)
  .onErrorTag("UserNotFoundError", (err) => <UserNotFoundMessage userId={err.userId} />)
  .onErrorTag("ChannelNotFoundError", (err) => <ChannelDeletedMessage />)
  .onErrorTag("SessionExpiredError", () => <RedirectToLogin />)
  .render()
```

## Error Naming Conventions

### Backend-only (Internal) Errors

| Pattern                 | Example                                         | Use For                    |
| ----------------------- | ----------------------------------------------- | -------------------------- |
| `{Integration}Error`    | `StripeCallError`, `WorkOSFetchError`           | External API call failures |
| `{Layer}{Action}Error`  | `DatabaseUpsertError`, `CacheGetError`          | Infrastructure operations  |
| `{Entity}RepoError`     | `UserRepoError`, `ChannelRepoError`             | Repository layer errors    |
| `{Service}Error`        | `AuthServiceError`, `PaymentServiceError`       | Service layer errors       |

### Boundary / Contract Errors (Frontend/API)

| Pattern                   | Example                                           | Use For                      |
| ------------------------- | ------------------------------------------------- | ---------------------------- |
| `{Entity}NotFoundError`   | `UserNotFoundError`, `ChannelNotFoundError`       | Resource lookups             |
| `{Entity}{Action}Error`   | `UserCreateError`, `MessageUpdateError`           | Mutations that fail          |
| `{Feature}Error`          | `SessionExpiredError`, `RateLimitExceededError`   | Feature-specific failures    |
| `Invalid{Field}Error`     | `InvalidEmailError`, `InvalidPasswordError`       | Validation failures          |

## Error Handling with catchTag/catchTags

**Never use `catchAll` or `mapError`** when you can use `catchTag`/`catchTags`. These preserve type information and enable precise error handling.

### catchTag for Single Error Types

```typescript
const findUser = Effect.fn("UserService.findUser")(function* (id: UserId) {
  return yield* repo.findById(id).pipe(
    Effect.catchTag("DatabaseError", (err) =>
      UserNotFoundError.make({
        userId: id,
        userMessage: "User not found",
      })
    )
  )
})
```

### catchTags for Multiple Error Types

```typescript
const processOrder = Effect.fn("OrderService.processOrder")(function* (input: OrderInput) {
  return yield* validateAndProcess(input).pipe(
    Effect.catchTags({
      ValidationError: (err) =>
        OrderValidationError.make({
          reason: err.reason,
          field: err.field,
          cause: err, // Preserves stack trace in Effect.log
        }),
      PaymentError: (err) =>
        OrderPaymentError.make({
          reason: `Payment failed: ${err.reason}`,
          code: err.code,
          cause: err, // Preserves stack trace in Effect.log
        }),
      InventoryError: (err) =>
        OrderInventoryError.make({
          productId: err.productId,
          reason: "Insufficient inventory",
          cause: err, // Preserves stack trace in Effect.log
        }),
    })
  )
})
```

### Why Not catchAll?

```typescript
// WRONG - Loses type information AND underlying error
yield* effect.pipe(
  Effect.catchAll((err) => Effect.fail(new InternalServerError({ reason: "Something failed" })))
)

// Problems:
// 1. Can't distinguish error types downstream
// 2. Hides useful error context
// 3. Makes debugging harder - no cause means no stack trace
// 4. Frontend can't show specific messages
```

## Retryable Errors Pattern

For errors that may be transient, add a `retryable` property:

```typescript
export class ServiceUnavailableError extends Schema.TaggedError<ServiceUnavailableError>()(
  "ServiceUnavailableError",
  {
    userMessage: Schema.String,
    retryable: Schema.optionalWith(Schema.Boolean, { default: () => true }),
  },
  HttpApiSchema.annotations({ status: 503 })
) {}

export class RateLimitError extends Schema.TaggedError<RateLimitError>()(
  "RateLimitError",
  {
    userMessage: Schema.String,
    retryAfter: Schema.optional(Schema.Number),
    retryable: Schema.optionalWith(Schema.Boolean, { default: () => true }),
  },
  HttpApiSchema.annotations({ status: 429 })
) {}

// Non-retryable error
export class ValidationError extends Schema.TaggedError<ValidationError>()(
  "ValidationError",
  {
    userMessage: Schema.String,
    field: Schema.String,
    retryable: Schema.optionalWith(Schema.Boolean, { default: () => false }),
  },
  HttpApiSchema.annotations({ status: 400 })
) {}
```

### Retry Based on Error Property

```typescript
import { Effect, Schedule } from "effect"

const withRetry = <A, E extends { retryable?: boolean }, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  effect.pipe(
    Effect.retry(
      Schedule.exponential("100 millis").pipe(
        Schedule.intersect(Schedule.recurs(3)),
        Schedule.whileInput((err: E) => err.retryable === true)
      )
    )
  )

// Usage
yield* callExternalApi(request).pipe(withRetry)
```

## Error Logging

Log errors with structured context:

```typescript
const processWithLogging = Effect.fn("OrderService.process")(function* (orderId: OrderId) {
  return yield* processOrder(orderId).pipe(
    Effect.tapError((err) =>
      Effect.log("Order processing failed", {
        orderId,
        errorTag: err._tag,
      })
    )
  )
})
```
