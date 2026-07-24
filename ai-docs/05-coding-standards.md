# Coding Standards

**Document:** `ai-docs/05-coding-standards.md`
**Project:** Arwal — The District Super App
**Stage:** 1 — Foundation
**Phase:** 6 — Coding Standards
**Status:** Approved for Engineering Reference
**Audience:** Architects, Engineers, Engineering Managers, Technical Reviewers, Government Technical Partners

> **Callout — Purpose of This Document**
> `ai-docs/00-project-vision.md` defined why Arwal exists. `ai-docs/01-product-goals.md` defined what Arwal must achieve. `ai-docs/02-engineering-principles.md` defined how engineers reason about design. `ai-docs/03-system-architecture-principles.md` defined how the system is structured. `ai-docs/04-folder-guidelines.md` defined where every piece of that structure physically lives. This document defines **how code is actually written, line by line** — the syntax-level, file-level, and pattern-level rules that make the previous five documents enforceable in a pull request, not just true in principle.

---

# Purpose of this Document

Architecture and folder structure describe the shape of the system. Coding standards describe the substance that fills that shape. A perfectly drawn module boundary (`ai-docs/03-system-architecture-principles.md`) placed in a perfectly named folder (`ai-docs/04-folder-guidelines.md`) can still contain a God Class, an untyped `any`, a swallowed exception, or a SQL-injectable query — and at that point, the architecture is a diagram, not a defense.

This document exists to:

1. Translate the engineering philosophy of `ai-docs/02-engineering-principles.md` (SOLID, DRY, KISS, YAGNI, Separation of Concerns) into **concrete, syntax-level rules** for TypeScript, React, and backend code.
2. Give every engineer a **single, citable reference** for how a variable, function, class, endpoint, table, or test should be written — removing thousands of small, inconsistent, re-litigated decisions from the daily development loop.
3. Make code review **objective wherever possible** — "this violates the Naming Standards in Phase 6" is exactly as legitimate and actionable as citing a boundary from Phase 4 or a principle from Phase 3.
4. Protect the **behavioral guarantees** the architecture depends on — a repository that swallows errors, a domain entity that isn't actually immutable where it claims to be, or a controller that leaks business logic, quietly breaks the System Layers model in Phase 4 even though the folder structure in Phase 5 looks correct.
5. Serve as the document engineers keep open in a side tab — Phase 1 through Phase 5 are read once at onboarding; this document is referenced daily, for the life of the project.

This document assumes familiarity with all five preceding phase documents and does not repeat their reasoning — it applies that reasoning at the level of an individual line of code, an individual file, an individual pull request.

---

# Coding Philosophy

Coding standards exist to make five philosophical commitments enforceable, not aspirational.

### Readability Over Cleverness

Code is read by other engineers — and by the same engineer, months later, with no memory of the clever trick — far more often than it is written. A one-line, dense, "clever" solution that requires a comment to explain what it does has already failed; the comment is a confession that the code did not communicate itself. Arwal engineers write the version of the code a tired engineer at 6pm, mid-incident, can understand in one pass — not the version that best demonstrates language mastery.

```typescript
// Rejected — clever, requires decoding
const active = users.filter(u => !!(u.status ^ INACTIVE_MASK));

// Required — reads as a sentence
const activeUsers = users.filter((user) => user.status !== UserStatus.Inactive);
```

### Maintainability

Every line of code is a liability the moment it is written — it must be read, tested, reasoned about, and eventually changed or removed. Maintainable code minimizes the cost of all four. This means favoring small, focused functions and modules; avoiding hidden side effects; and never optimizing for "fewer lines" as an end in itself. Lines of code are not a scarce resource at Arwal; comprehension time is.

### Simplicity

Per KISS in `ai-docs/02-engineering-principles.md`, the simplest solution that correctly and durably solves the actual, present problem is the default. A design pattern, an abstraction layer, or a generic utility is introduced only when a demonstrated need justifies its complexity cost — never because it is available, fashionable, or "more proper."

### Explicitness

Nothing in Arwal's codebase is implicit where it could be explicit: types are declared, not inferred from usage three files away; dependencies are injected, not reached for globally; error paths are handled, not assumed away; configuration is loaded and validated, not silently defaulted. Implicit behavior is a debt that compounds silently — exactly the failure mode `ai-docs/02-engineering-principles.md` identifies as fatal at Arwal's scale.

### Consistency

The same category of problem is solved the same way, everywhere in the codebase, every time. Consistency is what allows an engineer's understanding of one module to transfer, at zero additional cost, to the next module they open — this is the same principle that motivates the Naming Conventions and folder templates in `ai-docs/04-folder-guidelines.md`, applied here at the level of code idioms and patterns.

> **Callout — The One-Sentence Coding Philosophy**
> *"If two engineers, working independently, would write this differently, the standard hasn't done its job — write it down."*

---

# General Code Principles

These principles restate the Core Engineering Principles of `ai-docs/02-engineering-principles.md` specifically as they manifest in day-to-day code, with concrete before/after examples.

### SOLID in Practice

| Principle | Code-Level Manifestation | Example |
|---|---|---|
| **SRP** | A class/function has one reason to change. If describing what a function does requires the word "and," split it. | `validateBooking()` and `saveBooking()` are separate functions, never merged into `validateAndSaveBooking()`. |
| **OCP** | New behavior is added via a new implementation of an existing interface, not by editing a branching conditional in shared code. | Adding a new `PaymentProvider` means a new class implementing `PaymentProvider`, never a new `case` in an existing `switch (providerName)`. |
| **LSP** | Any two classes implementing the same interface are interchangeable from the caller's perspective, with no special-casing. | A caller of `NotificationChannel.send()` never checks `if (channel instanceof SmsChannel)` to behave differently. |
| **ISP** | Interfaces are small and role-specific, not broad "do everything" contracts. | `Readable` and `Writable` are separate interfaces, not one `ReadWritable` interface forced on read-only consumers. |
| **DIP** | Business logic depends on an interface it owns, with the concrete implementation injected — never imported directly. | `BookingService` takes a `BookingRepository` interface in its constructor; it never imports `PostgresBookingRepository` directly. |

### DRY — Don't Repeat Yourself

Duplication of a **business decision** (e.g., the 2-hour cancellation cutoff) is always eliminated by extracting it to one authoritative function or constant. Duplication of **coincidentally similar code** — two functions that happen to look alike today but represent different business concepts — is left alone, per the callout in `ai-docs/02-engineering-principles.md`. The test is always: "if this changes for one reason, does it change for the other too?" If no, they are not the same knowledge, and merging them is a bug waiting to happen.

### KISS — Keep It Simple, Stupid

Prefer a plain function over a class when no state or polymorphism is needed. Prefer a straightforward `if/else` over a strategy-pattern object when there are two stable cases. Complexity is earned by a real requirement, never budgeted in advance.

### YAGNI — You Aren't Gonna Need It

No unused generic type parameters "for future flexibility." No configuration flags for behavior nothing currently needs. No abstract base class with a single concrete subclass "in case we need a second one." Extensibility is a property of clean boundaries (interfaces, DTOs, event contracts), not of speculative code paths that exist only in anticipation.

### Composition Over Inheritance

Behavior is composed from small, focused units — functions, hooks, injected services — rather than assembled through inheritance chains. Inheritance is reserved for genuine is-a relationships with a stable, shallow hierarchy (rarely more than one level deep in Arwal's domain code).

### Separation of Concerns

A function that validates input never also persists data. A React component that renders UI never also contains a `fetch` call with business logic embedded in its `.then()`. A controller never contains a pricing calculation. Each concern lives in the layer `ai-docs/03-system-architecture-principles.md` assigns it, without exception.

### Single Responsibility, Applied to Functions

As a practical heuristic: if a function cannot be described in one short sentence without "and," "then," or "also," it has more than one responsibility and should be split.

---

# TypeScript Standards

TypeScript is Arwal's standard language across `apps/api`, `apps/web`, `apps/admin-web`, and `packages/*`. These rules exist to make the type system an active defense against defects, not decorative annotation.

### Strict Mode

Every `tsconfig.json` in the monorepo extends `packages/config`'s base configuration, which enables `strict: true` in full (`strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictPropertyInitialization`, and all other strict-family flags). Strict mode is never selectively disabled per file or per package — a module that "needs" a strict-mode exception is a module with a design problem, not a valid exception case. `// @ts-ignore` is forbidden; `// @ts-expect-error` is permitted only with an inline comment explaining why the error is expected and tracked, and is treated as tracked technical debt per `ai-docs/02-engineering-principles.md`.

### Explicit Types

Function parameters and return types are always explicitly annotated at public/exported boundaries (module `index.ts`, use cases, repository interfaces, API DTOs). Local variable types may rely on inference where the initializer makes the type unambiguous — inference is a readability aid, not a way to hide an unclear contract.

```typescript
// Required — explicit at the public boundary
export function calculateBookingPrice(
  baseRate: Money,
  distanceKm: number,
  promotion: Promotion | null
): Money {
  // ...
}

// Acceptable — inference is unambiguous for a local variable
const total = baseRate.amount + surcharge.amount;
```

`any` is forbidden outside of narrowly justified, commented infrastructure-boundary code (e.g., a raw third-party SDK response before it is mapped to a typed DTO). `unknown` is the required default for genuinely unknown external input, narrowed via validation before use.

### Interface vs. Type

| Use Case | Choice | Rationale |
|---|---|---|
| Shape of an object that may be extended or implemented (DTOs, domain entities, repository contracts) | `interface` | Interfaces support declaration merging and are the conventional contract shape for OOP-style extension points, consistent with Dependency Inversion in `ai-docs/03-system-architecture-principles.md`. |
| Unions, intersections, tuples, mapped/conditional types, primitive aliases | `type` | Only `type` can express these constructs; `interface` cannot model a union. |
| A React component's props | `interface`, suffixed `Props` | Consistency with the Naming Standards below; also supports future extension via declaration merging if a third-party pattern requires it. |

Once a shape is declared as an `interface`, it is not switched to `type` without a reason beyond preference — consistent with Consistency Over Local Preference in `ai-docs/04-folder-guidelines.md`.

### Enums

TypeScript `enum` is used for a small, closed, known set of named values with business meaning (`BookingStatus`, `UserRole`). String enums are preferred over numeric enums, because a string enum's serialized value is self-describing in logs, database rows, and API payloads, where a numeric enum's value is not.

```typescript
// Required
enum BookingStatus {
  Pending = "PENDING",
  Confirmed = "CONFIRMED",
  Completed = "COMPLETED",
  Cancelled = "CANCELLED",
}

// Avoided — numeric enum, opaque in logs and DB rows
enum BookingStatus {
  Pending,
  Confirmed,
  Completed,
  Cancelled,
}
```

For a small fixed set of string literals with no need for reverse-mapping or iteration, a union of string literals (`type Role = "citizen" | "merchant" | "officer"`) is an acceptable, lighter-weight alternative — the choice between the two is made once per domain concept and applied consistently within that domain.

### Generics

Generics are used to express a genuinely reusable, type-safe relationship (a `Repository<T>` interface, a `Result<T, E>` type) — never introduced speculatively for a function with exactly one current call-site type. A generic parameter without a clear, demonstrated need to vary is a YAGNI violation, per `ai-docs/02-engineering-principles.md`.

```typescript
// Justified — genuinely reusable across every module's repository
interface Repository<TEntity, TId> {
  findById(id: TId): Promise<TEntity | null>;
  save(entity: TEntity): Promise<void>;
}

// Rejected — generic with a single caller, no real variability
function wrap<T>(value: T): { value: T } {
  return { value };
}
```

### Utility Types

Built-in utility types (`Partial`, `Pick`, `Omit`, `Readonly`, `Record`) are preferred over hand-rolled equivalents, consistent with Convention over Configuration. A DTO derived from a domain entity uses `Omit`/`Pick` against the entity type rather than being retyped from scratch, keeping the two in sync structurally.

```typescript
interface Booking {
  id: string;
  citizenId: string;
  providerId: string;
  status: BookingStatus;
  scheduledAt: Date;
  createdAt: Date;
}

// Derived, not duplicated
type CreateBookingDto = Omit<Booking, "id" | "status" | "createdAt">;
```

### Readonly and Immutability

Domain entities' identity fields, and all Value Objects in full, are declared `readonly`. A Value Object (`Money`, `TimeSlot`, `GeoCoordinate`) is never mutated in place — a "change" always produces a new instance, consistent with the Value Object definition in `ai-docs/03-system-architecture-principles.md`.

```typescript
class Money {
  private constructor(
    public readonly amount: number,
    public readonly currency: string
  ) {}

  static of(amount: number, currency: string): Money {
    return new Money(amount, currency);
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new CurrencyMismatchError(this.currency, other.currency);
    }
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

### Const Assertions

`as const` is used for object/array literals intended to be treated as immutable, exact-typed configuration or lookup data — it removes an entire class of accidental-mutation bugs at compile time and produces more precise literal types than a mutable equivalent.

```typescript
const DISTRICT_TIERS = ["headquarters", "town", "village"] as const;
type DistrictTier = (typeof DISTRICT_TIERS)[number];
```

### Null Safety

`strictNullChecks` is always on (see Strict Mode above). `null` and `undefined` are used deliberately and distinctly, not interchangeably: `undefined` represents "not provided" (an optional field, an uninitialized value); `null` represents "explicitly known to have no value" (a repository's `findById` that found nothing). The non-null assertion operator (`!`) is forbidden outside of tests and narrowly justified, commented cases immediately following a runtime check the compiler cannot see. Optional chaining (`?.`) and nullish coalescing (`??`) are the required tools for safe access and defaulting, in place of manual truthy checks that silently also catch `0` or `""`.

```typescript
// Rejected — silently wrong for a zero or empty-string discount
const discount = promotion.discount || DEFAULT_DISCOUNT;

// Required — only null/undefined trigger the default
const discount = promotion.discount ?? DEFAULT_DISCOUNT;
```

---

# React Standards

These standards apply to `apps/web`, `apps/admin-web`, and the shared rendering layer inside `apps/mobile`, and implement the Frontend Engineering Principles of `ai-docs/02-engineering-principles.md` at the component level.

### Functional Components Only

Every component is a function component using Hooks. Class components are forbidden in new code; any legacy class component encountered during work in a file is migrated as part of that change, not left in place, per the Refactoring Principles of `ai-docs/02-engineering-principles.md`.

### Server vs. Client Components

Where the framework supports a server/client component split, a component defaults to a **Server Component** unless it requires interactivity (event handlers, state, browser-only APIs, effects) — in which case it is explicitly marked a **Client Component** at the smallest possible boundary. Interactivity is pushed as far down the component tree as possible (a single interactive button, not the entire page) so the majority of a screen's markup can be rendered without shipping unnecessary client-side JavaScript, directly serving the Performance-First and Design for the Slowest Device principles in `ai-docs/00-project-vision.md` and `ai-docs/02-engineering-principles.md`.

```tsx
// BookingSummary.tsx — Server Component (default, no directive needed)
export function BookingSummary({ booking }: BookingSummaryProps) {
  return (
    <div>
      <BookingDetails booking={booking} />
      <CancelBookingButton bookingId={booking.id} /> {/* client boundary is here, not the whole page */}
    </div>
  );
}
```

### Hooks Rules

The standard Rules of Hooks are enforced via lint, without exception: Hooks are called only at the top level of a function component or custom Hook, never inside a condition, loop, or nested function. A custom Hook is extracted whenever stateful logic is duplicated across two or more components, or whenever a single component's Hook calls exceed roughly five to seven lines of setup — this is the frontend expression of DRY and SRP from `ai-docs/02-engineering-principles.md`.

### Component Composition

Components are built by composing smaller components, never by adding a boolean prop to branch a single component's rendered output into two unrelated shapes (see Boolean Parameter Abuse in Common Code Smells below). Cross-feature composition happens at the `app/` routing layer per `ai-docs/04-folder-guidelines.md`, never by one feature importing another feature's internals.

```tsx
// Rejected — one component secretly rendering two different UIs
function Card({ isCompact, ...props }: { isCompact: boolean }) {
  return isCompact ? <CompactCardLayout {...props} /> : <FullCardLayout {...props} />;
}

// Required — composition, caller decides which to render
<CompactCard {...props} />
<FullCard {...props} />
```

### Props Design

Props are named for what they represent, not their type (`user`, not `userObject`). A component with more than roughly five to seven props is a signal to group related props into a single object, or to decompose the component, per KISS. Boolean props are named as clear predicates (`isDisabled`, `hasError`) and never default to a value that makes the common case require an explicit `true`/`false` at every call site.

### State Management

State is classified per the State Management Philosophy table in `ai-docs/02-engineering-principles.md` before it is written, not after:

| Category | Where It Lives | Never |
|---|---|---|
| Server/Remote State | The app's data-fetching layer (`services/`), with caching/revalidation | Duplicated into local `useState` "for convenience" |
| UI/Local State | `useState`/`useReducer`, scoped to the component that owns it | Lifted to global state without a cross-component need |
| Global App State | The app's minimal `store/` | Used for anything a single feature could own locally |
| Offline/Draft State | The explicit offline persistence + sync-queue layer | Left in transient component state, risking silent data loss |

### Memoization

`useMemo` and `useCallback` are applied deliberately, based on a measured or clearly predictable rendering cost (a large list, an expensive computation, a prop passed to a memoized child) — never applied reflexively to every value and function as a default habit, which adds cognitive overhead and marginal runtime cost without benefit. `React.memo` is applied to a component only once profiling or a clear re-render pattern justifies it, consistent with Performance Principles being evidence-based, not speculative, per `ai-docs/03-system-architecture-principles.md`'s Evidence over Prediction commitment.

### Error Boundaries

Every route-level and every major independently-failable UI region (e.g., a dashboard widget backed by its own data source) is wrapped in an Error Boundary, so a failure in one region degrades gracefully rather than blanking the entire screen — the frontend expression of Graceful Degradation and Failure Isolation from `ai-docs/03-system-architecture-principles.md`. Error Boundaries render a citizen-safe fallback and report the underlying error to the shared observability pipeline; they never expose a raw stack trace to the citizen.

---

# Backend Standards

These standards implement the four-layer System Layers model and the Module Folder Template from Phases 4 and 5 at the level of individual classes and functions.

### Controllers

A controller's only job is protocol translation: parse the request, call exactly one Application Layer use case, map the result to a response DTO. A controller never contains a conditional expressing a business rule, never calls a repository directly, and never performs a calculation beyond simple response shaping.

```typescript
// booking.controller.ts
export class BookingController {
  constructor(private readonly createBooking: CreateBookingUseCase) {}

  async create(req: Request, res: Response): Promise<void> {
    const dto = CreateBookingRequestDto.parse(req.body); // schema validation at the boundary
    const result = await this.createBooking.execute(dto);
    res.status(201).json(BookingResponseDto.fromDomain(result));
  }
}
```

### Use Cases

A use case is a single, named business operation (`CreateBookingUseCase`, `CancelBookingUseCase`) that orchestrates one or more domain objects and domain services to fulfill it. A use case coordinates; it does not itself contain business *rules* — those live in the domain layer it calls into. One file, one class, one `execute()` method per use case, named for the operation it performs, per the Module Folder Template.

### Services

Two distinct meanings of "service" coexist in Arwal's vocabulary, and engineers are expected to distinguish them precisely: a **Domain Service** (`PricingCalculatorService`) contains real business logic that spans multiple entities within a bounded context; an **Application/Infrastructure Service** (a wrapper around an external SDK, e.g., an SMS-sending client) contains no business logic and exists purely to isolate a technical concern. Conflating the two — putting business rules inside an SDK wrapper, or infrastructure calls inside a domain service — is a Dependency Rule violation per `ai-docs/03-system-architecture-principles.md`.

### Repositories

A repository exposes a small, intention-revealing set of methods (`findById`, `findActiveByProvider`, `save`) — never a generic `query(sql: string)` escape hatch that lets callers bypass the abstraction. The repository interface lives in `domain/repositories/`; its implementation lives in `infrastructure/repositories/`, per the Module Folder Template. A repository never returns a raw database row — it always returns (or accepts) a fully constructed domain entity, mapped via an `infrastructure/mappers/` mapper.

### DTOs

Every value crossing a module's public boundary — an API request/response, an event payload — is an explicit DTO class or type, never the domain entity itself serialized directly. DTOs are named for their direction and purpose (`CreateBookingDto`, `BookingResponseDto`, `BookingConfirmedEventPayload`) and are validated at the point they are constructed from untrusted input.

### Validation

Validation happens in exactly two places, each with a distinct responsibility, and is never conflated:

1. **Schema validation** (types, required fields, formats) — at the Presentation Layer boundary, using the shared validation library, before a request DTO is trusted to exist.
2. **Business-rule validation** (e.g., "a booking cannot be cancelled within 2 hours of the scheduled time") — inside the Domain Layer, as part of the entity or domain service that owns the rule, independent of transport.

```typescript
// Presentation layer — schema validation only
const CreateBookingSchema = z.object({
  providerId: z.string().uuid(),
  scheduledAt: z.string().datetime(),
});

// Domain layer — business-rule validation, owned by the aggregate
class Booking {
  cancel(now: Date): void {
    if (this.hoursUntilScheduled(now) < 2) {
      throw new BookingCancellationWindowExpiredError(this.id);
    }
    this.status = BookingStatus.Cancelled;
  }
}
```

### Dependency Injection

Every class dependency crossing a layer boundary (a use case's repository, a domain service's collaborators) is injected via the constructor — never instantiated with `new` inside the consuming class, and never resolved through a global singleton or service-locator lookup. Constructor injection is what makes Dependency Inversion (`ai-docs/02-engineering-principles.md`) and unit-testability (`ai-docs/03-system-architecture-principles.md`, Technology Independence) actually achievable in code, not just diagrammed.

### Error Handling

Errors are modeled as a typed hierarchy rooted in a shared `AppError` base class (living in `common/errors/`, per `ai-docs/04-folder-guidelines.md`), categorized exactly as defined in the Error Handling table of `ai-docs/02-engineering-principles.md` (Validation, Business Rule, Authorization, Not Found, Infrastructure). Every caught exception is either handled meaningfully (recovered, retried, or translated into a typed domain error) or re-thrown with added context — an empty `catch {}` block is a review-blocking defect, with zero exceptions.

```typescript
// Rejected — silently swallowed
try {
  await paymentGateway.charge(amount);
} catch (e) {
  // nothing
}

// Required — handled or re-thrown with context, never both silently ignored
try {
  await paymentGateway.charge(amount);
} catch (cause) {
  throw new PaymentProcessingError("Failed to charge citizen wallet", { cause, amount });
}
```

### Logging

Every service logs through the shared structured-logging interface in `common/logging/` — never `console.log`, which is forbidden outside of local scratch debugging and is blocked by lint in committed code. Every log statement includes the propagated correlation/trace ID (see `ai-docs/03-system-architecture-principles.md`, Observability Principles) automatically via middleware, never manually threaded through every function signature. Sensitive fields (passwords, tokens, payment instrument numbers, health data) are never passed to a log call, enforced by a shared log-scrubbing utility as a second line of defense.

---

# API Coding Standards

These standards make API-First Design (`ai-docs/03-system-architecture-principles.md`) concrete at the level of an individual endpoint definition.

### REST Naming

Resources are named as plural nouns; actions are expressed through HTTP methods, not verbs in the URL.

| Correct | Incorrect | Why |
|---|---|---|
| `POST /v1/bookings` | `POST /v1/createBooking` | The HTTP method already expresses the action; a verb in the path duplicates it and breaks REST convention. |
| `GET /v1/bookings/:id` | `GET /v1/getBookingById/:id` | Same reasoning; the resource and method fully express intent. |
| `PATCH /v1/bookings/:id/cancel` | `POST /v1/cancelBooking?id=` | A sub-resource action on an existing resource stays nested under it, using a method-appropriate verb suffix only where no pure REST verb applies. |

### Versioning

Every public API is prefixed with an explicit version (`/v1/...`), per the API Versioning principle in `ai-docs/02-engineering-principles.md`. A breaking change (removing a field, changing a field's type or meaning, changing required-ness in an incompatible direction) is never introduced into an existing version; it ships as `/v2/...` with a documented, time-bound deprecation path for `/v1/...`.

### Status Codes

| Code | Meaning at Arwal | Example |
|---|---|---|
| `200 OK` | Successful read or non-creating write | `GET /v1/bookings/:id` |
| `201 Created` | Successful resource creation | `POST /v1/bookings` |
| `204 No Content` | Successful action with no response body | `DELETE /v1/bookings/:id/draft` |
| `400 Bad Request` | Schema/validation error | Missing required field |
| `401 Unauthorized` | Missing or invalid authentication | Expired token |
| `403 Forbidden` | Authenticated, but not authorized for this action/resource | Citizen attempting to access another citizen's booking |
| `404 Not Found` | Referenced resource does not exist | Booking ID not found |
| `409 Conflict` | Business-rule/state conflict | Double-booking a time slot |
| `422 Unprocessable Entity` | Semantically invalid input passing schema but failing a domain rule at the boundary | Cancelling within the 2-hour window |
| `429 Too Many Requests` | Rate limit exceeded | Gateway-enforced throttling |
| `500 Internal Server Error` | Unexpected/infrastructure error | DB timeout — generic message only, per `ai-docs/02-engineering-principles.md` |

A status code is never chosen for convenience (e.g., returning `200` with an error payload) — the HTTP status is always the primary, machine-readable signal of outcome.

### Response Format

Every API response follows one fixed envelope shape, applied consistently across every module, per Convention over Configuration:

```json
{
  "data": { "...": "..." },
  "meta": { "requestId": "...", "timestamp": "..." }
}
```

Error responses follow their own fixed, consistent shape (see Error Responses below), never the success envelope with an error stuffed inside `data`.

### Pagination

List endpoints are always paginated, never returning an unbounded collection. Cursor-based pagination is the default for high-volume, frequently-changing collections (order history, notifications); offset-based pagination is acceptable for small, stable, admin-facing lists. The chosen strategy is documented per endpoint in its API contract, and pagination metadata is always present in the response envelope's `meta`.

```json
{
  "data": [ /* bookings */ ],
  "meta": {
    "pagination": { "nextCursor": "eyJpZCI6IjEyMyJ9", "hasMore": true }
  }
}
```

### Filtering

Filters are expressed as explicit, documented query parameters (`?status=confirmed&providerId=...`), never a free-form query language exposed directly to clients without validation. Every filterable field is backed by a deliberate database index per the Indexing principle in `ai-docs/02-engineering-principles.md` — a filter is never added to a public contract before its query-performance implication is considered.

### Sorting

Sortable fields are explicitly allow-listed per endpoint (`?sort=scheduledAt&order=desc`) — a client can never pass an arbitrary column name that reaches an ORM's `orderBy` unchecked, which would be both a performance risk and an information-disclosure risk (exposing internal column names).

### Error Responses

Every error response follows one fixed shape, consistent with the Error Handling categorization in `ai-docs/02-engineering-principles.md`:

```json
{
  "error": {
    "code": "BOOKING_CANCELLATION_WINDOW_EXPIRED",
    "message": "This booking can no longer be cancelled.",
    "details": [{ "field": "scheduledAt", "issue": "less than 2 hours before start" }],
    "requestId": "..."
  }
}
```

`code` is a stable, machine-readable identifier clients can branch on; `message` is a citizen-safe, human-readable string; `details` is present only for validation-style errors; a raw stack trace or internal exception message is never present in any environment reachable by a client, per the Error Handling table in `ai-docs/02-engineering-principles.md`.

---

# Database Coding Standards

These standards implement the Database Principles of `ai-docs/02-engineering-principles.md` at the level of an individual schema.

### Naming Conventions

| Object | Convention | Example |
|---|---|---|
| Table | `snake_case`, plural | `bookings`, `service_providers` |
| Column | `snake_case` | `scheduled_at`, `provider_id` |
| Primary key | `id` | `id` |
| Foreign key | `<referenced_singular>_id` | `provider_id` referencing `service_providers.id` |
| Index | `idx_<table>_<columns>` | `idx_bookings_provider_id_status` |
| Unique constraint | `uq_<table>_<columns>` | `uq_bookings_provider_id_scheduled_at` |
| Enum-backed column values | `SCREAMING_SNAKE_CASE` or lowercase, fixed per the application-level enum's serialization | Matches the TypeScript string enum's value exactly, never re-cased in the database. |

### Primary Keys

Every table has a single-column primary key, using a UUID (not an auto-incrementing integer) for any table whose ID may ever be referenced externally (in a URL, an API response, an event payload) — this avoids leaking sequential volume information (e.g., total booking count) and avoids collision risk during the district → ward → zone partitioning described in `ai-docs/03-system-architecture-principles.md`.

### Foreign Keys

Every logical reference between tables **within the same module's schema** is enforced with a real foreign-key constraint at the database level, per Data Integrity in `ai-docs/02-engineering-principles.md` — the database is the last line of defense, never trusted to be consistent by application discipline alone. A reference to a row owned by a **different module** is never a foreign key, because that would violate the Data Ownership Principles in `ai-docs/03-system-architecture-principles.md` (no module's schema may constrain against another module's tables); it is instead stored as a plain identifier, validated at the application layer via that module's public API.

### Transactions

Any operation that must succeed or fail as a single atomic unit (e.g., debiting a wallet and recording a transaction) is wrapped in an explicit database transaction — never left as two sequential, independently-committing writes. Transactions are scoped as narrowly as possible around the statements that genuinely need atomicity, never wrapped around an entire request handler "to be safe," which would hold locks longer than necessary and risk contention at scale.

### Migrations

Every schema change ships as a versioned, reviewed migration file, per the Migrations principle in `ai-docs/02-engineering-principles.md` — never a manual change against a live database. Migrations are written to be backward-compatible during rollout (e.g., a new `NOT NULL` column is added nullable first, backfilled, then constrained in a follow-up migration) so a rolling deployment never has a moment where the old code and new schema are incompatible.

### Indexing

An index is added only in response to a real, documented query pattern — never defensively on every column, per `ai-docs/02-engineering-principles.md`. Every migration that adds a new query pattern expected to run at meaningful volume is reviewed for whether it requires a matching index before merge; a slow, unindexed query discovered in production is treated as a defect, not an acceptable trade-off.

### Soft Deletes

Every table backing a domain entity of civic, financial, or trust significance (per the Soft Deletes principle in `ai-docs/02-engineering-principles.md`) includes a `deleted_at TIMESTAMP NULL` column. A "delete" operation on such an entity is always an `UPDATE ... SET deleted_at = now()`, never a `DELETE FROM`. Every repository query against such a table filters `WHERE deleted_at IS NULL` by default, enforced through a shared query-builder convention so no individual query can forget it.

### Audit Fields

Every table includes, at minimum, `created_at TIMESTAMP NOT NULL DEFAULT now()` and `updated_at TIMESTAMP NOT NULL`, maintained automatically (via ORM hook or database trigger, applied consistently across every module) rather than manually set per write. Tables backing sensitive state changes additionally participate in the immutable audit-log mechanism described in `ai-docs/02-engineering-principles.md`, which is a separate, append-only structure from the mutable operational row itself.

---

# Naming Standards

Naming is Convention over Configuration applied to identifiers — it eliminates the single most frequent, most bikeshed-prone micro-decision in daily coding.

| Category | Convention | Example |
|---|---|---|
| **Variables** | `camelCase`, named for what the value represents, never abbreviated ambiguously | `bookingCount`, not `bkCnt` |
| **Booleans (variables/props)** | `camelCase`, prefixed as a predicate (`is`, `has`, `can`, `should`) | `isConfirmed`, `hasActiveSubscription` |
| **Functions** | `camelCase`, verb-first, named for the action performed | `calculatePrice()`, `sendNotification()` |
| **Classes** | `PascalCase`, noun-based | `BookingService`, `PricingCalculator` |
| **Interfaces** | `PascalCase`, no `I` prefix (see `ai-docs/04-folder-guidelines.md`) | `BookingRepository`, not `IBookingRepository` |
| **Types** | `PascalCase`, role-suffixed where it disambiguates | `CreateBookingDto`, `BookingCardProps` |
| **Enums** | `PascalCase` type, `PascalCase` members | `enum BookingStatus { Pending, Confirmed }` |
| **Constants** | `SCREAMING_SNAKE_CASE` for primitives; `PascalCase` for constant objects/enum-like maps | `MAX_BOOKING_LEAD_DAYS`, `BookingStatusLabel` |
| **Files** | Per `ai-docs/04-folder-guidelines.md`'s Naming Conventions table | `pricingCalculator.ts`, `BookingCard.tsx` |
| **Folders** | `kebab-case` | `local-services/`, `booking-history/` |
| **React components** | `PascalCase`, matching the exported component name | `ServiceProviderProfile` |
| **Hooks** | `camelCase`, prefixed `use` | `useBookingAvailability` |
| **Events (Integration Events)** | `PascalCase`, past-tense, suffixed `Event` | `BookingConfirmedEvent`, never `ConfirmBookingEvent` (events describe facts that already happened, never commands) |
| **DTOs** | `PascalCase`, suffixed by direction/role | `CreateBookingDto` (inbound), `BookingResponseDto` (outbound) |

> **Callout — Events Are Past Tense, Commands Are Present Tense**
> A use case is a command: `CreateBookingUseCase` — an instruction to do something, which may fail. An event is a fact: `BookingCreatedEvent` — something that has already, irreversibly happened. Naming an event in the imperative (`CreateBookingEvent`) misrepresents its nature and has caused real confusion in other systems about whether a listener can "reject" it. It cannot — that boundary is enforced by naming.

---

# Commenting Standards

### When Comments Are Required

A comment is required wherever code cannot fully express *why* a decision was made, even though it can fully express *what* the code does:

- A non-obvious business rule whose origin isn't self-evident from the code alone (e.g., "cancellation cutoff is 2 hours per district government SLA, not an arbitrary choice").
- A workaround for a third-party library bug or limitation, including a link to the relevant issue/ticket.
- Any deliberately non-obvious performance optimization, so a future engineer doesn't "simplify" it back into the slow version.
- A `// @ts-expect-error` (see TypeScript Standards) — always with a reason and, wherever possible, a tracking reference.

### When Comments Should Be Avoided

A comment that restates what the next line already says in code is noise, not documentation, and is removed in review:

```typescript
// Rejected
// increment the counter
counter += 1;

// Rejected — the function name already says this
// calculates the price
function calculatePrice() { ... }
```

If a block of code needs a comment to explain *what* it does, the correct fix per Readability Over Cleverness is almost always to extract it into a well-named function, not to leave the code as-is and annotate it.

### TODO and FIXME

`// TODO:` marks intentionally deferred, non-urgent work; `// FIXME:` marks a known defect or shortcut that must be revisited before the surrounding code is considered fully correct. Both are required to include an owner or a tracking reference (an issue ID, an ADR number) — an anonymous, untracked `TODO` is exactly the "silent technical debt" `ai-docs/02-engineering-principles.md` explicitly forbids, and is a review-blocking finding on its own.

```typescript
// TODO(ARWAL-4821): replace with district-configurable cutoff once
// the multi-district config service ships (see ADR-0031).
const CANCELLATION_CUTOFF_HOURS = 2;
```

### ADR References

Any code implementing a decision that was significant enough to warrant an Architectural or Engineering Decision Record (per `ai-docs/02-engineering-principles.md` and `ai-docs/03-system-architecture-principles.md`) includes an inline reference to that ADR's number at the point of implementation — so a future engineer questioning "why is this built this way" is pointed to the recorded reasoning immediately, rather than needing to search for it.

---

# Error Handling Standards

Error handling in code follows directly from the Error Handling principle in `ai-docs/02-engineering-principles.md`, applied at the statement level:

- Every `try/catch` either recovers meaningfully, translates the error into a typed domain error with added context, or is not written at all — letting the error propagate to a layer that can handle it is a valid and often preferred choice over catching prematurely.
- Domain and Application layer code throws typed errors from the shared `AppError` hierarchy (see Backend Standards) — never a bare `throw new Error("something went wrong")`, which is unactionable both for the caller and for the Error Handling table's categorization.
- Presentation-layer code is the **only** layer responsible for translating a thrown domain/application error into an HTTP status code and the standard Error Response shape (see API Coding Standards) — a use case or domain service never constructs an HTTP status code itself, which would violate the Dependency Rules (Domain/Application depending on Presentation concerns).
- Async code always has its rejections handled — an un-awaited promise, or a `.then()` chain with no `.catch()`, is a lint-blocking defect, since an unhandled rejection is functionally identical to a swallowed exception.

---

# Logging Standards

- Every log entry is structured (a JSON object, not an interpolated string), consistent with the Logging principle in `ai-docs/02-engineering-principles.md`, so it is machine-parseable by the shared observability pipeline described in `ai-docs/03-system-architecture-principles.md`.
- Log level is chosen deliberately: `debug` for detail useful only during active investigation; `info` for normal, expected lifecycle events (a booking was created); `warn` for a recovered or degraded-but-handled condition; `error` for a failure requiring attention. `error` is never used for expected, handled business outcomes (e.g., a validation failure is `info` or `warn`, not `error` — it is not a system fault).
- Correlation/trace IDs are never manually passed as a log parameter by application code — they are injected automatically by the shared logging middleware described in `ai-docs/03-system-architecture-principles.md`, so no engineer can forget to include one.
- No sensitive field (password, token, card/UPI details, health record content) is ever passed to a logging call, including inside a larger object — this is enforced by both code review and an automated log-scrubbing safeguard, per `ai-docs/02-engineering-principles.md`'s Secrets Management and Logging sections.

---

# Security Coding Standards

Security coding rules make the Security Principles of `ai-docs/02-engineering-principles.md` and the Security Architecture Principles of `ai-docs/03-system-architecture-principles.md` enforceable in individual functions.

### Input Validation

Every external input — request body, query parameter, path parameter, header, event payload — is validated against an explicit schema before it is used, using the shared validation library. Validation happens once, at the boundary; code deeper in the call stack trusts the now-validated DTO rather than re-checking ad hoc, which would scatter validation logic and violate DRY.

### SQL Injection Prevention

Raw string concatenation into a SQL query is forbidden without exception. Every database access goes through the ORM/query-builder's parameterized query mechanism, or, where raw SQL is genuinely required, through parameterized placeholders only — never through template-literal interpolation of any externally influenced value.

```typescript
// Forbidden
db.query(`SELECT * FROM bookings WHERE provider_id = '${providerId}'`);

// Required
db.query("SELECT * FROM bookings WHERE provider_id = $1", [providerId]);
```

### XSS Prevention

Frontend code never renders unsanitized user-supplied HTML via `dangerouslySetInnerHTML` (React) or an equivalent raw-HTML injection mechanism without passing through the shared, audited sanitization utility first. User-generated text content is rendered as text by default, relying on the framework's automatic escaping, which is the safe default and is never deliberately bypassed for convenience.

### CSRF Protection

Every state-changing request (`POST`, `PATCH`, `PUT`, `DELETE`) from a browser-based client is protected by the shared CSRF-token mechanism wired into the API Gateway and the frontend's data-fetching layer — an individual module or endpoint never opts out of this protection, since a single unprotected mutation endpoint compromises the guarantee for the entire session.

### Authentication

No module implements its own authentication logic. Every protected endpoint relies exclusively on the unified Authentication shared service described in `ai-docs/03-system-architecture-principles.md`, enforced at the API Gateway and re-validated at the module boundary (defense-in-depth, never relying on the gateway alone). Session/token handling (short-lived access tokens, secure refresh-token storage) is implemented once, in the shared Authentication service, and consumed everywhere else — never reimplemented per module.

### Authorization

Every operation that touches another actor's data, or performs a privileged action, includes an explicit authorization check against the acting user's role and resource ownership — implemented at the Application layer of the use case, never assumed from the fact that a request reached the controller. A missing authorization check on a new endpoint is treated with the same severity as a missing test, and is a review-blocking finding.

```typescript
async execute(dto: CancelBookingDto, actor: AuthenticatedActor): Promise<void> {
  const booking = await this.bookingRepository.findById(dto.bookingId);
  if (!booking) throw new BookingNotFoundError(dto.bookingId);
  if (booking.citizenId !== actor.id && !actor.hasRole("admin")) {
    throw new UnauthorizedBookingAccessError(actor.id, booking.id);
  }
  booking.cancel(new Date());
  await this.bookingRepository.save(booking);
}
```

### Secrets Handling

No secret, API key, or credential is ever written into source code, a config file committed to the repository, or a log statement, per `ai-docs/02-engineering-principles.md`'s Secrets Management principle. Secrets are read exclusively through the shared runtime configuration-loading module (`config/`), which sources them from the dedicated secrets-management system — a secret is never read directly from `process.env` scattered through business logic, which would make it impossible to audit where a given secret is actually used.

---

# Performance Coding Standards

### Avoid Unnecessary Renders

Frontend state updates are scoped as narrowly as possible so an update to one piece of state does not cascade into unrelated re-renders. A large, monolithic context provider that every component subscribes to for a single field is avoided in favor of granular state slices, consistent with the State Management Philosophy's rejection of one blunt global mechanism.

### Lazy Loading

Routes and heavy, non-critical-path components are code-split and lazily loaded by default, per Lazy Loading and Code Splitting in `ai-docs/02-engineering-principles.md` — a citizen using only the commerce module never downloads the civic module's bundle until they navigate there.

### Memoization

Applied per the React Standards section above: deliberately, where a measured cost justifies it — never reflexively.

### Efficient Queries

Every database-backed list or detail endpoint is checked for N+1 query patterns before merge — a loop that issues one query per iteration is replaced with a single batched query or an explicit join/`IN` clause, per Database Optimization in `ai-docs/02-engineering-principles.md`. Query plans are reviewed for any endpoint expected to carry significant read load, not guessed at.

### Caching

Any newly introduced cache — at the client, API Gateway, module-application, or cross-module read-model layer described in `ai-docs/03-system-architecture-principles.md` — is not merged without an explicit, documented invalidation strategy defined in the same pull request. A cache without a defined invalidation plan is rejected in review, exactly as `ai-docs/03-system-architecture-principles.md` requires at the architecture level.

---

# Testing Standards

Testing standards implement the Testing Principles and Testing Pyramid of `ai-docs/02-engineering-principles.md` at the level of an individual test file.

### Unit Tests

Unit tests exercise Domain and Application layer logic in complete isolation, with all Infrastructure Layer dependencies (repositories, external SDKs) replaced by test doubles injected through the same constructor-injection mechanism used in production code. A unit test never touches a real database, network call, or filesystem — if it does, it is an integration test and belongs in `tests/integration/`, not co-located as a unit test.

### Integration Tests

Integration tests exercise a module's real interaction with its actual dependencies (a real test database, a real cache) to verify the seams the unit tests intentionally mock out. They live in the module's own `tests/integration/` folder per `ai-docs/04-folder-guidelines.md`, are required for any cross-boundary change, and are run against an isolated, disposable test environment — never against shared or production infrastructure.

### End-to-End (E2E) Tests

E2E tests exercise a complete citizen-facing journey (checkout, booking, application submission) across the full stack, from the client through the API Gateway to the database and back. They are deliberately few, curated, and reserved for the flows that matter most to citizen trust, consistent with the Testing Pyramid's shape — E2E tests are the most expensive to write and maintain, and are never used as a substitute for adequate unit or integration coverage lower in the pyramid.

### Test Naming

Test names describe the behavior under test as a full sentence, not as an abbreviated label, so a failing test's name alone communicates what broke:

```typescript
describe("Booking.cancel", () => {
  it("throws BookingCancellationWindowExpiredError when cancelled less than 2 hours before the scheduled time", () => {
    // ...
  });

  it("transitions status to Cancelled when cancelled more than 2 hours before the scheduled time", () => {
    // ...
  });
});
```

### Test Organization

Tests are co-located with the code they test, per the Testing Folder Structure in `ai-docs/04-folder-guidelines.md` — `Booking.ts` next to `Booking.test.ts`. Each test is independent and can run in any order or in isolation; shared setup lives in explicit `beforeEach`/fixture helpers, never in mutable module-level state that could leak between tests. A flaky test is fixed or deleted immediately upon discovery, per `ai-docs/02-engineering-principles.md` — it is never left "quarantined" indefinitely.

---

# Code Review Standards

These standards extend the Code Review Standards of `ai-docs/02-engineering-principles.md` into a concrete, line-level checklist reviewers apply to every pull request.

### Review Checklist

- [ ] Does the change do what it claims, including edge cases and failure modes?
- [ ] Does it honor SOLID, DRY, KISS, and YAGNI as applied in this document, not just in spirit?
- [ ] Are all types explicit at public boundaries, with no unjustified `any`?
- [ ] Are Domain, Application, Infrastructure, and Presentation responsibilities correctly separated, per the Backend Standards above?
- [ ] Is every external input validated at the boundary?
- [ ] Is every error either handled meaningfully or re-thrown with context — no empty `catch` blocks?
- [ ] Is authorization explicitly checked for any operation touching another actor's data?
- [ ] Are naming conventions from this document followed consistently?
- [ ] Is test coverage present at the appropriate levels of the Testing Pyramid for this change's risk profile?
- [ ] Are there any of the Common Code Smells listed below present in the diff?

### Blocking Issues

The following are always merge-blocking, regardless of deadline pressure: a missing or bypassed authorization check; a swallowed exception; a raw SQL string built via concatenation of external input; a secret or credential present in the diff; a missing test for new business-rule logic; a forbidden cross-module or cross-layer import per `ai-docs/03-system-architecture-principles.md` and `ai-docs/04-folder-guidelines.md`; a breaking API change without a version bump.

### Minor Issues

Naming inconsistencies, missing but non-critical comments, a slightly-too-long function that doesn't yet warrant a blocking objection, or a stylistic preference not covered by an explicit rule in this document are raised as non-blocking suggestions — the author may address them in the same PR or track them as follow-up technical debt per `ai-docs/02-engineering-principles.md`, at the reviewer's and author's shared judgment.

### Merge Requirements

A pull request merges only when: all Blocking Issues are resolved, CI (build, lint, type-check, test suite) passes, at least one qualified reviewer has approved, and — for any change touching a shared boundary (a module's `index.ts`, an Integration Event schema, this document's own rules) — the relevant owning team has explicitly reviewed, per the Folder Ownership Rules in `ai-docs/04-folder-guidelines.md`.

---

# Refactoring Standards

Refactoring code follows the Refactoring Principles of `ai-docs/02-engineering-principles.md`, applied concretely:

- A refactor is committed separately from a behavioral change whenever both are needed in the same area of code, so a reviewer can verify "structure changed, behavior didn't" independently of "behavior changed as intended."
- A refactor is covered by passing tests **before** it begins — refactoring code with no test coverage is preceded by writing characterization tests first, never performed on trust alone.
- A refactor that touches a module's public `index.ts` surface, or a shared/common utility consumed by multiple modules, requires the same review rigor as a new public API change, since it can silently break consumers outside the file being edited.

---

# Technical Debt Policy

Consistent with the Technical Debt Policy in `ai-docs/02-engineering-principles.md`, any deliberate shortcut in code is marked inline with a tracked `TODO`/`FIXME` (see Commenting Standards above) at the moment it is introduced — never left unmarked in the hope it will be remembered later. Code review is expected to catch and require this marking as a condition of merge; an unmarked, undocumented shortcut discovered later is treated as a process failure, not merely a code issue.

---

# Common Code Smells

The following patterns are explicitly called out because they recur across large, long-lived codebases and are cheaper to name and reject early than to unwind at Phase 200.

| Smell | Description | Why It's Rejected |
|---|---|---|
| **Long Methods** | A function that exceeds roughly 30–40 lines, or that requires scrolling to read in full. | Almost always doing more than one job; violates SRP and makes the function hard to test in isolation. |
| **God Classes** | A class that knows about and manipulates far more of the system than its single responsibility justifies (e.g., a `BookingManager` that also sends notifications and calculates pricing). | Violates SRP and Domain Boundaries; becomes the file every engineer is afraid to touch, exactly the failure mode `ai-docs/03-system-architecture-principles.md`'s Common Anti-Patterns table warns against. |
| **Duplicate Logic** | The same business decision expressed in two or more places that must be kept manually in sync. | A DRY violation per `ai-docs/02-engineering-principles.md`; the two copies inevitably drift, producing silent inconsistency (e.g., two different cancellation cutoffs). |
| **Magic Numbers** | An unexplained literal value embedded directly in logic (`if (hours < 2)`). | Undocumented, unsearchable, and easy to duplicate incorrectly; replaced with a named constant (`CANCELLATION_CUTOFF_HOURS`) that documents itself and has exactly one authoritative definition. |
| **Deep Nesting** | Conditional/loop nesting beyond roughly 3 levels. | Hard to read and reason about; usually resolved with early returns/guard clauses or by extracting the nested block into its own named function. |
| **Boolean Parameter Abuse** | A function whose behavior branches significantly based on a boolean flag (`renderCard(compact: boolean)`), especially multiple such flags stacked together. | The call site becomes unreadable (`renderCard(true, false, true)`), and the function is secretly doing two jobs — split into two functions/components instead, per Composition over Inheritance. |
| **Large Interfaces** | An interface with many unrelated methods, forcing every implementer to support all of them. | Violates the Interface Segregation Principle; split into smaller, role-specific interfaces. |
| **Tight Coupling** | Code that reaches directly into another module's internals, or that cannot be tested without instantiating unrelated parts of the system. | Defeats the Modular Monolith strategy and the entire purpose of the Module Folder Template's `index.ts` boundary, per `ai-docs/03-system-architecture-principles.md` and `ai-docs/04-folder-guidelines.md`. |

---

# Engineering Excellence Checklist

Before any code is considered ready for review, it satisfies the following, drawn together from every section above:

- [ ] Types are explicit at every public boundary; `any` is absent or narrowly justified.
- [ ] Every function has a single, describable responsibility.
- [ ] No business logic exists in a controller, a React component's render body, or an infrastructure wrapper.
- [ ] Every external input is validated at the boundary before use.
- [ ] Every error path is handled or explicitly re-thrown with context — nothing is silently swallowed.
- [ ] Every operation touching another actor's data has an explicit authorization check.
- [ ] No raw SQL string concatenation; no unsanitized HTML injection; no secret in source or logs.
- [ ] Naming follows the conventions in this document, without exception or local preference.
- [ ] Tests exist at the appropriate level(s) of the Testing Pyramid, named descriptively, co-located per `ai-docs/04-folder-guidelines.md`.
- [ ] No Common Code Smell from the table above is present without a justified, reviewed exception.
- [ ] Any deliberate shortcut is marked as tracked technical debt, per the Technical Debt Policy.
- [ ] The change is consistent with every architectural and folder boundary defined in Phases 3, 4, and 5 — nothing here contradicts them.

---

# Closing Statement

> **Callout — Closing Statement**
> Architecture defines the shape of the system; folder guidelines define where that shape lives on disk; coding standards define what fills it, keystroke by keystroke, for every one of the ~300 micro-phases still ahead. A citizen's booking, payment, or government application is only as trustworthy as the least-disciplined function in the code path that serves it — this document exists so that discipline is never a matter of individual mood or deadline pressure, but a shared, citable, enforced standard every engineer inherits the moment they open the repository. Where a future phase must deviate from a rule stated here, that deviation is made explicitly — through a code review exception with recorded reasoning, or an ADR where the deviation is structural — never silently, and never by default.

This document, `ai-docs/05-coding-standards.md`, is the sixth phase of approximately 300. Every function, component, endpoint, and query written in the phases that follow is expected to conform to the standards defined here, or to justify its deviation in writing.

**End of Phase 6 — `ai-docs/05-coding-standards.md`**
