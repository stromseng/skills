# Schema Patterns

## Why Schema?

TypeScript's built-in tools for modeling data are limited. Effect's `Schema` library provides:

- **Single source of truth**: define once, get TypeScript types + runtime validation + JSON serialization
- **Parse safely**: validate HTTP/CLI/config data with detailed errors
- **Rich domain types**: branded primitives prevent confusion; classes add methods and behavior
- **Ecosystem integration**: use the same schema everywhere—RPC, HttpApi, CLI, frontend, backend

## Foundations

All representable data is composed of two primitives:

- **Records** (AND): a `User` has a name AND an email AND a createdAt
- **Variants** (OR): a `Result` is a Success OR a Failure

Schema gives you tools for both, plus runtime validation and serialization.

## Branded Types

Use branded types to prevent mixing values that have the same underlying type. **In a well-designed domain model, nearly all primitives should be branded.**

```typescript
import { Schema } from "effect"

// IDs - prevent mixing different entity IDs
export const UserId = Schema.String.pipe(Schema.brand("UserId"))
export type UserId = typeof UserId.Type

export const PostId = Schema.String.pipe(Schema.brand("PostId"))
export type PostId = typeof PostId.Type

// Domain primitives - create a rich type system
export const Email = Schema.String.pipe(Schema.brand("Email"))
export type Email = typeof Email.Type

export const Port = Schema.Int.pipe(Schema.between(1, 65535), Schema.brand("Port"))
export type Port = typeof Port.Type

// Usage - impossible to mix types
const userId = UserId.make("user-123")
const postId = PostId.make("post-456")
const email = Email.make("alice@example.com")

function getUser(id: UserId) { return id }
function sendEmail(to: Email) { return to }

// This works
getUser(userId)
sendEmail(email)

// All of these produce type errors
// getUser(postId)           // Can't pass PostId where UserId expected
// sendEmail(userId)         // Can't pass UserId where Email expected
// const bad: UserId = "raw" // Can't assign raw string to branded type
```

### When NOT to Brand

Don't brand simple strings that don't need type safety:

```typescript
// NOT branded - acceptable for internal-only strings
export const Url = Schema.String
export const FilePath = Schema.String
```

## Records (AND Types) with Schema.Class

Use `Schema.Class` for composite data models with multiple fields:

```typescript
import { Schema } from "effect"

const UserId = Schema.String.pipe(Schema.brand("UserId"))
type UserId = typeof UserId.Type

export class User extends Schema.Class<User>("User")({
  id: UserId,
  name: Schema.String,
  email: Schema.String,
  createdAt: Schema.Date,
}) {
  // Add custom getters and methods to extend functionality
  get displayName() {
    return `${this.name} (${this.email})`
  }
}

// Usage
const user = User.make({
  id: UserId.make("user-123"),
  name: "Alice",
  email: "alice@example.com",
  createdAt: new Date(),
})

console.log(user.displayName) // "Alice (alice@example.com)"
```

## Variants (OR Types) with Schema.TaggedClass

Use `Schema.Literal` for simple string or number alternatives:

```typescript
import { Schema } from "effect"

const Status = Schema.Literal("pending", "active", "completed")
type Status = typeof Status.Type // "pending" | "active" | "completed"
```

For structured variants with fields, combine `Schema.TaggedClass` with `Schema.Union`:

```typescript
import { Match, Schema } from "effect"

// Define variants with a tag field
export class Success extends Schema.TaggedClass<Success>()("Success", {
  value: Schema.Number,
}) {}

export class Failure extends Schema.TaggedClass<Failure>()("Failure", {
  error: Schema.String,
}) {}

// Create the union
export const Result = Schema.Union(Success, Failure)
export type Result = typeof Result.Type

// Pattern match with Match.valueTags
const success = Success.make({ value: 42 })
const failure = Failure.make({ error: "oops" })

Match.valueTags(success, {
  Success: ({ value }) => `Got: ${value}`,
  Failure: ({ error }) => `Error: ${error}`
}) // "Got: 42"

Match.valueTags(failure, {
  Success: ({ value }) => `Got: ${value}`,
  Failure: ({ error }) => `Error: ${error}`
}) // "Error: oops"
```

**Benefits:**
- Type-safe exhaustive matching
- Compiler ensures all cases handled
- No possibility of invalid states

## JSON Encoding & Decoding

Use `Schema.parseJson` to parse JSON strings and validate them with your schema in one step:

```typescript
import { Effect, Schema } from "effect"

const Row = Schema.Literal("A", "B", "C", "D", "E", "F", "G", "H")
const Column = Schema.Literal("1", "2", "3", "4", "5", "6", "7", "8")

class Position extends Schema.Class<Position>("Position")({
  row: Row,
  column: Column,
}) {}

class Move extends Schema.Class<Move>("Move")({
  from: Position,
  to: Position,
}) {}

// parseJson combines JSON.parse + schema decoding
// MoveFromJson is a schema that takes a JSON string and returns a Move
const MoveFromJson = Schema.parseJson(Move)

const program = Effect.gen(function* () {
  // Parse and validate JSON string in one step
  const jsonString = '{"from":{"row":"A","column":"1"},"to":{"row":"B","column":"2"}}'
  const move = yield* Schema.decodeUnknown(MoveFromJson)(jsonString)

  yield* Effect.log("Decoded move", move)

  // Encode to JSON string in one step (typed as string)
  const json = yield* Schema.encode(MoveFromJson)(move)
  return json
})
```

## Input Types for Mutations

```typescript
export const CreateUserInput = Schema.Struct({
  email: Schema.String.pipe(
    Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/),
    Schema.annotations({ description: "Valid email address" })
  ),
  name: Schema.String.pipe(
    Schema.minLength(1),
    Schema.maxLength(100)
  ),
  organizationId: OrganizationId,
  role: Schema.optionalWith(
    Schema.Literal("admin", "member", "viewer"),
    { default: () => "member" as const }
  ),
})
export type CreateUserInput = typeof CreateUserInput.Type

export const UpdateUserInput = Schema.Struct({
  name: Schema.optional(Schema.String.pipe(Schema.minLength(1))),
  role: Schema.optional(Schema.Literal("admin", "member", "viewer")),
})
export type UpdateUserInput = typeof UpdateUserInput.Type
```

## Schema.transform and transformOrFail

**Use transforms** instead of manual parsing:

```typescript
// Transform string to Date
export const DateFromString = Schema.transform(
  Schema.String,
  Schema.DateTimeUtc,
  {
    decode: (s) => new Date(s),
    encode: (d) => d.toISOString(),
  }
)

// Transform with validation (can fail)
export const PositiveNumber = Schema.transformOrFail(
  Schema.Number,
  Schema.Number.pipe(Schema.brand("PositiveNumber")),
  {
    decode: (n, _, ast) =>
      n > 0
        ? ParseResult.succeed(n as typeof PositiveNumber.Type)
        : ParseResult.fail(new ParseResult.Type(ast, n, "Must be positive")),
    encode: ParseResult.succeed,
  }
)
```

### Common Transforms

```typescript
// JSON string to object
export const JsonFromString = <A, I>(schema: Schema.Schema<A, I>) =>
  Schema.transform(
    Schema.String,
    schema,
    {
      decode: (s) => JSON.parse(s),
      encode: (a) => JSON.stringify(a),
    }
  )

// Comma-separated string to array
export const CommaSeparatedList = Schema.transform(
  Schema.String,
  Schema.Array(Schema.String),
  {
    decode: (s) => s.split(",").map((x) => x.trim()).filter(Boolean),
    encode: (arr) => arr.join(","),
  }
)

// Cents to dollars
export const DollarsFromCents = Schema.transform(
  Schema.Number.pipe(Schema.int()),
  Schema.Number,
  {
    decode: (cents) => cents / 100,
    encode: (dollars) => Math.round(dollars * 100),
  }
)
```

## Schema.annotations

Add annotations for documentation and validation messages:

```typescript
export const CreateOrderInput = Schema.Struct({
  productId: ProductId.pipe(
    Schema.annotations({ description: "The product to order" })
  ),
  quantity: Schema.Number.pipe(
    Schema.int(),
    Schema.positive(),
    Schema.annotations({
      description: "Number of items to order",
      examples: [1, 5, 10],
    })
  ),
  shippingAddress: Schema.Struct({
    line1: Schema.String.pipe(Schema.annotations({ description: "Street address" })),
    line2: Schema.optional(Schema.String),
    city: Schema.String,
    state: Schema.String.pipe(Schema.length(2)),
    zip: Schema.String.pipe(Schema.pattern(/^\d{5}(-\d{4})?$/)),
  }).pipe(Schema.annotations({ description: "Shipping destination" })),
}).pipe(
  Schema.annotations({
    title: "Create Order Input",
    description: "Input for creating a new order",
  })
)
```

## Optional Fields

Use `Schema.optional` and `Schema.optionalWith`:

```typescript
export const UserPreferences = Schema.Struct({
  // Optional, undefined if not provided
  theme: Schema.optional(Schema.Literal("light", "dark")),

  // Optional with default value
  language: Schema.optionalWith(Schema.String, { default: () => "en" }),

  // Optional with null support (for database compatibility)
  bio: Schema.NullOr(Schema.String),

  // Optional but must be present if set (no undefined)
  timezone: Schema.optional(Schema.String, { exact: true }),
})
```

## Discriminated Unions

```typescript
// Discriminated union (tagged)
export const PaymentDetails = Schema.Union(
  Schema.Struct({
    _tag: Schema.Literal("Card"),
    cardNumber: Schema.String,
    expiry: Schema.String,
    cvv: Schema.String,
  }),
  Schema.Struct({
    _tag: Schema.Literal("BankTransfer"),
    accountNumber: Schema.String,
    routingNumber: Schema.String,
  }),
  Schema.Struct({
    _tag: Schema.Literal("Crypto"),
    walletAddress: Schema.String,
    network: Schema.Literal("ethereum", "bitcoin", "solana"),
  }),
)
export type PaymentDetails = typeof PaymentDetails.Type

// Usage with Match.valueTags
import { Match } from "effect"

const process = (details: PaymentDetails) =>
  Match.valueTags(details, {
    Card: ({ cardNumber }) => `Processing card ${cardNumber}`,
    BankTransfer: ({ accountNumber }) => `Processing bank ${accountNumber}`,
    Crypto: ({ walletAddress }) => `Processing crypto ${walletAddress}`,
  })
```

## Enums and Literals

```typescript
// Use Literal for small, fixed sets
export const UserRole = Schema.Literal("admin", "member", "viewer")
export type UserRole = typeof UserRole.Type

// Use Enums for larger sets or when you need runtime values
export const OrderStatus = Schema.Enums({
  Pending: "pending",
  Processing: "processing",
  Shipped: "shipped",
  Delivered: "delivered",
  Cancelled: "cancelled",
} as const)
export type OrderStatus = typeof OrderStatus.Type
```

## Recursive Schemas

```typescript
interface Category {
  id: string
  name: string
  children: readonly Category[]
}

export const Category: Schema.Schema<Category> = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  children: Schema.Array(Schema.suspend(() => Category)),
})
```

## Decoding and Encoding

```typescript
// Decode (parse) - use in services
const parseUser = Schema.decodeUnknown(User)
const result = yield* parseUser(rawData) // Effect<User, ParseError>

// Decode sync - only in controlled contexts
const user = Schema.decodeUnknownSync(User)(rawData)

// Encode - for serialization
const encodeUser = Schema.encode(User)
const encoded = yield* encodeUser(user) // Effect<UserEncoded, ParseError>
```
