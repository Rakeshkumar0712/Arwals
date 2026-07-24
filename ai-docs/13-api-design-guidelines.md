# API Design Guidelines

**Document:** `ai-docs/13-api-design-guidelines.md`
**Project:** Arwal — The District Super App
**Stage:** 1 — Foundation
**Phase:** 14 — API Design Guidelines
**Status:** Approved for Engineering Reference
**Audience:** Architects, Backend Engineers, Frontend Engineers, Mobile Engineers, QA Engineers, Technical Reviewers, Government Technical Partners, Third-Party Integration Partners

> **Callout — Purpose of This Document**
> `ai-docs/00-project-vision.md` defined why Arwal exists. `ai-docs/01-product-goals.md` defined what Arwal must achieve. `ai-docs/02-engineering-principles.md` defined how engineers reason about design. `ai-docs/03-system-architecture-principles.md` defined how the system is structured. `ai-docs/04-folder-guidelines.md` defined where that structure physically lives. `ai-docs/05-coding-standards.md` defined how individual lines of code are written. `ai-docs/06-git-workflow.md` defined how change moves through version control. `ai-docs/07-development-workflow.md` defined how a day of engineering work unfolds. `ai-docs/08-definition-of-done.md` defined when a piece of work is actually finished. `ai-docs/09-tech-stack.md` named the concrete technologies everything is built from. `ai-docs/10-security-standards.md` defined the enforceable security standard those technologies must satisfy. `ai-docs/11-performance-standards.md` defined the enforceable performance standard those technologies must satisfy. `ai-docs/12-accessibility-standards.md` defined the enforceable accessibility standard every screen must satisfy. This document defines **the enforceable API design standard** — the specific, citable rules that govern every endpoint Arwal will ever expose, across `apps/api`, consumed by `apps/web`, `apps/admin-web`, `apps/mobile`, and, eventually, trusted third parties, for every one of the ~300 micro-phases still ahead.

---

# Purpose of this Document

Every phase document preceding this one touches the API surface without fully specifying it. `ai-docs/00-project-vision.md` commits Arwal to an API-first, cloud-native architecture serving PWA, Android, iOS, and a future third-party ecosystem from a single backend contract. `ai-docs/01-product-goals.md` names API-First, Contract-Driven Development as a Technical Goal and Platform Parity Across Surfaces as a Non-Functional Goal. `ai-docs/02-engineering-principles.md` establishes API-First Design, DTO Usage, API Versioning, and a first pass at status codes and error categorization as Backend Engineering Principles. `ai-docs/03-system-architecture-principles.md` defines the API Gateway Philosophy, the System Layers a request travels through (Presentation → Application → Domain → Infrastructure), and the distinction between synchronous API calls and asynchronous Integration Events. `ai-docs/04-folder-guidelines.md` gives every module's public surface a physical home (`presentation/controllers`, `presentation/dto`, `index.ts`). `ai-docs/05-coding-standards.md` gives line-level REST naming, status code, response envelope, pagination, filtering, sorting, and error-response rules. `ai-docs/06-git-workflow.md` and `ai-docs/07-development-workflow.md` define how an API contract moves from draft to production, including the API Change Workflow. `ai-docs/08-definition-of-done.md` makes API contract completeness, versioning, and documentation a non-negotiable exit gate. `ai-docs/09-tech-stack.md` names NestJS, Prisma, PostgreSQL, REST, OpenAPI, and JWT as the concrete technologies the API is built from. `ai-docs/10-security-standards.md` defines the enforceable API Security Standards — authentication, authorization, rate limiting, idempotency, and the OWASP API risk mapping. `ai-docs/11-performance-standards.md` defines the enforceable API latency, payload, and pagination performance targets. `ai-docs/12-accessibility-standards.md` establishes that every citizen-facing surface, including the data an API returns, must ultimately support an accessible experience regardless of device or network.

What none of those documents does — because it is not their job to — is define, in one place, **the complete, specific, citable API design standard itself**: exactly what a URI should look like, exactly which HTTP method a given operation should use, exactly what a paginated list response's envelope contains, and exactly how a breaking change is proposed, reviewed, and retired. An API design principle repeated across twelve documents but never assembled into one canonical reference is not an API standard — it is a set of hints an engineer must reconstruct from memory, which guarantees drift the moment two engineers reconstruct it differently.

This document exists to:

1. **Consolidate every API-relevant principle scattered across Phases 1–13 into one authoritative, standalone reference** — the document a backend engineer, frontend integrator, or third-party partner opens first, and the document every other phase document's API references ultimately resolve to.
2. **Give every engineer, reviewer, and integration partner a single, citable API standard** — "this violates the URI Design Standards in Phase 14" is exactly as legitimate and actionable a review comment as citing SOLID from Phase 3 or a security control from Phase 11.
3. **Protect Platform Parity across PWA, Android, iOS, and Admin** (`ai-docs/01-product-goals.md`) by ensuring every client consumes the exact same, precisely specified contract — never a contract inferred by trial and error against whatever the backend currently happens to return.
4. **Make API correctness testable, not subjective.** Every rule in this document resolves to something a linter, a contract test, or a reviewer can check mechanically — a resource name, a status code, an envelope shape — never a vague aspiration like "the API should feel intuitive."
5. **Serve as the binding reference for API design review, OpenAPI generation, contract testing, and third-party integration** for the entire life of the ~300-phase roadmap, revisited and amended only through the same Architectural Decision Record discipline established in `ai-docs/02-engineering-principles.md` and `ai-docs/03-system-architecture-principles.md`.

This document assumes and requires familiarity with all thirteen preceding phase documents. It does not re-argue their reasoning — it is where that reasoning becomes a specific, enforceable API design rule.

---

# API Design Philosophy

Arwal's API design rests on seven commitments. Together they answer the question every subsequent section of this document exists to make concrete: **what does a well-designed API actually require, by default, before a single endpoint is implemented?**

### Resource-Oriented APIs

Arwal's domain — bookings, orders, applications, wallets, listings — is composed of clearly identifiable things a citizen, merchant, or government officer creates, reads, updates, and removes. REST's resource/verb model, per the API Style decision in `ai-docs/09-tech-stack.md`, maps directly onto this domain shape: a URI names a resource (a noun), and an HTTP method expresses the action performed on it (a verb). An API organized around resources, not around procedures ("createBooking", "getBookingById"), is discoverable by convention — an engineer who understands `/v1/bookings` immediately understands `/v1/applications` without re-learning a naming scheme, directly extending the Consistency Over Local Preference principle in `ai-docs/04-folder-guidelines.md` to the API surface.

### Consistency

The same category of design decision is resolved identically, every time, across every module — the same envelope shape, the same pagination pattern, the same error format, the same naming convention, regardless of which team or engineer builds the endpoint. Consistency is what lets a frontend engineer who has integrated the Commerce module integrate the Civic Services module without re-learning a new contract shape, exactly as the Naming Standards in `ai-docs/05-coding-standards.md` and `ai-docs/04-folder-guidelines.md` already require at the code and folder level.

### Predictability

An API is predictable when a consumer can correctly guess the shape of an endpoint they have never used, based on endpoints they have. `POST /v1/bookings` returning `201 Created` with the created resource predicts that `POST /v1/applications` behaves the same way. Predictability reduces integration time, reduces defect rate at integration boundaries, and is the API-level expression of Convention over Configuration (`ai-docs/02-engineering-principles.md`).

### Backward Compatibility

Per the API Versioning principle in `ai-docs/02-engineering-principles.md` and the Platform Parity commitment in `ai-docs/01-product-goals.md`, a contract already shipped to a client is never broken out from under it. PWA, Android, and iOS release on independent cadences; an Android client built against `/v1/bookings` six months ago must still work today, unless it has been given an explicit, time-bound migration path to a new version. Backward compatibility is not a courtesy — it is a structural requirement of a platform with more than one client surface.

### Security by Default

Every endpoint starts authenticated, authorized, validated, and rate-limited — opened up only as far as a real requirement demands, per Secure by Default (`ai-docs/02-engineering-principles.md`) and the Security Philosophy in `ai-docs/10-security-standards.md`. An API is Arwal's largest attack surface, per the Threat Model in `ai-docs/10-security-standards.md`, and this document's rules exist, in large part, to close that surface by design rather than by review-time discovery.

### Performance by Default

Every endpoint is designed against the latency, payload, and pagination targets in `ai-docs/11-performance-standards.md` from its first draft, not tuned reactively after a citizen on 3G experiences it as slow. Pagination, field selection, and compression are not optional add-ons — they are the default shape of any endpoint capable of returning more than a small, bounded amount of data.

### Developer Experience

An API is a product in its own right, consumed by Arwal's own frontend and mobile teams today and by trusted third parties in the Open Ecosystem Phase (`ai-docs/01-product-goals.md`) tomorrow. Clear naming, complete OpenAPI documentation (`ai-docs/09-tech-stack.md`), consistent error messages, and predictable pagination are not cosmetic polish — they are what makes the difference between an integration that takes an afternoon and one that takes a week of guesswork against undocumented behavior.

```mermaid
graph TD
    A[Resource-Oriented] --> G[Arwal API Design Philosophy]
    B[Consistency] --> G
    C[Predictability] --> G
    D[Backward Compatibility] --> G
    E[Security by Default] --> G
    F[Performance by Default] --> G
    H[Developer Experience] --> G
    G --> I[Every Endpoint, Every Module,<br/>Every Client Surface, Same Contract Discipline]
```

> **Callout — The One-Sentence API Philosophy**
> *"An API is a promise to every client that will ever call it — design it as something a stranger, a teammate, and a future government partner could all integrate against correctly on the first try, without asking you a single question."*

---

# API Versioning

Versioning is what makes Backward Compatibility (above) achievable in practice rather than aspirational — it is the mechanism by which Arwal evolves its contract without breaking a client that has not yet migrated.

### URI Versioning

Every public API is prefixed with an explicit major version segment (`/v1/...`), per the API Versioning principle in `ai-docs/02-engineering-principles.md` and the Versioning standard in `ai-docs/05-coding-standards.md`. URI versioning — rather than a header-based or content-negotiation-based scheme — is chosen because it is the most discoverable, most cache-friendly, and least error-prone scheme for a platform with client teams (PWA, Android, iOS) that release independently and cannot be assumed to correctly set a custom version header on every request.

```
https://api.arwal.in/v1/bookings
https://api.arwal.in/v2/bookings   ← only introduced when v1 cannot serve a breaking change
```

### What Constitutes a Breaking Change

| Change | Breaking? | Reasoning |
|---|---|---|
| Adding a new optional field to a response | No | Existing clients that don't know about the field simply ignore it. |
| Adding a new optional query parameter | No | Existing clients that don't send it get current default behavior. |
| Adding a new endpoint | No | No existing client is calling it yet. |
| Removing a field from a response | Yes | Any client reading that field now fails or silently misbehaves. |
| Renaming a field | Yes | Functionally identical to removal plus addition — the old name disappears. |
| Changing a field's type (`string` → `number`) | Yes | A client's parsing logic built against the old type breaks. |
| Changing a field from optional to required in a **request** | Yes | An existing client not sending that field now fails validation. |
| Changing a success status code (`200` → `202`) | Yes | Client logic branching on status code breaks. |
| Changing pagination style (offset → cursor) | Yes | Client pagination logic is fundamentally incompatible. |
| Tightening a validation rule that rejects previously-accepted input | Yes | A client that previously succeeded now fails. |
| Loosening a validation rule | No | A client that previously succeeded still succeeds. |

### Deprecation

A version is never retired without a documented deprecation period. Deprecation is announced through three simultaneous channels: (1) a `Deprecation` HTTP response header on every response from the deprecated version, (2) an entry in the OpenAPI specification marking the endpoint or version `deprecated: true`, and (3) direct communication to every consuming client team (PWA, Android, iOS, Admin, and any registered third party), per the API Change Workflow in `ai-docs/07-development-workflow.md`.

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Wed, 31 Dec 2026 23:59:59 GMT
Link: <https://docs.arwal.in/api/migration/v1-to-v2-bookings>; rel="successor-version"
```

### Sunset Policy

A deprecated version's minimum supported lifetime is **180 days** from the date deprecation is announced, or longer where telemetry shows meaningful client traffic remains on it — the version is never retired purely because the calendar date has passed if consuming clients have not yet migrated, consistent with Transparency over Opacity (`ai-docs/00-project-vision.md`). The `Sunset` HTTP header (per the IETF Sunset header draft convention) communicates the exact retirement date. Retirement proceeds only after: the deprecation window has closed, telemetry confirms traffic on the old version has fallen to a negligible, explicitly-agreed threshold, and every internally-known consuming client (PWA, Android, iOS, Admin) has confirmed migration.

### Compatibility Within a Version

Within a single version (`/v1/...`), only non-breaking, additive changes are permitted, per the table above. A version is a promise: anything that was true about `/v1/bookings`'s contract on the day a client integrated against it remains true for the entire life of that version.

```mermaid
graph TD
    A[Change Proposed] --> B{Breaking per<br/>the table above?}
    B -->|No| C[Ship within current version<br/>after standard review]
    B -->|Yes| D[New version: /v2/...]
    D --> E[Old version marked deprecated:<br/>Deprecation header, OpenAPI flag]
    E --> F[180-day minimum sunset window,<br/>client teams notified]
    F --> G[Telemetry confirms traffic<br/>has migrated]
    G --> H[Old version retired]
```

---

# URI Design

### Naming Conventions

URIs are lowercase, `kebab-case` for multi-word path segments, and never contain verbs, file extensions, or implementation detail (`.php`, `.json` in the path itself belongs in the `Accept`/`Content-Type` header, never the URI).

| Correct | Incorrect | Why |
|---|---|---|
| `/v1/service-providers` | `/v1/serviceProviders` | `kebab-case` matches the folder-naming convention in `ai-docs/04-folder-guidelines.md` and avoids case-sensitivity ambiguity across clients. |
| `/v1/bookings` | `/v1/booking` | Resources are always plural (see below), regardless of collection size. |
| `/v1/bookings/:id/cancel` | `/v1/cancelBooking` | Actions are sub-resources or method-appropriate verbs, never a verb baked into the path root. |

### Plural Resources

Every resource collection is named as a plural noun, without exception — `/v1/bookings`, not `/v1/booking`; `/v1/applications`, not `/v1/application`. This removes an entire category of "is it singular or plural here" decision an engineer would otherwise re-litigate per endpoint, per Convention over Configuration (`ai-docs/02-engineering-principles.md`).

### Resource Identifiers

Every individual resource is addressed by its unique identifier as a path segment: `/v1/bookings/:bookingId`. Per the Primary Keys standard in `ai-docs/05-coding-standards.md`, every externally-addressable identifier is a UUID, never a sequential integer — this prevents leaking volume information (total booking count inferred from an incrementing ID) and avoids collision risk under the district → ward → zone partitioning strategy (`ai-docs/03-system-architecture-principles.md`).

```
GET /v1/bookings/8f14e45f-ceea-4e9c-8b2a-1a3c4d5e6f7a
```

### Nested Resources

A resource is nested under its parent only when it is genuinely owned by and cannot meaningfully exist independent of that parent — a booking's line items, for example, are always accessed through their owning booking.

```
GET /v1/bookings/:bookingId/line-items
GET /v1/bookings/:bookingId/line-items/:lineItemId
```

Nesting is limited to **one level** wherever practical. A URI requiring three or more nested segments (`/v1/districts/:id/wards/:id/zones/:id/bookings`) is a signal that the deeper resource should instead be addressed directly with its parent's ID as a filter, per the Deep Nesting anti-pattern already rejected for folders in `ai-docs/04-folder-guidelines.md`, applied here to URIs:

```
# Rejected — excessive nesting
GET /v1/districts/:districtId/wards/:wardId/zones/:zoneId/bookings

# Required — flat resource, parent as a filter
GET /v1/bookings?zoneId=:zoneId
```

### Cross-Module Resource References

A resource belonging to one module never appears nested under another module's URI root, per the Data Ownership Principles in `ai-docs/03-system-architecture-principles.md` — `/v1/bookings/:id/citizen` is rejected in favor of the Commerce/Local Services module returning a `citizenId` reference, with the Identity module's own `/v1/citizens/:id` endpoint as the authoritative source, consistent with the Single Source of Truth principle applied at the API layer.

### Actions on Resources

An operation that is not a pure CRUD action on a resource (cancelling a booking, confirming a payment, approving an application) is expressed as a sub-resource action, using a noun-based sub-path with a `POST`, never a verb baked into the collection root:

```
PATCH /v1/bookings/:id/cancel
POST  /v1/applications/:id/submit
POST  /v1/payments/:id/refund
```

| Correct | Incorrect | Why |
|---|---|---|
| `PATCH /v1/bookings/:id/cancel` | `POST /v1/cancelBooking?id=` | The resource stays the URI root; the action is a sub-resource, never a top-level verb-named endpoint. |
| `POST /v1/applications/:id/submit` | `PUT /v1/applications/:id?action=submit` | An action-as-query-parameter obscures intent and is not cacheable/loggeable the same way a distinct path is. |

---

# HTTP Methods

Every HTTP method has exactly one meaning at Arwal, applied consistently across every module, per the API Coding Standards in `ai-docs/05-coding-standards.md`.

```mermaid
graph LR
    A[GET<br/>Read, safe, idempotent] --> E[Resource]
    B[POST<br/>Create, or non-idempotent action] --> E
    C[PUT<br/>Full replace, idempotent] --> E
    D[PATCH<br/>Partial update] --> E
    F[DELETE<br/>Soft-delete, idempotent] --> E
```

| Method | Use For | Idempotent? | Safe? | Example |
|---|---|---|---|---|
| **GET** | Retrieving a resource or collection. Never mutates state. | Yes | Yes | `GET /v1/bookings/:id` |
| **POST** | Creating a new resource, or invoking a non-idempotent action/process. | No (unless an idempotency key is supplied — see Request Design) | No | `POST /v1/bookings` |
| **PUT** | Replacing a resource in full — every field the resource owns is supplied, and any field omitted is reset to its default/null. | Yes | No | `PUT /v1/service-providers/:id/profile` |
| **PATCH** | Updating a resource partially — only the supplied fields change; omitted fields are left untouched. | Yes (for a well-formed patch) | No | `PATCH /v1/bookings/:id` |
| **DELETE** | Removing a resource. Per the Soft Deletes principle (`ai-docs/02-engineering-principles.md`), this is always a soft delete at the persistence layer for any entity of civic, financial, or trust significance — the API contract still returns `204 No Content` as if the resource were gone. | Yes | No | `DELETE /v1/addresses/:id` |

### When to Use PATCH vs. PUT

Arwal defaults to **PATCH** for the overwhelming majority of update operations, because most real-world updates are partial ("cancel this booking," "update this phone number") and a client should never be forced to resend an entire resource just to change one field — resending a full resource on every partial intent risks a stale-data overwrite (a client's cached copy of unrelated fields silently reverting fields it didn't intend to touch). **PUT** is reserved for the rare case where a resource is genuinely and deliberately replaced wholesale (e.g., overwriting an entire configuration document).

```typescript
// PATCH — only the supplied fields change
// PATCH /v1/bookings/:id
{ "scheduledAt": "2026-08-01T10:00:00Z" }

// PUT — the entire resource is replaced; omitted fields reset
// PUT /v1/service-providers/:id/profile
{ "displayName": "...", "bio": "...", "serviceCategories": [...], "availability": {...} }
```

### Method Safety and Idempotency Guarantees

- **GET** never has a side effect. A `GET` request that changes application state (e.g., "mark as read" via a `GET`) is a Blocking Issue in review, since it violates HTTP semantics that caches, browsers, prefetchers, and retry logic all rely on.
- **PUT** and **DELETE** are idempotent by definition — issuing the same request twice produces the same end state as issuing it once, and no additional idempotency-key mechanism is required for them.
- **POST** is not idempotent by default; where a `POST` represents a state-mutating operation reachable via client retry (booking creation, payment initiation), an idempotency key is required — see Request Design below.

---

# Status Codes

Arwal uses a fixed, documented subset of HTTP status codes, applied consistently, extending the Status Codes table in `ai-docs/05-coding-standards.md` with full usage guidance.

| Code | Meaning | When Used |
|---|---|---|
| `200 OK` | Successful read or a non-creating write that returns a body | `GET`, successful `PATCH`/`PUT` returning the updated resource |
| `201 Created` | A new resource was successfully created | `POST /v1/bookings` — response includes the created resource and a `Location` header |
| `202 Accepted` | The request was accepted for asynchronous processing; the result is not yet final | A civic application submission queued for downstream government-system processing |
| `204 No Content` | The request succeeded and there is no response body | `DELETE`, or a `POST` action with no meaningful return value |
| `400 Bad Request` | The request is malformed or fails schema validation | Missing required field, malformed JSON, invalid enum value |
| `401 Unauthorized` | The request has no, or an invalid/expired, authentication credential | Missing/expired JWT |
| `403 Forbidden` | The request is authenticated, but the actor is not authorized for this action/resource | A citizen attempting to cancel another citizen's booking |
| `404 Not Found` | The referenced resource does not exist, or exists but the actor has no right to know it exists | A booking ID that doesn't exist, or belongs to another citizen (see callout below) |
| `409 Conflict` | The request conflicts with the resource's current state | Double-booking an already-taken time slot |
| `410 Gone` | The resource existed but has been permanently, deliberately removed | A retired API version or a hard-deleted, non-recoverable resource |
| `422 Unprocessable Entity` | The request is well-formed and passes schema validation, but fails a domain/business rule | Cancelling within the 2-hour cancellation window |
| `429 Too Many Requests` | The actor has exceeded a rate limit | Login attempts, OTP requests, API abuse |
| `500 Internal Server Error` | An unexpected, unhandled server-side fault | A database timeout, an unhandled exception — generic message only |
| `503 Service Unavailable` | The service is temporarily unable to handle the request | Planned maintenance, or a dependency circuit breaker is open |

> **Callout — 403 vs. 404 for Resource-Ownership Failures**
> Where disclosing that a resource *exists* would itself leak information (e.g., confirming another citizen has a booking with a specific ID), Arwal returns `404 Not Found` rather than `403 Forbidden`, per the Broken Access Control mitigation in `ai-docs/10-security-standards.md` — a `403` confirms existence to an attacker probing IDs; a `404` reveals nothing. This choice is made explicitly per endpoint based on its data sensitivity tier (`ai-docs/10-security-standards.md`), not applied blindly everywhere `403` would otherwise be technically correct.

### Status Code Decision Flow

```mermaid
graph TD
    A[Request Received] --> B{Authenticated?}
    B -->|No| C[401]
    B -->|Yes| D{Schema Valid?}
    D -->|No| E[400]
    D -->|Yes| F{Authorized for<br/>this resource/action?}
    F -->|No, and existence<br/>should be hidden| G[404]
    F -->|No, existence<br/>can be disclosed| H[403]
    F -->|Yes| I{Business Rule Passes?}
    I -->|No| J[422 or 409]
    I -->|Yes| K{Sync or Async?}
    K -->|Sync, resource created| L[201]
    K -->|Sync, no body| M[204]
    K -->|Sync, body returned| N[200]
    K -->|Async, queued| O[202]
```

---

# Request Design

### Headers

Every request carries a fixed set of standard headers, applied consistently across every endpoint:

| Header | Purpose | Required? |
|---|---|---|
| `Authorization: Bearer <JWT>` | Authentication credential, per `ai-docs/10-security-standards.md` | Yes, for every protected endpoint |
| `Content-Type: application/json` | Declares the request body format | Yes, for any request with a body |
| `Accept: application/json` | Declares the expected response format | Recommended |
| `Idempotency-Key: <uuid>` | Ensures a retried request produces a single effect | Required for `POST` operations that mutate state and are reachable via client retry |
| `X-Correlation-Id: <uuid>` | Traces a single citizen action across every module and event it touches | Auto-injected at the API Gateway if absent, per the Observability Principles in `ai-docs/03-system-architecture-principles.md` |
| `Accept-Language` | Signals the citizen's preferred language, driving `packages/i18n` response localization (`ai-docs/04-folder-guidelines.md`) | Recommended |

### Validation

Every request is validated in exactly two places, never conflated, exactly as established in `ai-docs/05-coding-standards.md`: schema validation (types, required fields, formats) at the Presentation Layer via Zod (`ai-docs/09-tech-stack.md`), and business-rule validation inside the Domain Layer. An API request is never trusted past the Presentation Layer boundary without having passed schema validation first.

### DTOs

Every request body and response body is an explicit DTO, never a raw domain entity serialized directly, per the DTO Usage principle in `ai-docs/02-engineering-principles.md` and `ai-docs/05-coding-standards.md`. A request DTO also enforces the Mass Assignment defense from `ai-docs/10-security-standards.md`: unexpected fields are rejected, not silently ignored, so a client can never smuggle a server-controlled field (a role, a price, a status) into a request payload.

```typescript
// CreateBookingRequestDto — presentation/dto/CreateBookingRequestDto.ts
const CreateBookingRequestSchema = z
  .object({
    providerId: z.string().uuid(),
    scheduledAt: z.string().datetime(),
    notes: z.string().max(500).optional(),
  })
  .strict(); // rejects unrecognized fields — closes Mass Assignment
```

### Idempotency Keys

Every state-mutating `POST` operation reachable via client retry (booking creation, payment initiation, government application submission) requires an `Idempotency-Key` header, per the Idempotency resilience pattern in `ai-docs/03-system-architecture-principles.md` and the API Security Standards in `ai-docs/10-security-standards.md`. The server stores the key alongside the operation's result for a bounded retention window (24 hours, sufficient to absorb realistic client retry/backoff timing); a repeated request with the same key returns the original result without re-executing the operation.

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Store as Idempotency Key Store (Redis)

    Client->>API: POST /v1/bookings, Idempotency-Key: abc-123
    API->>Store: Key seen before?
    Store-->>API: No
    API->>API: Execute booking creation
    API->>Store: Store result keyed by abc-123 (24h TTL)
    API-->>Client: 201 Created

    Note over Client,API: Network timeout — client retries
    Client->>API: POST /v1/bookings, Idempotency-Key: abc-123 (same)
    API->>Store: Key seen before?
    Store-->>API: Yes — return stored result
    API-->>Client: 201 Created (same booking, not duplicated)
```

### Correlation IDs

Every request is assigned a correlation ID (`X-Correlation-Id`), injected at the API Gateway if the client does not supply one, and propagated automatically through every module and asynchronous event the request touches, per the Observability Principles in `ai-docs/03-system-architecture-principles.md`. No engineer manually threads a correlation ID through a function signature — it is carried by shared logging/tracing middleware (`ai-docs/09-tech-stack.md`), never as a parameter an engineer could forget to pass.

---

# Response Design

### Success Envelope

Every successful response follows one fixed envelope shape, applied consistently across every module, per `ai-docs/05-coding-standards.md`:

```json
{
  "data": { "id": "8f14e45f-...", "status": "CONFIRMED", "scheduledAt": "2026-08-01T10:00:00Z" },
  "meta": {
    "requestId": "b3f1c2a4-...",
    "timestamp": "2026-07-24T09:12:00Z"
  }
}
```

### Error Envelope

Every error response follows its own fixed shape — never the success envelope with an error stuffed inside `data`, per `ai-docs/05-coding-standards.md`:

```json
{
  "error": {
    "code": "BOOKING_CANCELLATION_WINDOW_EXPIRED",
    "message": "This booking can no longer be cancelled.",
    "details": [{ "field": "scheduledAt", "issue": "less than 2 hours before start" }],
    "requestId": "b3f1c2a4-..."
  }
}
```

### Metadata

The `meta` object carries response-level information that is not part of the resource itself: `requestId` (for support/debugging correlation), `timestamp` (server response time, useful for clock-drift-sensitive clients), and, for list responses, `pagination` (see below). `meta` never carries citizen-sensitive data — it exists purely to help a client and a support engineer reason about the response, per the Data Classification tiers in `ai-docs/10-security-standards.md`.

### Pagination Metadata

Every list response includes a `pagination` object inside `meta`, whose shape depends on the pagination strategy chosen for that endpoint (see Pagination below):

```json
{
  "data": [ /* array of bookings */ ],
  "meta": {
    "requestId": "b3f1c2a4-...",
    "timestamp": "2026-07-24T09:12:00Z",
    "pagination": {
      "nextCursor": "eyJpZCI6IjEyMyJ9",
      "hasMore": true
    }
  }
}
```

---

# Pagination

Every list-returning endpoint is paginated, without exception, per the Pagination standard in `ai-docs/05-coding-standards.md` and the API payload budgets in `ai-docs/11-performance-standards.md` — an unbounded list response is both a performance risk and, per `ai-docs/10-security-standards.md`, a denial-of-service risk.

### Cursor Pagination

**Default for high-volume, frequently-changing collections** (order history, notifications, booking history, application timelines). A cursor is an opaque, encoded pointer to a specific position in the result set, immune to the "shifted page" problem that occurs when rows are inserted/deleted between two sequential page reads under offset pagination.

```
GET /v1/bookings?limit=20&cursor=eyJpZCI6IjEyMyJ9
```

```json
{
  "data": [ /* 20 bookings */ ],
  "meta": {
    "pagination": {
      "nextCursor": "eyJpZCI6IjE0MyJ9",
      "hasMore": true
    }
  }
}
```

### Offset Pagination

**Acceptable only for small, stable, admin-facing lists** where the total count is genuinely useful to display (e.g., "Page 3 of 12" in a government-officer dashboard) and the underlying collection does not experience high write-volume during a typical browsing session.

```
GET /v1/admin/departments?page=2&pageSize=25
```

```json
{
  "data": [ /* 25 departments */ ],
  "meta": {
    "pagination": {
      "page": 2,
      "pageSize": 25,
      "totalItems": 287,
      "totalPages": 12
    }
  }
}
```

### Choosing Between Them

| Factor | Cursor | Offset |
|---|---|---|
| Collection write frequency | High (safe against shifting pages) | Low/stable only |
| Need for "total count" / "jump to page N" UX | No | Yes |
| Collection size | Large / unbounded | Small, bounded |
| Arwal default | **Yes — citizen-facing lists** | Admin/government dashboards only |

### Limits

Every paginated endpoint enforces a maximum page size, never trusting a client-supplied `limit`/`pageSize` value unconditionally: a default of **20** items and a hard ceiling of **100** items per request, per the API payload budget in `ai-docs/11-performance-standards.md`. A request exceeding the ceiling is not rejected outright — it is silently clamped to the ceiling, since a client requesting too much data is a performance risk to guard against, not a malformed request to punish.

```mermaid
graph TD
    A[List Request] --> B{Client-supplied limit<br/>within 1-100?}
    B -->|No, too high| C[Clamp to 100]
    B -->|No limit supplied| D[Default to 20]
    B -->|Yes| E[Use requested value]
    C --> F[Execute paginated query]
    D --> F
    E --> F
```

---

# Filtering

Filters are expressed as explicit, allow-listed query parameters (`?status=confirmed&providerId=...`), never a free-form query language exposed directly to clients, per `ai-docs/05-coding-standards.md`. Every filterable field is backed by a deliberate database index (`ai-docs/11-performance-standards.md`) before it is exposed publicly — a filter parameter is never added to a contract before its query-performance implication has been reviewed.

```
GET /v1/bookings?status=confirmed&providerId=8f14e45f-...&scheduledAfter=2026-08-01
```

| Filter Pattern | Example | Notes |
|---|---|---|
| Exact match | `?status=confirmed` | Backed by an equality index. |
| Range | `?scheduledAfter=2026-08-01&scheduledBefore=2026-08-31` | Backed by a range-supporting composite index, filter columns ordered per the Indexing Strategy in `ai-docs/11-performance-standards.md`. |
| Multi-value | `?status=confirmed,pending` | Comma-separated, mapped to a database `IN` clause, never string-concatenated. |
| Full-text/ranked search | `?q=electrician+wiring` | Routed to the shared Search service (`ai-docs/03-system-architecture-principles.md`), never a raw `LIKE` scan on a primary table. |

An unrecognized filter parameter is rejected with a `400 Bad Request`, not silently ignored — silently ignoring an unrecognized filter risks a client believing a filter was applied when it was not, a class of defect that is dangerous specifically when the filter is a security-relevant scope narrower (e.g., filtering to "my own bookings only").

---

# Sorting

Sortable fields are explicitly allow-listed per endpoint (`?sort=scheduledAt&order=desc`), never passed unchecked into an ORM's `orderBy`, per `ai-docs/05-coding-standards.md` — this is both a performance safeguard (an unindexed sort forces an expensive in-memory sort at scale, per `ai-docs/11-performance-standards.md`) and a security safeguard (an unchecked sort column is an information-disclosure risk that could expose internal column names, per `ai-docs/10-security-standards.md`).

```
GET /v1/bookings?sort=scheduledAt&order=desc
```

A request naming a sort field outside the endpoint's allow-list returns `400 Bad Request` with the specific list of permitted values in the error's `details`, so a client integrating for the first time can self-correct without needing to consult separate documentation.

---

# Field Selection

Where a resource carries a genuinely large or expensive-to-compute set of fields, an endpoint may support explicit field selection via a `fields` query parameter, allow-listed exactly as Sorting above:

```
GET /v1/bookings/:id?fields=id,status,scheduledAt
```

Field selection is an **optimization**, never a substitute for correct DTO design (`ai-docs/05-coding-standards.md`) — a response DTO is still scoped to exclude internal-only fields regardless of what a client requests; field selection can only narrow an already-safe response further, never widen it to expose a field the DTO would not otherwise include. Field selection is applied selectively, only to endpoints where profiling (`ai-docs/11-performance-standards.md`'s Measure Before Optimizing) has shown a genuine payload-size benefit — it is not a default feature added to every endpoint speculatively, per YAGNI (`ai-docs/02-engineering-principles.md`).

---

# Authentication

Authentication is enforced exclusively through the unified Authentication shared service (`ai-docs/03-system-architecture-principles.md`, `ai-docs/10-security-standards.md`) — no module's API implements its own authentication logic.

### JWT

Every protected endpoint requires a `Bearer` JWT in the `Authorization` header, signed with RS256, short-lived (minutes, not hours), and validated independently at both the API Gateway and the module boundary (defense in depth), per `ai-docs/10-security-standards.md`.

```http
GET /v1/bookings/me
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Refresh Tokens

A client exchanges an expired access token for a new one via a dedicated, non-resource endpoint:

```
POST /v1/auth/refresh
{ "refreshToken": "..." }
```

Refresh tokens rotate on every exchange and are never accepted twice, per `ai-docs/10-security-standards.md` — a reused refresh token is treated as a signal of compromise and invalidates the entire session chain.

### OAuth 2.0 / OpenID Connect

Where Arwal exposes API access to a trusted third party (per the Open Ecosystem Phase, `ai-docs/01-product-goals.md`), authentication follows the OAuth 2.0 Authorization Code flow with PKCE, per `ai-docs/09-tech-stack.md` — never a bespoke third-party API key scheme, since OAuth 2.0/OIDC is an open, well-audited standard that gives Arwal a proven path to government single-sign-on integration without a future protocol rewrite.

### Authorization

Authentication answers "who is this actor"; authorization — enforced at the Application Layer of every use case, never assumed from a request reaching a controller — answers "can this actor perform this specific action on this specific resource," per the full Authorization Standards in `ai-docs/10-security-standards.md`. Every endpoint's OpenAPI specification documents which role(s) may call it and whether a resource-ownership check applies.

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant Module
    Client->>Gateway: Request + Bearer JWT
    Gateway->>Gateway: Validate signature, expiry, issuer
    Gateway->>Module: Forward (authenticated)
    Module->>Module: Re-validate JWT (defense in depth)
    Module->>Module: Role check
    Module->>Module: Resource-ownership check
    Module-->>Client: 200 / 403 / 404
```

---

# Validation

### Zod (Presentation Layer)

Every request DTO is defined and validated using Zod (`ai-docs/09-tech-stack.md`), applied at the Presentation Layer boundary before any business logic executes, per `ai-docs/05-coding-standards.md`. Zod's TypeScript-first inference means the validated shape and the compile-time type are always in lockstep — there is no separate, hand-maintained type definition that can drift from the actual runtime validation rule.

### class-validator (NestJS DTO Layer)

Where a NestJS controller's DTO class benefits from decorator-based validation integrated with NestJS's own pipe architecture (`ai-docs/09-tech-stack.md`), `class-validator` decorators express the same schema-level rules as an equivalent Zod schema — Arwal does not run both validation libraries redundantly on the same field; a module chooses one validation mechanism per DTO and applies it consistently, with Zod as the default per the Tech Stack decision and `class-validator` reserved for cases where tight NestJS pipe integration is the deciding factor, documented as such.

### DTO Validation

Validation happens in exactly two places, per `ai-docs/05-coding-standards.md`: schema validation (types, required fields, formats, allow-listed enum values) at the Presentation Layer, and business-rule validation (a booking cannot be cancelled within 2 hours) inside the Domain Layer. A DTO's schema validation is never bypassed by a "trusted" internal caller — every request, including one internal module calling another's public API, is validated identically, per Zero Trust (`ai-docs/10-security-standards.md`).

### Sanitization

Beyond structural validation, textual input intended for storage or downstream rendering is sanitized against injection classes documented in `ai-docs/10-security-standards.md`: no raw HTML is accepted without passing through the shared sanitization utility, and no field is trusted to be free of adversarial content merely because it passed a type check — a syntactically valid string can still be a XSS payload, a prompt-injection attempt (for any AI-consuming field), or an SSRF-triggering URL.

```typescript
const CreateDisputeMessageSchema = z.object({
  bookingId: z.string().uuid(),
  message: z.string().min(1).max(1000), // length-bound
  // rendered client-side via React's default escaping — never dangerouslySetInnerHTML
});
```

---

# Error Handling

### Standard Error Format

Every error, across every module and every endpoint, uses the single fixed error envelope defined in Response Design above — no module defines its own error shape, per Convention over Configuration.

### Error Codes

`error.code` is a **stable, machine-readable, `SCREAMING_SNAKE_CASE` identifier** a client can branch logic on — it never changes once published, even if `error.message`'s wording is later improved for clarity. Error codes are namespaced implicitly by their specificity, not by a module prefix, since a client should never need to know which internal module produced an error to handle it correctly.

| Error Code | HTTP Status | Meaning |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Generic schema validation failure; `details` carries field-level specifics |
| `AUTHENTICATION_REQUIRED` | 401 | No or invalid credential supplied |
| `TOKEN_EXPIRED` | 401 | A specific, actionable variant a client can use to trigger silent token refresh |
| `FORBIDDEN` | 403 | Authenticated but not authorized |
| `RESOURCE_NOT_FOUND` | 404 | Generic not-found; specific domain errors (e.g., `BOOKING_NOT_FOUND`) are preferred where meaningful |
| `BOOKING_CANCELLATION_WINDOW_EXPIRED` | 422 | Domain-specific business-rule violation |
| `BOOKING_SLOT_CONFLICT` | 409 | Concurrent booking of an already-taken slot |
| `RATE_LIMIT_EXCEEDED` | 429 | Actor has exceeded a rate limit |
| `INTERNAL_ERROR` | 500 | Generic, unexpected server fault — never more specific to an internal client, per the Error Handling table in `ai-docs/02-engineering-principles.md` |

### User-Safe Messages

`error.message` is always citizen-safe, plain-language text suitable for direct display — "This booking can no longer be cancelled," never "BookingCancellationWindowExpiredError: hoursUntilScheduled(1.2) < CANCELLATION_CUTOFF_HOURS(2)." A raw exception message, a stack trace, an internal file path, or a database engine detail is never present in any response reachable by a client, in any environment, per `ai-docs/05-coding-standards.md` and `ai-docs/10-security-standards.md` — such detail is direct reconnaissance value to an attacker and a citizen-trust failure regardless.

```mermaid
graph TD
    A[Exception Raised in Domain/Application Layer] --> B{Typed AppError<br/>subclass?}
    B -->|No, unexpected| C[500, INTERNAL_ERROR,<br/>generic message, full detail logged only]
    B -->|Yes| D[Presentation Layer maps<br/>to HTTP status + error code]
    D --> E[Citizen-safe message returned]
    E --> F[Full internal detail<br/>never leaves the server]
```

---

# Rate Limiting

Rate limiting is enforced at two layers, per `ai-docs/09-tech-stack.md` and `ai-docs/10-security-standards.md`: coarse-grained, IP/route-based limiting at Nginx/the API Gateway to absorb abusive or automated traffic before it reaches application code, and fine-grained, per-actor/per-endpoint limiting inside NestJS (`@nestjs/throttler`) for sensitive operations where a stricter limit is warranted.

| Operation Class | Limit | Window | Rationale |
|---|---|---|---|
| General authenticated read | 300 requests | per minute, per actor | Generous enough for normal browsing; catches runaway client polling. |
| General authenticated write | 60 requests | per minute, per actor | Writes carry more downstream cost; a tighter ceiling. |
| Login / OTP request | 5 requests | per 10 minutes, per phone number/IP | Blunts brute-force and OTP-bombing, per `ai-docs/10-security-standards.md`. |
| Payment initiation | 10 requests | per hour, per actor | Sensitive, financially consequential operation; deliberately conservative. |

Every rate-limited response returns `429 Too Many Requests` with a `Retry-After` header — never a silent drop, per the API Security Standards in `ai-docs/10-security-standards.md`.

```mermaid
graph LR
    A[Request] --> B[Nginx: Coarse IP/Route Limit]
    B -->|Within limit| C[API Gateway]
    B -->|Exceeded| D[429 at the edge]
    C --> E[NestJS: Fine-Grained Per-Actor Limit]
    E -->|Within limit| F[Business Logic Executes]
    E -->|Exceeded| G[429, Retry-After header]
```

---

# OpenAPI Standards

Every public and internal-module-boundary API is documented as an OpenAPI 3.x specification, generated directly from NestJS's decorator-based controller/DTO definitions via `@nestjs/swagger`, per `ai-docs/09-tech-stack.md` — never hand-maintained prose that can drift from the actual implementation, per the Documentation Standards in `ai-docs/02-engineering-principles.md`.

### Requirements

- Every endpoint declares its request schema, response schema (success and every documented error case), required authentication/role, and a short, plain-language description of its purpose.
- Every DTO field carries a description sufficient for a new integrator to understand its meaning without consulting the source code.
- The specification is the single source from which `packages/sdk`'s typed API client is generated (`ai-docs/09-tech-stack.md`), meaning a contract drift between implementation and documentation is structurally impossible — the SDK simply reflects whatever the decorators actually declare.
- Deprecated endpoints/versions are marked `deprecated: true` in the specification the moment deprecation is announced (see API Versioning above), never left implicitly deprecated in prose alone.

```typescript
@ApiOperation({ summary: "Cancel an existing booking" })
@ApiResponse({ status: 200, description: "Booking cancelled", type: BookingResponseDto })
@ApiResponse({ status: 404, description: "Booking not found" })
@ApiResponse({ status: 422, description: "Cancellation window has expired" })
@Patch(":id/cancel")
async cancel(@Param("id") id: string, @CurrentActor() actor: AuthenticatedActor) {
  // ...
}
```

---

# API Observability

Every API request is observable end to end, per the Observability Principles in `ai-docs/03-system-architecture-principles.md` and the Observability section of `ai-docs/11-performance-standards.md`:

- **Structured logs**, correlated by `X-Correlation-Id`, capture request method, path, status code, latency, and actor identity (never the request body's sensitive fields, per the Sensitive Data Masking standard in `ai-docs/10-security-standards.md`).
- **Metrics** (latency histograms, request rate, error rate) are emitted per endpoint via OpenTelemetry, scraped by Prometheus, visualized on a Grafana dashboard — a service is not production-ready until this dashboard exists, per `ai-docs/02-engineering-principles.md`.
- **Distributed tracing** spans every module and shared service a request touches, letting a single slow or failing citizen-facing call be diagnosed precisely rather than guessed at.
- **Health/readiness endpoints** (`/health/live`, `/health/ready`) are a mandatory, standardized contract for every deployable service, per `ai-docs/09-tech-stack.md`.

```mermaid
graph LR
    A[API Request] -->|X-Correlation-Id propagated| B[Structured Logs]
    A --> C[OpenTelemetry Metrics]
    A --> D[Distributed Trace Span]
    B --> E[Log Aggregation]
    C --> F[Prometheus]
    D --> F
    F --> G[Grafana Dashboard:<br/>p50/p95/p99, error rate, traffic]
```

---

# API Testing

API testing follows the Testing Pyramid established in `ai-docs/02-engineering-principles.md`, applied specifically to the API surface:

| Test Type | Tooling | Verifies |
|---|---|---|
| **Unit tests** | Jest (`ai-docs/09-tech-stack.md`) | The use case and domain logic behind an endpoint, in isolation from HTTP. |
| **Integration tests** | Jest + Supertest | The actual HTTP request/response contract — status codes, envelope shape, headers — against a real or in-memory NestJS instance, per the Integration Tests standard in `ai-docs/05-coding-standards.md`. |
| **Contract tests** | Generated from the OpenAPI specification | That the implementation matches its published contract exactly — a divergence here is treated as a Blocking Issue, since it means the OpenAPI spec (and therefore `packages/sdk`) has silently drifted from reality. |
| **E2E tests** | Playwright (`ai-docs/09-tech-stack.md`) | A full citizen journey (checkout, booking, application submission) spanning multiple API calls across the full stack, per the curated E2E scope in `ai-docs/02-engineering-principles.md`. |

Every new or modified endpoint's PR includes, at minimum, an integration test asserting its success response shape, its documented error responses, and its authorization behavior (an unauthenticated or wrongly-authorized request receives the correct `401`/`403`/`404`), per the API Definition of Done in `ai-docs/08-definition-of-done.md`.

---

# API Review Checklist

Every pull request introducing or modifying an API endpoint is checked against the following before merge, extending the API Definition of Done in `ai-docs/08-definition-of-done.md`:

- [ ] **URI follows naming conventions** — plural, `kebab-case`, no verbs in the path, nesting limited to one level.
- [ ] **Correct HTTP method chosen** — matches the semantics in the HTTP Methods table above.
- [ ] **Correct status codes used** — success and every documented error case map to the approved table above.
- [ ] **Versioned** — the endpoint lives under an explicit `/v1/...` (or current) prefix.
- [ ] **Request validated** — schema validation at the Presentation Layer via Zod/class-validator; business-rule validation in the Domain Layer; unrecognized fields rejected (`.strict()`).
- [ ] **Response uses the standard envelope** — success (`data`/`meta`) or error (`error`) shape, never a bespoke structure.
- [ ] **Paginated where applicable** — every list endpoint is paginated with the correct strategy (cursor default, offset for small admin lists), bounded page size, and pagination metadata present.
- [ ] **Filtering and sorting allow-listed** — no unchecked pass-through to the ORM.
- [ ] **Idempotency key required** — for any state-mutating `POST` reachable via client retry.
- [ ] **Authentication and authorization enforced** — role check and resource-ownership check at the Application Layer, never assumed from reaching the controller.
- [ ] **Error responses are citizen-safe** — no stack trace, no internal detail, stable `error.code`, plain-language `error.message`.
- [ ] **Rate limiting applied** — appropriate to the operation's sensitivity class.
- [ ] **OpenAPI specification complete** — request/response schemas, auth requirements, and descriptions are present and generated from the actual decorators, not hand-written separately.
- [ ] **Tests present** — integration tests covering success, documented error cases, and authorization behavior; E2E coverage if the endpoint is part of a critical citizen journey.
- [ ] **Performance budget respected** — payload size within the limits in `ai-docs/11-performance-standards.md`; no unreviewed N+1 query.
- [ ] **Breaking-change determination made explicitly** — any change classified as breaking per the table in API Versioning above ships as a new version, never silently inside the current one.
- [ ] **Deprecation properly signaled** — if this change deprecates a prior endpoint/version, the `Deprecation`/`Sunset` headers and OpenAPI flags are in place.

A pull request failing any item above is not merged until resolved — this checklist carries the same non-negotiable authority as every other review checklist established in the preceding thirteen phase documents.

---

# Deprecation Policy

Deprecation is never silent, and it is never permanent limbo — an endpoint or version is either actively supported, actively (and visibly) deprecated with a committed retirement date, or retired. The policy mirrors, at the API layer, the Technology Deprecation Policy in `ai-docs/09-tech-stack.md`.

### Deprecation Steps

1. **Decision and documentation** — Deprecating an endpoint or version is itself an architecturally significant decision; per the Architecture Review Workflow (`ai-docs/07-development-workflow.md`), it is documented via an ADR stating why the old contract is being retired and what replaces it.
2. **Signal immediately** — The `Deprecation` header, the OpenAPI `deprecated: true` flag, and direct communication to every known consuming client team are applied in the same change that ships the replacement, never after the fact.
3. **Minimum 180-day sunset window** — per the Sunset Policy in API Versioning above, extended if telemetry shows meaningful remaining traffic.
4. **Monitor migration** — Traffic on the deprecated endpoint/version is tracked on a dashboard (`ai-docs/11-performance-standards.md`) throughout the sunset window, not merely assumed to be draining.
5. **Retire deliberately** — The endpoint/version is removed only once the sunset window has closed and traffic has fallen to a negligible, explicitly agreed threshold — retirement is a scheduled, communicated event, never a surprise.

```mermaid
graph TD
    A[Replacement Contract Ready] --> B[ADR: Document Deprecation Decision]
    B --> C[Deprecation Signaled:<br/>Headers + OpenAPI + Direct Notice]
    C --> D[180-Day Minimum Sunset Window]
    D --> E[Migration Traffic Monitored]
    E --> F{Traffic Negligible &<br/>Window Closed?}
    F -->|No| D
    F -->|Yes| G[Endpoint/Version Retired]
```

---

# Closing Statement

> **Callout — Closing Statement**
> Every preceding phase document described a dimension of how Arwal is built well, safely, fast, and accessibly; this document describes the single surface through which every citizen, merchant, government officer, and future third-party partner actually touches all of that work — the API. A perfectly architected, perfectly secured, perfectly performant module is only as good as the contract it exposes: a URI a stranger can guess correctly, a status code that means the same thing everywhere, an error message a citizen can act on, and a version that never breaks a client without warning. This document exists so that "the API" is never an accumulation of a thousand small, inconsistent decisions made independently by whoever happened to build a given endpoint that week — it is one contract, held to one standard, for every one of the ~300 micro-phases still ahead, from the first `POST /v1/bookings` to the millionth citizen's daily use of the platform. Where a future phase must deviate from a rule stated here, that deviation is made explicitly — through the API Change Workflow (`ai-docs/07-development-workflow.md`) and an Architectural Decision Record where the deviation is structural — never silently, and never by default.

This document, `ai-docs/13-api-design-guidelines.md`, is the fourteenth phase of approximately 300. Every endpoint, DTO, status code, and error response built in the phases that follow is expected to satisfy the standards defined here, or to justify its deviation in writing.

**End of Phase 14 — `ai-docs/13-api-design-guidelines.md`**