# Go Backend Rewrite — Design Document

**Date:** 2026-03-07
**Status:** Approved
**Scope:** Full rewrite of `backend/` Go code using squad-pattern architecture

## Problem

The existing Go backend code (`backend/`) was written as a quick scaffold with:
- Manual `os.Getenv` config loading (no validation, no struct tags)
- Manual goroutine lifecycle management (channels, `go func()` in server.Start)
- Global exported interfaces in `adapter/messenger.go` (violates local interface principle)
- Flat package structure without the App pattern
- No structured service lifecycle (no oklog/run or similar)

The code works but doesn't follow the architecture patterns we want for production quality.

## Decision

Rewrite all Go backend code from scratch using the architecture style guide (documented in `.claude/agents/go-backend.md`). Archive existing code as reference.

## Architecture Changes

### Before (Current)

```
backend/
├── cmd/server/main.go              # Manual signal handling, goroutine lifecycle
├── internal/
│   ├── adapter/messenger.go        # Global exported Adapter interface
│   ├── config/config.go            # Manual os.Getenv, no validation
│   ├── server/
│   │   ├── server.go               # Start() spawns goroutine, returns address
│   │   └── middleware.go
│   ├── database/
│   ├── telegram/
│   │   ├── adapter.go
│   │   ├── botapi/
│   │   └── mtproto/
│   ├── salesforce/
│   ├── media/
│   └── realtime/
└── go.mod
```

### After (Rewrite)

```
backend/
├── cmd/messenger/main.go           # oklog/run lifecycle, Init() in same file
├── internal/
│   ├── common/
│   │   └── models/models.go        # Platform, Direction, Message (shared types)
│   ├── messenger/
│   │   ├── app.go                  # const AppName, NewApp(), Run()
│   │   └── config/config.go        # Struct tags + validator
│   ├── server/
│   │   ├── server.go               # Run() blocks, Shutdown() for interrupt
│   │   ├── handlers.go             # Health, webhook handlers
│   │   └── middleware.go           # HMAC auth
│   ├── database/
│   │   ├── pool.go                 # pgxpool connection
│   │   └── migrations.go           # SQL migrations
│   ├── telegram/
│   │   ├── adapter.go              # TelegramAdapter struct
│   │   ├── botapi/
│   │   │   └── handlers.go         # Webhook handler
│   │   └── mtproto/
│   │       ├── client.go           # MTProto client
│   │       ├── session_store.go    # PostgreSQL session store
│   │       ├── messages.go         # Message listener
│   │       └── ratelimit.go        # Rate limiter, flood waiter
│   ├── salesforce/
│   │   ├── auth.go                 # JWT Bearer auth
│   │   ├── client.go               # Auto-refreshing SF client
│   │   └── ingestion.go            # Batch collector with sync.Pool
│   ├── media/
│   │   └── r2.go                   # R2Client via AWS SDK v2
│   └── realtime/
│       └── centrifugo.go           # Centrifugo publisher
└── go.mod
```

## Key Pattern Changes

### 1. Service Lifecycle: oklog/run

**Before:** Manual `signal.NotifyContext` + `go func()` + `<-ctx.Done()` + explicit `Shutdown()` call.

**After:** `oklog/run.Group` with `(execute, interrupt)` actor pairs. Each long-running service registers as an actor. When any actor returns, all others are interrupted.

**Library:** `github.com/oklog/run` — open-source, 100 lines, same pattern as the style guide's `squad`.

### 2. App Pattern

**Before:** No app abstraction. `main.go` directly creates server and manages lifecycle.

**After:** `internal/messenger/app.go` with `const AppName`, `NewApp()` (constructor at end of file), `Run()` returning `(execute, interrupt)`.

### 3. Config: Struct Tags

**Before:** 85 lines of manual `os.Getenv` with hand-rolled validation.

**After:** Config struct with `env:"VAR_NAME,required"` tags + `validate:"required,min=1"`. Parsed by `caarlos0/env`, validated by `go-playground/validator`.

### 4. Local Interfaces

**Before:** Exported `adapter.Adapter` interface in `adapter/messenger.go`, consumed across packages.

**After:** No exported interfaces. Each consumer defines its own local (lowercase) interface with only the methods it needs. Shared types (Message, Platform, Direction) move to `common/models/`.

### 5. HTTP Server Pattern

**Before:** `Start()` spawns goroutine, returns address. Caller must call `Shutdown()` manually.

**After:** `Run()` blocks (oklog/run manages the goroutine). `Shutdown()` is the interrupt function. No manual goroutines.

### 6. Error Handling

**Before:** Mixed `slog.Error` + `return err` in some places.

**After:** Strict rule: wrap and return only (`fmt.Errorf("context: %w", err)`). Never log and return. Log only at top level.

### 7. Logging

**Keep:** `log/slog` with JSON handler. No change from current.

## What Stays the Same

- PostgreSQL via `pgx/v5` — correct choice, keep
- `gotd/td` for MTProto — correct choice, keep
- `aws-sdk-go-v2` for R2 — correct choice, keep
- HTTP (not gRPC) for server — correct for webhooks and SF callbacks
- gRPC client for SF Pub/Sub API — keep
- `sync.Pool` for batch serialization — keep
- HMAC-SHA256 middleware — keep logic, adapt to new structure
- All business logic — rewrite structure, keep algorithms

## Archive Strategy

1. Move `backend/` to `backend/_archived/` (git-tracked for reference)
2. Create new code in `backend/` following new structure
3. Old code serves as reference for business logic extraction
4. Delete archive after rewrite is validated

## Dependencies

### New
- `github.com/oklog/run` — service lifecycle
- `github.com/caarlos0/env/v11` — config parsing
- `github.com/go-playground/validator/v10` — config validation

### Kept
- `github.com/jackc/pgx/v5`
- `github.com/gotd/td`
- `github.com/aws/aws-sdk-go-v2`
- `google.golang.org/grpc`
- `golang.org/x/sync/errgroup`

### Removed
- None (no proprietary libraries were used)

## Risk Assessment

**Low risk:** This is a structural rewrite. All business logic is preserved. The existing code is stubs/scaffolding (no production traffic). The archive preserves the original for reference.

## Success Criteria

- All existing test cases pass (adapted to new structure)
- `go vet ./...` clean
- `golangci-lint run` clean
- No exported interfaces (except in `common/models/` for shared types)
- All services managed by `oklog/run`
- Config loaded via struct tags with validation
- No manual goroutines for service lifecycle
- Constructors at end of files
- No log-and-return error pattern
