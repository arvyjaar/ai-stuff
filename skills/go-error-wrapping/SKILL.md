---
name: go-error-wrapping
description: Go error wrapping and error naming conventions. Use when writing or reviewing Go code that returns, wraps, defines, or maps errors (sentinel errors, repository boundary, layers adding context).
---

# Go Error Wrapping

References:
- https://github.com/uber-go/guide/blob/master/style.md#error-wrapping
- https://preslav.me/2026/05/19/10-golang-error-handling-commandments/

## Rules

- **Bubble errors up and add context when wrapping.** Each layer describes what it was trying to do, not what went wrong; the inner error already says that.
```go
return fmt.Errorf("%w: %w", ErrSomethingHappened, err)
// extra details can be added:
return fmt.Errorf("%w: code %s %w", ErrSomethingHappened, "400", err)
```
- **Each layer adds its own context without repeating inner errors.** Example chain:
  - initial: vendor DB error
  - repository: `database.ErrUpdateRecord`
  - app service: `ErrInvalidateSession`
- **Include relevant IDs** where they help diagnose failures. Add only what callers above cannot already see:
```go
return fmt.Errorf("%w: user %s: %w", ErrLoadUser, userID, err)
```
- **Map vendor/SQL errors to project errors at the repository boundary.** Raw `mysql.*` or `sql.*` errors must never leak past the repository. Translate them to domain-level sentinel errors (e.g. in `internal/database/errors.go`):
```go
// adapters/repository.go
user, err := r.db.UserFindByID(ctx, id)
if errors.Is(err, sql.ErrNoRows) {
    return domain.User{}, fmt.Errorf("%w: user %s: %w", database.ErrRecNotFound, id, err)
}
```
- **Place error definitions in the package's `errors.go`.**

## Error variable naming

Don't put "failed" or "unable" in the name; it is an error already. The error text explains the details.
Good: `ErrValidateUser`. Bad: `ErrFailedToValidateUser`.

## Alternative: action phrases (not yet adopted, worth considering)

Instead of sentinel vars, wrap with short action phrases so the chain reads like prose. Skip prefixes like `failed to` or `cannot`:
```
placing order [ORDER_ID]: reserving stock [STOCK_ID]: connection refused
```

## Logging errors (related)

- Log at the outermost boundary (e.g. HTTP handler), not in every layer.
- Don't use `Msg(err.Error())`: unique messages pollute Sentry and block filtering. Use a fixed message, e.g. `.Msg("Unable to find user by ID")`, and attach the error with `.Err(err)`.
- Log context keys in snake_case, e.g. `.Str("remote_seller_id", id)`.

### Adding variables to the log context

Deeper layers don't log; they only **enrich the logger stored in `ctx`**. The outer boundary that finally logs the error then prints all that context. Build a sub-logger with `With()`, then put it back into the context with `log.WithContext(ctx)`:
```go
func (p *Processor) Durable(ctx context.Context, input CreateDurableInput) (CreateDurableOutput, error) {
    log := zerolog.Ctx(ctx).With().
        Str("idempotency_id", input.IdempotencyID.ToString()).
        Str("wallet_from_id", input.FromWalletID.ToString()).
        Str("wallet_to_id", input.ToWalletID.ToString()).
        Int64("debited_amount", input.DebitedFunds.Amount).
        Str("debited_currency", input.DebitedFunds.Currency).
        Str("transfer_type", input.Metadata.TransferType).
        Str("remote_order_id", input.Metadata.RemoteOrderID).
        Str("remote_seller_id", input.Metadata.RemoteSellerID).
        Logger()
    ctx = log.WithContext(ctx)
    // pass ctx on; calls below inherit these fields
```
- Do this once, at the start of a use case, with identifiers useful for diagnosis (IDs, amounts, types).
- Use snake_case keys. No secrets or personal data.
- Adding context is not logging: don't add `log.Error()` calls in inner layers.

## Tracing errors (related)

Record the wrapped error on the span before returning:
```go
if err != nil {
    wrpErr := fmt.Errorf("%w: %w", ErrFindUserAccount, err)
    m.tracer.RecordSpanError(wrpErr, span)
    return wrpErr
}
```

## Checklist

- [ ] Wrapped with `%w` and a layer-specific sentinel/action phrase?
- [ ] Message says what this layer was doing, not a repeat of the inner error?
- [ ] No `mysql.*`/`sql.*` errors escape the repository?
- [ ] IDs added only if the caller can't see them?
- [ ] Error name has no "failed"/"unable"; defined in `errors.go`?
- [ ] Useful IDs added to the context logger (`With()` + `WithContext`) instead of logging deeper?
- [ ] Logged once, at the outer boundary, with a fixed message?
