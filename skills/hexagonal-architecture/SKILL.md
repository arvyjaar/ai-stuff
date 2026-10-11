---
name: hexagonal-architecture
description: Hexagonal / DDD-lite architecture playbook for Go services (domain, app commands/queries, ports, adapters, repositories, CQRS, cross-feature dependencies). Use when designing a feature, adding a use case, deciding which layer code belongs to, or reviewing package structure.
---

# Hexagonal Architecture (DDD-lite) – playbook

For: you, your mentee, and agents. Sources:
- https://threedots.tech/post/introducing-clean-architecture/
- https://threedots.tech/post/ddd-cqrs-clean-architecture-combined/
- https://github.com/ThreeDotsLabs/wild-workouts-go-ddd-example

Items marked **(house rule)** are our own choices and may differ from the articles. Items marked **(article)** come from the articles above.

## The idea in one minute

Keep business rules free of technical details (HTTP, SQL, Kafka). Then code is easier to test, change and understand. Think of domain structs as **types with behavior**, not dumb ORM-like data. Stakeholders say "User passed KYC", not "User KYC level set to REGULAR".

## Names (the same thing has many names)

| Our folder  | Meaning                                                                            | Other names                 |
|-------------|------------------------------------------------------------------------------------|-----------------------------|
| `ports/`    | Entry points: how the outside world calls us (HTTP, Kafka consumer, Temporal, CLI) | primary / driving adapters  |
| `app/`      | Use cases; thin orchestration                                                      | application services        |
| `domain/`   | Business rules and state changes                                                   | entities                    |
| `adapters/` | How we call the outside world (DB, external APIs, notifications)                   | secondary / driven adapters |

## Dependency rule (article)

Outer layers depend on inner layers, never the other way round.
- `domain` imports nothing from our other layers.
- `app` imports `domain` only. It must not know which database or URL is used. **Test:** if you read app code and can't tell what DB or URL it uses, the layering is right.
- `ports` and `adapters` may import `app` and `domain`, but **never each other**. A port must not call an adapter directly.
- Inner layers define the interfaces they need; adapters implement them. Wiring (Wire / `main`) connects them.
- Enforce with a linter (e.g. `go-cleanarch`) locally or in CI. Don't merge packages to escape an import cycle; fix the dependency direction.

**Flow:** request → router/registry → `ports/*` (translate protocol ↔ command) → `app/command|query` (use case) → driven-port interface → `adapters/` (DB, external API).

## What goes where

| Question                                                                  | Layer      |
|---------------------------------------------------------------------------|------------|
| Parse HTTP/Kafka/Temporal input, map result and errors to a response      | `ports`    |
| Load data, call domain methods, call external services, save              | `app`      |
| Is this allowed? What is the new state? Calculations about business rules | `domain`   |
| SQL, HTTP clients, message publishing, vendor error translation           | `adapters` |

If an `if` in `app` expresses a business rule, move it into the domain. If it is just "what step comes next", it stays in `app`.

## Ports (driving)

Thin: parse input, call `app.Commands` / `app.Queries`, map the result to a response. No business logic. Translate app errors into protocol answers (e.g. "incorrect input" → HTTP 400).

## Application layer

- Thin orchestrator: **fetch → call domain → persist/notify**. Sometimes very short; that is fine.
- Returns **port-agnostic errors** (article): the app never knows about HTTP or gRPC status codes. Ports translate. Optionally give input errors a slug (`date-from-after-date-to`) the frontend can localize.
- Wrap calls to other services behind app-level interfaces named after business concepts, not after microservices.
- Constructors take interfaces and fail fast on nil dependencies; wiring happens in one place (Wire / `main`).

### Command / Query split (CQRS)

- `app/command/`: change state. One file per use case; handler takes a typed command struct and returns a view or nothing.
- `app/query/`: read state, no side effects. One file per use case. Input is a typed query struct (`TrainingsForUser{User}`), like commands.
- **Read models (wild-workouts example):** a query declares its own small read-model interface and its own view structs (`query/types.go`). The adapter fills those view structs straight from the DB and returns them. Queries do **not** load domain entities. Derived display fields (e.g. `CanBeCancelled`) are computed in the adapter, calling a domain method if needed. A query handler stays thin: validate input, call the read model.
- Each of command/ and query/ has an `interfaces.go` with **only** the methods its use cases call; prefer one small interface per use case. No god-interface; dependencies stay explicit; mocks stay small.
- Benefit: no reusable structs with undefined fields.

### Interfaces: consumer side (article)

Define interfaces next to the code that uses them. Producers (DB driver, other features) then don't need to know every way they are used. Keep them small.

## Domain layer

- Put business decisions here: `CanBeCancelled()`, `Cancel()`, `PassKYC()`. Name methods after business actions, not setters.
- **Always-valid objects (article):** validate in a constructor (`NewX(...) (X, error)`) and keep every method preserving validity. Prefer unexported fields where callers could otherwise build an invalid state.
- Expected business failures are sentinel errors in `domain/errors.go` (`ErrTrainingAlreadyCanceled`).
- Calculations that don't belong to one entity: small plain functions, not "domain service" objects.
- Separate state changes from side-effect calculations (single responsibility).
- Typed enums with `IsValid()`, defined in the domain package. Raw strings only at boundaries, then convert.
```go
type KYCStatus string
const (
    KYCStatusPending   KYCStatus = "PENDING"
    KYCStatusValidated KYCStatus = "VALIDATED"
)
func (e KYCStatus) IsValid() bool {
    switch e {
    case KYCStatusPending, KYCStatusValidated:
        return true
    }
    return false
}
```
- **No mutation (house rule):** don't mutate structs through pointers. Use CQRS targeted commands, or methods with a value receiver that return a copy:
```go
func (u User) WithMangoData(mangoID, kycLevel string) User { // value receiver = copy
    u.MangoID = mangoID
    u.KYCLevel = kycLevel
    return u
}
```
- Domain tests are easy and fast (no infrastructure). Test business rules there.

## Models and DTOs

- No central `dto` package (couples DB, HTTP and external API models). Models live in their domain package.
- Keep separate models for DB, HTTP and domain so they can evolve alone.
- Avoid one God struct with many nil/zero fields. Split into:
  - **Command structs** (input): strict, every field required (`UpdateKYCStatusCmd`).
  - **Core entities**: true business state.
  - **Read/view structs** (output): tailored for responses.
- **Push mapping to the edges:** handlers map request → command; repositories map entity ↔ `db.Params`. Domain and app know nothing about HTTP or SQL.
- Structs hold only fields actually returned; no meaningless zero values; prefer composition. Use the `exhauststruct` linter on refactored code.

## Repository pattern

- sqlc `db.Queries` is the DAO. The service calls the repository; the repository calls `db.Queries` and maps results **directly to domain objects**.
- **No `db.*` leaks.** Generated structs never leave the repository.
- Map vendor/SQL errors to project errors at the repository boundary (see `go-error-wrapping`).
- **No business decisions in the repository or inside a DB transaction.** Rules live in domain/app; the repository stores and loads.
- Prefer SQL JOINs over Go-level joins where sqlc allows. Colliding column names: `sqlc.embed(...)`, or explicit aliases (`u.id AS user_id`), or as last resort separate queries stitched together inside the repository.
- A repository does **not** have to equal one table. Think "data that must be stored transactionally together" (aggregate).
- Name methods after the business action (`UpdateAddress`, `UpdateStatus`, `LockWallet`), never `UpdateModelByDTO`. Pass specific command structs or individual args **(house rule)**.
- Updates:
  - **Same aggregate** (e.g. User + Address): handled inside the repository with targeted command structs.
  - **Different aggregates** (e.g. User + KycRequest): orchestrated by the service in one transaction. Prefer `Transactor.InTx(ctx, func(tx *sql.Tx) error {...})` over passing `*sql.Tx` manually.
  - **Current state (project):** `Transactor` / `InTx` / `WithTx` are the target, not fully implemented yet; today `*sql.Tx` is passed through repository signatures. Treat that as a known tradeoff; don't copy it into new abstractions.
- **Alternative (article), not adopted here:** the *update closure*. The repository runs the transaction and calls a function supplied by the app layer, so the rule stays outside the repository:
```go
repo.UpdateTraining(ctx, id, func(t Training) (Training, error) {
    if !t.CanBeCancelled() { return t, ErrTooLateToCancel }
    return t, nil
})
```
  Consider it when many use cases only need "load, apply rule, save".
- Deprecated: separate Retriever/Updater layers that map `db.*` → `dto.*`. Fetch and map in the repository.

## Cross-feature dependencies

If feature `kyc` needs data from feature `user`: `kyc` defines the interface it needs (`app/command/interfaces.go`); an adapter (`adapters/userrepo.go`) implements it by delegating to the user package's public contract; Wire connects them. This avoids circular imports and keeps `kyc` unaware of `user` internals. Never depend on another feature's internals.

## Feature folder structure

```
<feature>/
├── adapters/    errors.go  repository.go  userrepo.go  alertnotifier.go
├── app/
│   ├── app.go
│   ├── command/ errors.go  interfaces.go  <usecase>.go ...
│   └── query/   errors.go  interfaces.go  <usecase>.go ...
├── domain/      domain.go  errors.go  <entity>.go
├── ports/
│   ├── kafka/   errors.go  <consumer>.go  wire.go
│   ├── temporal/ activities.go  workflow.go  worker.go  dto.go  errors.go  wire.go
│   └── websrv/  errors.go  <handler>.go  wire.go
└── wire.go
```

File names: prefer one word describing one role (`repository.go`, `service.go`, `handler.go`). No underscores; if you want one, the file may not belong in that package. Error definitions go in the package's `errors.go`.

## Testing

- Unit test the app layer with table-driven tests and small hand-written mocks of the consumer-defined interfaces. Fast; no database.
- Integration tests cover repositories and a few end-to-end scenarios.
- Arrange with fixture builders (`fixtures.NewUser()`), never sqlc `Create*` directly. Fixtures give valid defaults so tests specify only relevant fields.
- Service/handler tests assert final DB state through the **domain repository**. Repository tests assert through sqlc getters to verify exact column persistence.

## When not to use it (article)

Tiny services with no application logic don't need all layers; the structure adds boilerplate that pays off as the project grows. Re-evaluate when a small service becomes long-lived. Introduce the domain layer when business rules grow; keep `app` thin.

## Common mistakes

- Business `if`s in handlers or in app services instead of domain methods.
- A port importing an adapter, or `app` importing `adapters`/`ports`.
- `db.*` structs or `sql.ErrNoRows` crossing the repository boundary.
- One big interface in a shared package instead of small consumer-side ones.
- A reusable "God" struct with fields that are zero in half the use cases.
- Mutating a struct through a pointer; generic repository methods like `Save(anything)`.
- Domain package importing HTTP/SQL/vendor types.

## Checklist

- [ ] Each layer does only its job (ports translate, app orchestrates, domain decides, adapters adapt)?
- [ ] Imports follow the dependency rule; ports and adapters don't import each other?
- [ ] Interfaces defined by the consumer, with only the methods used?
- [ ] Command or query, not both? One file per use case?
- [ ] Business rule is a domain method (valid-by-construction), not an `if` in app/ports?
- [ ] App errors are port-agnostic; ports map them to protocol responses?
- [ ] No `db.*` or vendor errors leaving the repository; no business logic in it?
- [ ] Repository method named after a business action, taking a command struct?
- [ ] Cross-feature access through a consumer-defined port + adapter?
- [ ] Typed enums with `IsValid()`; mapping at the edges; no pointer mutation?
- [ ] Tests: domain rules unit-tested, app tested with small mocks, repository tested against a DB?
