# MessageForge Go Style Guide

## Architecture

- Clean Architecture: layers depend on interfaces, not implementations
- One service = one responsibility
- Interfaces defined where consumed (not where implemented)

## Project Structure

```
cmd/messenger/main.go        # Entry point — Init(), oklog/run group
internal/
├── common/
│   └── models/              # Shared domain models (Message, ChatInfo)
├── messenger/
│   ├── app.go               # NewApp() — wires all dependencies, Run() returns actor pair
│   ├── config/config.go     # Config struct (caarlos0/env + go-playground/validator)
│   ├── pipeline.go          # Inbound fan-out: source → media → [realtime, ingester]
│   ├── queue_worker.go      # PostgreSQL SKIP LOCKED queue consumer
│   └── outbound_worker.go   # Outbound message delivery
├── database/
│   ├── pool.go              # pgx connection pool
│   ├── migrations.go        # Embed + auto-apply SQL migrations
│   └── migrations/*.sql     # Numbered migration files
├── server/
│   ├── server.go            # net/http server (no frameworks)
│   ├── handlers.go          # HTTP endpoint handlers
│   └── middleware.go        # HMAC auth, body limits, request logging
├── telegram/
│   ├── adapter.go           # MessengerAdapter interface for Telegram
│   ├── outbound.go          # Outbound message sending
│   ├── botapi/              # Bot API webhook handler + throttle
│   └── mtproto/             # MTProto client, session store, rate limiting
├── salesforce/
│   ├── auth.go              # OAuth 2.0 JWT Bearer Flow
│   ├── client.go            # SF REST API client
│   ├── ingestion.go         # Batch Platform Event publisher (sync.Pool)
│   └── delivery_status.go   # Delivery status PE publisher
├── media/
│   ├── r2.go                # Cloudflare R2 S3-compatible client
│   ├── inbound.go           # Download from Telegram → upload to R2
│   └── outbound.go          # Presigned URL generation for LWC upload
└── realtime/
    └── centrifugo.go        # Centrifugo gocent v3 publisher
```

## Lifecycle Management

Use `oklog/run` for ALL service lifecycle. Every long-running component returns an execute/interrupt pair:

```go
// Good: oklog/run actor pattern
func (s *Service) Run() (func() error, func(error)) {
    ctx, cancel := context.WithCancel(context.Background())
    return func() error {
            return s.serve(ctx)
        }, func(error) {
            cancel()
        }
}

// Bad: raw goroutines
go func() { s.serve() }()  // Don't do this
```

Register actors in `main.go`:
```go
var g run.Group
// Signal handler
// HTTP server
// Inbound pipeline
// Queue workers
// Session backup
```

## Configuration

Use `caarlos0/env` struct tags + `go-playground/validator`:

```go
type Config struct {
    HTTPPort    int    `env:"HTTP_PORT" envDefault:"8080" validate:"required,min=1,max=65535"`
    DatabaseURL string `env:"DATABASE_URL,required"`
}
```

Never hardcode secrets. All credentials via environment variables.

## Error Handling

- Always wrap errors with context: `fmt.Errorf("failed to create user: %w", err)`
- Never log AND return the same error — do one or the other
- Use sentinel errors sparingly; prefer wrapping with `%w`
- Non-fatal errors (e.g., media processing) should be logged and allow the pipeline to continue

## Logging

Use `log/slog` with JSON handler. Never `log.Println` or `fmt.Println`:

```go
slog.Info("message processed",
    "external_id", msg.ExternalID,
    "chat_id", msg.ChatID,
    "duration_ms", elapsed.Milliseconds(),
)
```

## Interfaces

Define small interfaces (1-3 methods) where consumed:

```go
// Good: local interface in pipeline.go
type messageIngester interface {
    Add(ctx context.Context, msg models.Message) error
}

// Bad: interfaces.go with large shared interfaces
```

## Constructors

`New*()` functions always at the END of the file:

```go
// Business logic and methods first...

func (s *Service) Process(ctx context.Context) error { ... }

// Constructor at the bottom
func NewService(db *pgxpool.Pool, log *slog.Logger) *Service {
    return &Service{db: db, log: log}
}
```

## Database

- Use `pgx` (not `database/sql`) for PostgreSQL
- Parameterized queries only — never concatenate SQL
- Use `FOR UPDATE SKIP LOCKED` for queue processing
- Connection pool via `pgxpool.Pool`
- UNLOGGED tables for high-frequency ephemeral data (MTProto sessions)

## Concurrency

- `errgroup` for bounded fan-out within a request
- `oklog/run` for service-level goroutine lifecycle
- `sync.Pool` for high-allocation-rate buffers (batch serialization)
- Token bucket rate limiters for external API calls

## HTTP Server

Use `net/http` — no frameworks (Gin, Echo, Chi):

```go
mux := http.NewServeMux()
mux.Handle("POST /webhook/telegram", handler)
mux.HandleFunc("GET /health", healthHandler)
```

## File Naming

- Plural for handlers: `handlers.go`, `messages.go`
- Singular for single-concern: `adapter.go`, `client.go`
- Test files: `*_test.go` with table-driven tests
- No `interfaces.go` files

## Testing

- Table-driven tests with `t.Run()` sub-tests
- Race detection: `go test -race ./...`
- Coverage: `go test -cover ./...`
- Mocks via interfaces (no code generation unless necessary)
- Test fixtures in `testdata/` directories
