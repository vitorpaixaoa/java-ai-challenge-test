# Backend Architecture

## What this is

A small Spring Boot REST API that backs a payments admin app. It exposes endpoints
to create and look up "charges" (simulated payment transactions) and a couple of
support-tooling endpoints (customer search by email, support message rendering).
There is no real payment processor or database integration — everything is
simulated in memory, which makes the service self-contained for local development
and testing (`mvn spring-boot:run`, listens on port `8080`).

## Tech stack

- **Java 17**
- **Spring Boot 3.3.4** (`spring-boot-starter-web`, `spring-boot-starter-validation`)
- **Maven** for build/dependency management (`pom.xml`)
- **JUnit 5 + Spring Boot Test** (`spring-boot-starter-test`) for integration tests,
  run against a random port via `TestRestTemplate`

## Package layout

Everything lives under `com.taller.charges`:

| Class | Role |
|---|---|
| `ChargesApplication` | `@SpringBootApplication` entry point that boots the embedded server |
| `ChargesController` | `@RestController` — HTTP layer, maps routes to service/store calls |
| `ChargesService` | `@Service` — business logic for creating a charge (idempotency check, calling the processor, persisting, auditing) |
| `PaymentProcessor` | `@Service` (package-private, defined alongside `ChargesService`) — simulates calling out to a payment gateway and returns a `Charge` |
| `AuditLog` | `@Service` (package-private) — logs charge events for traceability |
| `ChargeStore` | `@Component` — in-memory data store (maps/lists) standing in for a database |
| `Charge` | Java `record` — the response/domain model for a charge |
| `ChargeRequest` | Java `record` — the request DTO for creating a charge |

## Request flow (create charge)

```
Client
  -> POST /charges (ChargesController.createCharge)
     -> ChargesService.createCharge
        -> ChargeStore.findByKey(idempotencyKey)   // idempotency lookup
        -> PaymentProcessor.charge(...)             // simulate the charge
        -> ChargesService.persist(...)               // store the result
        -> AuditLog.logCharge(...)                   // record the event
     <- Charge (201 Created)
```

This is a classic **Controller -> Service -> Store/Domain** layering:
- The **controller** only handles HTTP concerns (status codes, request/response shape).
- The **service** owns business rules (idempotency, orchestration of processor + persistence + audit).
- The **store** is the persistence abstraction, currently backed by in-memory collections instead of a real database.

## API surface

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/charges` | Create a charge from a `ChargeRequest` body; requires an idempotency key |
| `GET` | `/charges` | List the most recent charges |
| `GET` | `/charges/{id}` | Fetch a single charge by id |
| `GET` | `/customers/search?email=` | Support tool: search charges by customer email |
| `POST` | `/support/render-message` | Support tool: render an HTML snippet for a customer name |

CORS is wide open (`@CrossOrigin(origins = "*")`) so the React frontend (or any
client) can call the API directly during local development.

## Data model

- `ChargeRequest` (input): idempotency key, amount, currency, customer email, card token, description.
- `Charge` (output/domain): id, amount, currency, customer email, description, status, created-at timestamp.

Both are immutable Java records — no setters, no mutable state, equality/`toString`
generated for free.

## Persistence

`ChargeStore` is a `@Component` singleton holding three in-memory collections:
- `byKey` — idempotency key -> `Charge`, used to short-circuit duplicate submissions.
- `byId` — charge id -> `Charge`, used for direct lookups.
- `all` — insertion-ordered list, used for "latest charges" and email search.

There is no real database; state lives only for the lifetime of the running process.

## Testing

`ChargesControllerTest` is a `@SpringBootTest` integration test that boots the full
Spring context on a random port and drives the `/charges` endpoint with
`TestRestTemplate`, covering the happy path (201 on a fresh idempotency key) and one
validation path (400 on a missing key).

## Build & run

```bash
cd backend
mvn spring-boot:run   # starts the API on :8080
mvn test               # runs the integration tests
```
