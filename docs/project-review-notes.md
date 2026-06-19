# Project Review Notes

Date: 2026-06-19

## Project Shape

This is a Go 1.22 Telegram digital-goods shop bot.

- `cmd/server`: application entrypoint.
- `internal/app`: component wiring and HTTP router setup.
- `internal/bot`: Telegram bot flows, product purchase, balance recharge, orders.
- `internal/httpadmin`: web admin panel, auth, payment callbacks, admin handlers.
- `internal/store`: GORM models and data access.
- `internal/payment/epay`: EPay signing, payment URL creation, callback parsing.
- `internal/worker`: retry and order maintenance jobs.
- `templates` and `static`: admin UI.

## Verification

Ran:

- `go test ./...`: passes, but every package reports `[no test files]`.
- `go vet ./...`: passes.

## Highest Priority Risks

1. Payment callback validation is incomplete.
   - `internal/httpadmin/payment.go`
   - Callback processing verifies signature and success status, but does not verify paid amount against `order.PaymentAmount`.
   - It also does not verify callback merchant PID against configured `EPAY_PID`.

2. Payment callback idempotency is not transaction-safe.
   - `internal/httpadmin/payment.go`
   - `order.Status != "pending"` is checked before the transaction.
   - Move the pending-status guard inside the transaction using conditional update with `WHERE id = ? AND status = 'pending'` and check `RowsAffected`.

3. PostgreSQL stock claiming can treat no stock as success.
   - `internal/store/repository.go`
   - `Raw(...).Scan(&code)` may not return `gorm.ErrRecordNotFound` when no row exists.
   - Check `code.ID == 0`, use `Row().Scan`, or restructure as an atomic update.

4. Balance and recharge-card locking is unreliable.
   - `internal/store/balance.go`
   - Uses `tx.Set("gorm:query_option", "FOR UPDATE")`, which is not reliable in GORM v2 across dialects.
   - Use `clause.Locking{Strength: "UPDATE"}` or atomic conditional updates.

5. CSRF is configured but not wired.
   - `internal/httpadmin/server.go`
   - `ENABLE_CSRF` exists, but the middleware is not applied.
   - Admin writes rely on cookies, so CSRF should be implemented or cookie auth should be adjusted.

6. Startup does production-affecting mutations.
   - `cmd/server/main.go`
   - Startup seeds test products/codes and runs DDL fixes directly.
   - Gate seed behind an explicit env var and move DDL to migrations.

## Deployment Note

`deploy.sh` currently checks and invokes the legacy Compose V1 command `docker-compose`.
Many modern Docker installations, including common panel-managed setups, install Compose V2 as:

```bash
docker compose
```

The script should detect both forms and use whichever exists.
