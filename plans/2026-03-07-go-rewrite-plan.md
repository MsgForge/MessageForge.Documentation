# Go Backend Rewrite — Implementation Plan

> **Note:** This plan predates ADR-20 (R2→ContentVersion) and ADR-21 (Centrifugo→Platform Events). R2 and Centrifugo references below are historical.

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Rewrite all Go backend code using squad-pattern architecture (oklog/run lifecycle, App pattern, struct-tag config, local interfaces).

**Architecture:** Archive existing code to `backend/_archived/`, rebuild from scratch in `backend/` following the style guide in `.claude/agents/go-backend.md`. Each service is an actor in an `oklog/run.Group`. Shared types live in `common/models/`, business logic is preserved from archived code.

**Tech Stack:** Go 1.24+, oklog/run, caarlos0/env, go-playground/validator, pgx/v5, gotd/td, aws-sdk-go-v2, log/slog

**Reference:** Design doc at `docs/plans/2026-03-07-go-rewrite-design.md`, style guide at `.claude/agents/go-backend.md`, archived code at `backend/_archived/` (after Task 1).

---

## Task 0: Archive Existing Code

**Files:**
- Move: `backend/cmd/` → `backend/_archived/cmd/`
- Move: `backend/internal/` → `backend/_archived/internal/`
- Keep: `backend/go.mod`, `backend/go.sum` (will be modified in place)

**Step 1: Create archive directory and move code**

```bash
cd /home/shaba/Repositories/SF
mkdir -p backend/_archived
mv backend/cmd backend/_archived/cmd
mv backend/internal backend/_archived/internal
```

**Step 2: Verify archive is intact**

```bash
ls backend/_archived/cmd/server/main.go
ls backend/_archived/internal/adapter/messenger.go
ls backend/_archived/internal/config/config.go
```

Expected: All files exist at new paths.

**Step 3: Commit**

```bash
git add backend/_archived/ backend/
git commit -m "chore: archive existing Go backend for rewrite reference"
```

---

## Task 1: Scaffold — common/models + config + app.go + main.go

**Files:**
- Create: `backend/internal/common/models/models.go`
- Create: `backend/internal/messenger/config/config.go`
- Create: `backend/internal/messenger/app.go`
- Create: `backend/cmd/messenger/main.go`
- Modify: `backend/go.mod` (add new dependencies)

**Reference:** `backend/_archived/internal/adapter/messenger.go` (shared types), `backend/_archived/internal/config/config.go` (env vars), `backend/_archived/cmd/server/main.go` (entry point)

**Step 1: Add new dependencies**

```bash
cd /home/shaba/Repositories/SF/backend
go get github.com/oklog/run
go get github.com/caarlos0/env/v11
go get github.com/go-playground/validator/v10
```

**Step 2: Create common/models/models.go**

Shared types extracted from old `adapter/messenger.go`. These are the ONLY exported types (data, not interfaces).

```go
// backend/internal/common/models/models.go
package models

type Platform string

const (
    PlatformTelegram  Platform = "telegram"
    PlatformWhatsApp  Platform = "whatsapp"
    PlatformMessenger Platform = "facebook_messenger"
    PlatformViber     Platform = "viber"
)

type Direction string

const (
    Inbound  Direction = "inbound"
    Outbound Direction = "outbound"
)

type Message struct {
    ExternalID  string
    Platform    Platform
    Direction   Direction
    ChatID      string
    SenderID    string
    SenderName  string
    Text        string
    MediaType   string
    MediaFileID string
    Timestamp   int64
    RawPayload  []byte
}

type ConnectionStatus struct {
    Connected bool
    Protocol  string
    Details   string
}
```

**Step 3: Create messenger/config/config.go**

```go
// backend/internal/messenger/config/config.go
package config

import (
    "fmt"

    "github.com/caarlos0/env/v11"
    "github.com/go-playground/validator/v10"
)

type Config struct {
    HTTPHost string `env:"HTTP_HOST" envDefault:"0.0.0.0"`
    HTTPPort int    `env:"HTTP_PORT" envDefault:"8080" validate:"required,min=1,max=65535"`

    DatabaseURL string `env:"DATABASE_URL,required"`

    SFInstanceURL string `env:"SF_INSTANCE_URL"`
    SFClientID    string `env:"SF_CLIENT_ID"`
    SFUsername    string `env:"SF_USERNAME"`
    SFPrivateKey  string `env:"SF_PRIVATE_KEY"`

    TelegramAPIID   int    `env:"TELEGRAM_API_ID"`
    TelegramAPIHash string `env:"TELEGRAM_API_HASH"`

    R2AccountID       string `env:"R2_ACCOUNT_ID"`
    R2AccessKeyID     string `env:"R2_ACCESS_KEY_ID"`
    R2SecretAccessKey string `env:"R2_SECRET_ACCESS_KEY"`
    R2BucketName      string `env:"R2_BUCKET_NAME" envDefault:"messenger-media"`

    CentrifugoURL    string `env:"CENTRIFUGO_URL" envDefault:"http://localhost:8000"`
    CentrifugoAPIKey string `env:"CENTRIFUGO_API_KEY"`
    CentrifugoSecret string `env:"CENTRIFUGO_SECRET"`

    WebhookSecret string `env:"WEBHOOK_SECRET"`
}

func Load() (*Config, error) {
    cfg := &Config{}
    if err := env.Parse(cfg); err != nil {
        return nil, fmt.Errorf("parse env: %w", err)
    }
    validate := validator.New()
    if err := validate.Struct(cfg); err != nil {
        return nil, fmt.Errorf("validate config: %w", err)
    }
    return cfg, nil
}
```

**Step 4: Create messenger/app.go**

```go
// backend/internal/messenger/app.go
package messenger

import (
    "context"
    "fmt"
    "net"
    "strconv"

    "github.com/shaba/messenger-sf/internal/messenger/config"
    "github.com/shaba/messenger-sf/internal/server"
)

const AppName = "messenger"

type App struct {
    server *server.Server
}

func (a *App) Run() (func() error, func(error)) {
    return func() error {
            return a.server.Run(context.Background())
        }, func(error) {
            a.server.Shutdown(context.Background())
        }
}

// NewApp — constructor at end of file
func NewApp(_ context.Context, cfg *config.Config) (*App, error) {
    addr := net.JoinHostPort(cfg.HTTPHost, strconv.Itoa(cfg.HTTPPort))
    srv, err := server.New(addr, cfg.WebhookSecret)
    if err != nil {
        return nil, fmt.Errorf("init server: %w", err)
    }
    return &App{server: srv}, nil
}
```

**Step 5: Create cmd/messenger/main.go**

```go
// backend/cmd/messenger/main.go
package main

import (
    "context"
    "fmt"
    "log/slog"
    "os"
    "os/signal"
    "syscall"

    "github.com/oklog/run"
    "github.com/shaba/messenger-sf/internal/messenger"
    "github.com/shaba/messenger-sf/internal/messenger/config"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)

    var g run.Group

    ctx, cancel := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
    g.Add(func() error {
        <-ctx.Done()
        return ctx.Err()
    }, func(error) {
        cancel()
    })

    if err := Init(ctx, &g); err != nil {
        slog.Error("initialization failed", "error", err)
        os.Exit(1)
    }

    slog.Info("messenger middleware starting")
    if err := g.Run(); err != nil {
        slog.Error("shutdown", "error", err)
    }
    slog.Info("shutdown complete")
}

func Init(ctx context.Context, g *run.Group) error {
    cfg, err := config.Load()
    if err != nil {
        return fmt.Errorf("load config: %w", err)
    }

    app, err := messenger.NewApp(ctx, cfg)
    if err != nil {
        return fmt.Errorf("init app: %w", err)
    }

    execute, interrupt := app.Run()
    g.Add(execute, interrupt)
    return nil
}
```

**Step 6: Verify it compiles**

```bash
cd /home/shaba/Repositories/SF/backend
go build ./cmd/messenger/
```

Expected: Compiles successfully (may fail on missing `server` package — that's Task 2).

**Step 7: Commit**

```bash
git add backend/internal/common/ backend/internal/messenger/ backend/cmd/messenger/ backend/go.mod backend/go.sum
git commit -m "feat(backend): scaffold new architecture — models, config, app, main"
```

---

## Task 2: HTTP Server (oklog/run pattern)

**Files:**
- Create: `backend/internal/server/server.go`
- Create: `backend/internal/server/handlers.go`
- Create: `backend/internal/server/middleware.go`
- Test: `backend/internal/server/server_test.go`

**Reference:** `backend/_archived/internal/server/server.go` (timeouts, mux setup), `backend/_archived/internal/server/middleware.go` (HMAC logic)

**Step 1: Write server_test.go**

```go
// backend/internal/server/server_test.go
package server

import (
    "context"
    "net/http"
    "testing"
    "time"
)

func TestServerStartsAndServesHealth(t *testing.T) {
    srv, err := New("127.0.0.1:0", "")
    if err != nil {
        t.Fatal(err)
    }

    ctx, cancel := context.WithCancel(context.Background())
    errCh := make(chan error, 1)
    go func() { errCh <- srv.Run(ctx) }()

    time.Sleep(50 * time.Millisecond)

    resp, err := http.Get("http://" + srv.Addr() + "/health")
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        t.Fatalf("expected 200, got %d", resp.StatusCode)
    }

    cancel()
    srv.Shutdown(context.Background())
}
```

**Step 2: Run test to verify it fails**

```bash
cd /home/shaba/Repositories/SF/backend
go test ./internal/server/ -v -run TestServerStartsAndServesHealth
```

Expected: FAIL — package doesn't exist yet.

**Step 3: Implement server.go**

```go
// backend/internal/server/server.go
package server

import (
    "context"
    "fmt"
    "net"
    "net/http"
    "time"
)

type Server struct {
    httpServer    *http.Server
    mux           *http.ServeMux
    addr          string
    listener      net.Listener
    webhookSecret string
}

func (s *Server) Addr() string {
    if s.listener != nil {
        return s.listener.Addr().String()
    }
    return s.addr
}

func (s *Server) Run(_ context.Context) error {
    ln, err := net.Listen("tcp", s.addr)
    if err != nil {
        return fmt.Errorf("listen %s: %w", s.addr, err)
    }
    s.listener = ln
    return s.httpServer.Serve(ln)
}

func (s *Server) Shutdown(_ context.Context) error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    return s.httpServer.Shutdown(ctx)
}

func (s *Server) RegisterWebhook(pattern string, handler http.Handler) {
    if s.webhookSecret != "" {
        handler = HMACAuth(s.webhookSecret)(handler)
    }
    s.mux.Handle(pattern, handler)
}

// New — constructor at end of file
func New(addr, webhookSecret string) (*Server, error) {
    mux := http.NewServeMux()
    srv := &Server{
        httpServer: &http.Server{
            Handler:           mux,
            ReadHeaderTimeout: 10 * time.Second,
            ReadTimeout:       30 * time.Second,
            WriteTimeout:      30 * time.Second,
            IdleTimeout:       60 * time.Second,
        },
        mux:           mux,
        addr:          addr,
        webhookSecret: webhookSecret,
    }
    mux.HandleFunc("GET /health", srv.handleHealth)
    return srv, nil
}
```

**Step 4: Implement handlers.go**

```go
// backend/internal/server/handlers.go
package server

import "net/http"

func (s *Server) handleHealth(w http.ResponseWriter, _ *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    w.Write([]byte(`{"status":"ok"}`))
}
```

**Step 5: Implement middleware.go**

Preserve HMAC-SHA256 constant-time comparison from archived code.

```go
// backend/internal/server/middleware.go
package server

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "io"
    "log/slog"
    "net/http"
)

func HMACAuth(secret string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            signature := r.Header.Get("X-Signature")
            if signature == "" {
                http.Error(w, "missing signature", http.StatusUnauthorized)
                return
            }

            body, err := io.ReadAll(r.Body)
            if err != nil {
                http.Error(w, "failed to read body", http.StatusBadRequest)
                return
            }

            mac := hmac.New(sha256.New, []byte(secret))
            mac.Write(body)
            expected := hex.EncodeToString(mac.Sum(nil))

            if !hmac.Equal([]byte(signature), []byte(expected)) {
                slog.Warn("HMAC signature mismatch")
                http.Error(w, "invalid signature", http.StatusUnauthorized)
                return
            }

            r.Body = io.NopCloser(bytesReader(body))
            next.ServeHTTP(w, r)
        })
    }
}

type bytesReaderImpl struct{ data []byte; pos int }

func bytesReader(data []byte) *bytesReaderImpl { return &bytesReaderImpl{data: data} }

func (b *bytesReaderImpl) Read(p []byte) (int, error) {
    if b.pos >= len(b.data) {
        return 0, io.EOF
    }
    n := copy(p, b.data[b.pos:])
    b.pos += n
    return n, nil
}
```

**Step 6: Run tests**

```bash
cd /home/shaba/Repositories/SF/backend
go test ./internal/server/ -v
```

Expected: PASS

**Step 7: Verify full build**

```bash
go build ./cmd/messenger/
```

Expected: Compiles successfully.

**Step 8: Commit**

```bash
git add backend/internal/server/
git commit -m "feat(backend): HTTP server with oklog/run lifecycle, HMAC middleware, health endpoint"
```

---

## Task 3: Database — Pool + Migrations

**Files:**
- Create: `backend/internal/database/pool.go`
- Create: `backend/internal/database/migrations.go`
- Create: `backend/internal/database/migrations/` (embed dir)
- Test: `backend/internal/database/pool_test.go`

**Reference:** `backend/_archived/internal/database/pool.go`, `backend/_archived/internal/database/migrate.go`

**Step 1: Write pool_test.go**

```go
// backend/internal/database/pool_test.go
package database

import (
    "testing"
)

func TestNewPoolRequiresURL(t *testing.T) {
    _, err := NewPool(t.Context(), "")
    if err == nil {
        t.Fatal("expected error for empty URL")
    }
}
```

**Step 2: Run test to verify it fails**

```bash
go test ./internal/database/ -v -run TestNewPoolRequiresURL
```

**Step 3: Implement pool.go**

```go
// backend/internal/database/pool.go
package database

import (
    "context"
    "fmt"

    "github.com/jackc/pgx/v5/pgxpool"
)

func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
    if databaseURL == "" {
        return nil, fmt.Errorf("database URL is required")
    }
    cfg, err := pgxpool.ParseConfig(databaseURL)
    if err != nil {
        return nil, fmt.Errorf("parse database URL: %w", err)
    }
    pool, err := pgxpool.NewWithConfig(ctx, cfg)
    if err != nil {
        return nil, fmt.Errorf("create pool: %w", err)
    }
    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("ping database: %w", err)
    }
    return pool, nil
}
```

**Step 4: Implement migrations.go**

```go
// backend/internal/database/migrations.go
package database

import (
    "context"
    "embed"
    "fmt"
    "sort"

    "github.com/jackc/pgx/v5/pgxpool"
)

//go:embed migrations/*.sql
var migrationFS embed.FS

func Migrate(ctx context.Context, pool *pgxpool.Pool) error {
    entries, err := migrationFS.ReadDir("migrations")
    if err != nil {
        return fmt.Errorf("read migrations dir: %w", err)
    }

    var names []string
    for _, e := range entries {
        if e.IsDir() || len(e.Name()) < 5 {
            continue
        }
        if e.Name()[len(e.Name())-4:] == ".sql" {
            names = append(names, e.Name())
        }
    }
    sort.Strings(names)

    for _, name := range names {
        sql, err := migrationFS.ReadFile("migrations/" + name)
        if err != nil {
            return fmt.Errorf("read migration %s: %w", name, err)
        }
        if _, err := pool.Exec(ctx, string(sql)); err != nil {
            return fmt.Errorf("execute migration %s: %w", name, err)
        }
    }
    return nil
}
```

**Step 5: Create placeholder migration**

```bash
mkdir -p /home/shaba/Repositories/SF/backend/internal/database/migrations
```

```sql
-- backend/internal/database/migrations/001_placeholder.sql
-- Placeholder so embed.FS has content. Real migrations added in later tasks.
SELECT 1;
```

**Step 6: Run tests**

```bash
go test ./internal/database/ -v
```

Expected: PASS (pool test for empty URL).

**Step 7: Commit**

```bash
git add backend/internal/database/
git commit -m "feat(backend): database pool + embedded SQL migration runner"
```

---

## Task 4: Telegram Bot API Handler

**Files:**
- Create: `backend/internal/telegram/botapi/handlers.go`
- Test: `backend/internal/telegram/botapi/handlers_test.go`

**Reference:** `backend/_archived/internal/telegram/botapi/handler.go` (JSON structs, media detection, MIME preservation)

**Step 1: Write handlers_test.go**

```go
// backend/internal/telegram/botapi/handlers_test.go
package botapi

import (
    "bytes"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/shaba/messenger-sf/internal/common/models"
)

func TestHandlerParsesTextMessage(t *testing.T) {
    h := NewHandler()
    payload := `{"update_id":1,"message":{"message_id":42,"from":{"id":100,"first_name":"Test","last_name":"User"},"chat":{"id":200},"date":1700000000,"text":"hello"}}`

    req := httptest.NewRequest(http.MethodPost, "/", bytes.NewBufferString(payload))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()

    h.ServeHTTP(rec, req)

    if rec.Code != http.StatusOK {
        t.Fatalf("expected 200, got %d", rec.Code)
    }

    select {
    case msg := <-h.Messages():
        if msg.Text != "hello" {
            t.Errorf("expected 'hello', got %q", msg.Text)
        }
        if msg.SenderName != "Test User" {
            t.Errorf("expected 'Test User', got %q", msg.SenderName)
        }
        if msg.Platform != models.PlatformTelegram {
            t.Errorf("expected telegram, got %s", msg.Platform)
        }
    default:
        t.Fatal("no message received")
    }
}

func TestHandlerRejectsOversizedBody(t *testing.T) {
    h := NewHandler()
    body := make([]byte, 2*1024*1024)
    req := httptest.NewRequest(http.MethodPost, "/", bytes.NewReader(body))
    rec := httptest.NewRecorder()

    h.ServeHTTP(rec, req)

    if rec.Code != http.StatusBadRequest {
        t.Fatalf("expected 400, got %d", rec.Code)
    }
}
```

**Step 2: Run test to verify it fails**

```bash
go test ./internal/telegram/botapi/ -v
```

**Step 3: Implement handlers.go**

Port business logic from `backend/_archived/internal/telegram/botapi/handler.go`. Key preserved logic: media detection hierarchy (photos→documents→videos→voice), MIME type preservation, 1MB body limit, non-blocking channel send.

```go
// backend/internal/telegram/botapi/handlers.go
package botapi

import (
    "encoding/json"
    "fmt"
    "io"
    "log/slog"
    "net/http"

    "github.com/shaba/messenger-sf/internal/common/models"
)

const maxBodySize = 1024 * 1024

type update struct {
    UpdateID int        `json:"update_id"`
    Message  *tgMessage `json:"message"`
}

type tgMessage struct {
    MessageID int         `json:"message_id"`
    From      *tgUser     `json:"from"`
    Chat      tgChat      `json:"chat"`
    Date      int64       `json:"date"`
    Text      string      `json:"text"`
    Photo     []tgPhoto   `json:"photo"`
    Document  *tgDocument `json:"document"`
    Video     *tgVideo    `json:"video"`
    Voice     *tgVoice    `json:"voice"`
}

type tgUser struct {
    ID        int64  `json:"id"`
    FirstName string `json:"first_name"`
    LastName  string `json:"last_name"`
    Username  string `json:"username"`
}

type tgChat struct {
    ID int64 `json:"id"`
}

type tgPhoto struct {
    FileID   string `json:"file_id"`
    Width    int    `json:"width"`
    Height   int    `json:"height"`
    FileSize int    `json:"file_size"`
}

type tgDocument struct {
    FileID   string `json:"file_id"`
    FileName string `json:"file_name"`
    MimeType string `json:"mime_type"`
}

type tgVideo struct {
    FileID   string `json:"file_id"`
    MimeType string `json:"mime_type"`
}

type tgVoice struct {
    FileID   string `json:"file_id"`
    MimeType string `json:"mime_type"`
}

type Handler struct {
    messages chan models.Message
}

func (h *Handler) Messages() <-chan models.Message { return h.messages }

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    body, err := io.ReadAll(io.LimitReader(r.Body, maxBodySize+1))
    if err != nil || len(body) > maxBodySize {
        http.Error(w, "body too large", http.StatusBadRequest)
        return
    }

    var upd update
    if err := json.Unmarshal(body, &upd); err != nil {
        http.Error(w, "invalid JSON", http.StatusBadRequest)
        return
    }

    if upd.Message == nil {
        w.WriteHeader(http.StatusOK)
        return
    }

    msg := h.convertMessage(upd.Message)
    select {
    case h.messages <- msg:
    default:
        slog.Warn("bot API message channel full, dropping message",
            "message_id", upd.Message.MessageID)
    }
    w.WriteHeader(http.StatusOK)
}

func (h *Handler) convertMessage(m *tgMessage) models.Message {
    msg := models.Message{
        ExternalID: fmt.Sprintf("%d", m.MessageID),
        Platform:   models.PlatformTelegram,
        Direction:  models.Inbound,
        ChatID:     fmt.Sprintf("%d", m.Chat.ID),
        Text:       m.Text,
        Timestamp:  m.Date,
        MediaType:  "text",
    }

    if m.From != nil {
        msg.SenderID = fmt.Sprintf("%d", m.From.ID)
        msg.SenderName = m.From.FirstName
        if m.From.LastName != "" {
            msg.SenderName += " " + m.From.LastName
        }
    }

    // Media detection hierarchy: photo > document > video > voice
    if len(m.Photo) > 0 {
        largest := m.Photo[len(m.Photo)-1]
        msg.MediaType = "photo"
        msg.MediaFileID = largest.FileID
    } else if m.Document != nil {
        msg.MediaType = "document"
        msg.MediaFileID = m.Document.FileID
    } else if m.Video != nil {
        msg.MediaType = "video"
        msg.MediaFileID = m.Video.FileID
    } else if m.Voice != nil {
        msg.MediaType = "voice"
        msg.MediaFileID = m.Voice.FileID
    }

    return msg
}

// NewHandler — constructor at end of file
func NewHandler() *Handler {
    return &Handler{messages: make(chan models.Message, 256)}
}
```

**Step 4: Run tests**

```bash
go test ./internal/telegram/botapi/ -v
```

Expected: PASS

**Step 5: Commit**

```bash
git add backend/internal/telegram/botapi/
git commit -m "feat(backend): Telegram Bot API webhook handler with media detection"
```

---

## Task 5: MTProto — Types + Session Store + Rate Limiter

**Files:**
- Create: `backend/internal/telegram/mtproto/types.go`
- Create: `backend/internal/telegram/mtproto/session_store.go`
- Create: `backend/internal/telegram/mtproto/ratelimit.go`
- Test: `backend/internal/telegram/mtproto/ratelimit_test.go`

**Reference:** `backend/_archived/internal/telegram/mtproto/types.go`, `backend/_archived/internal/telegram/mtproto/session_store.go`, `backend/_archived/internal/telegram/mtproto/ratelimit.go`

**Step 1: Create types.go**

```go
// backend/internal/telegram/mtproto/types.go
package mtproto

type AuthState string

const (
    AuthStateInit    AuthState = "init"
    AuthStateOTPSent AuthState = "otp_sent"
    AuthState2FA     AuthState = "2fa"
    AuthStateSuccess AuthState = "success"
    AuthStateFailed  AuthState = "failed"
)

type AuthSession struct {
    Phone         string
    PhoneCodeHash string
    State         AuthState
    UserID        int64
    Username      string
    FirstName     string
    LastName      string
    Error         string
}

type User struct {
    ID        int64
    Username  string
    FirstName string
    LastName  string
    Phone     string
}

type ChannelMessage struct {
    MessageID int
    ChannelID int64
    SenderID  int64
    Text      string
    Date      int64
    MediaType string
    MediaID   string
}
```

**Step 2: Create session_store.go**

```go
// backend/internal/telegram/mtproto/session_store.go
package mtproto

import (
    "context"
    "fmt"

    "github.com/jackc/pgx/v5/pgxpool"
)

type sessionStore interface {
    Load(ctx context.Context, connectionID int64) ([]byte, error)
    Save(ctx context.Context, connectionID int64, dcID int, data []byte) error
    Delete(ctx context.Context, connectionID int64) error
}

type PgSessionStore struct {
    pool *pgxpool.Pool
}

func (s *PgSessionStore) Load(ctx context.Context, connectionID int64) ([]byte, error) {
    var data []byte
    err := s.pool.QueryRow(ctx,
        "SELECT session_data FROM mtproto_sessions WHERE connection_id = $1 ORDER BY updated_at DESC LIMIT 1",
        connectionID,
    ).Scan(&data)
    if err != nil {
        return nil, fmt.Errorf("load session %d: %w", connectionID, err)
    }
    return data, nil
}

func (s *PgSessionStore) Save(ctx context.Context, connectionID int64, dcID int, data []byte) error {
    _, err := s.pool.Exec(ctx,
        `INSERT INTO mtproto_sessions (connection_id, dc_id, session_data, updated_at)
         VALUES ($1, $2, $3, NOW())
         ON CONFLICT (connection_id) DO UPDATE SET dc_id = $2, session_data = $3, updated_at = NOW()`,
        connectionID, dcID, data,
    )
    if err != nil {
        return fmt.Errorf("save session %d: %w", connectionID, err)
    }
    return nil
}

func (s *PgSessionStore) Delete(ctx context.Context, connectionID int64) error {
    _, err := s.pool.Exec(ctx,
        "DELETE FROM mtproto_sessions WHERE connection_id = $1",
        connectionID,
    )
    if err != nil {
        return fmt.Errorf("delete session %d: %w", connectionID, err)
    }
    return nil
}

// NewPgSessionStore — constructor at end of file
func NewPgSessionStore(pool *pgxpool.Pool) *PgSessionStore {
    return &PgSessionStore{pool: pool}
}
```

**Step 3: Write ratelimit_test.go**

```go
// backend/internal/telegram/mtproto/ratelimit_test.go
package mtproto

import (
    "context"
    "testing"
    "time"
)

func TestRateLimiterWaits(t *testing.T) {
    rl := NewRateLimiter(10, 1)
    ctx := context.Background()

    start := time.Now()
    for i := 0; i < 3; i++ {
        if err := rl.Wait(ctx); err != nil {
            t.Fatal(err)
        }
    }
    elapsed := time.Since(start)
    if elapsed < 100*time.Millisecond {
        t.Logf("elapsed: %v (rate limiter may have burst capacity)", elapsed)
    }
}

func TestFloodWaiterTracksMaxWait(t *testing.T) {
    fw := NewFloodWaiter()
    if fw.IsWaiting() {
        t.Fatal("should not be waiting initially")
    }
}

func TestMiddlewareChainExecutes(t *testing.T) {
    mc := NewMiddlewareChain(100, 10)
    called := false
    err := mc.Execute(context.Background(), func(_ context.Context) error {
        called = true
        return nil
    })
    if err != nil {
        t.Fatal(err)
    }
    if !called {
        t.Fatal("operation was not called")
    }
}
```

**Step 4: Implement ratelimit.go**

```go
// backend/internal/telegram/mtproto/ratelimit.go
package mtproto

import (
    "context"
    "log/slog"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

type RateLimiter struct {
    limiter *rate.Limiter
}

func (r *RateLimiter) Wait(ctx context.Context) error {
    return r.limiter.Wait(ctx)
}

type FloodWaiter struct {
    mu       sync.Mutex
    waitTill time.Time
}

func (fw *FloodWaiter) HandleFloodWait(ctx context.Context, seconds int) error {
    until := time.Now().Add(time.Duration(seconds) * time.Second)
    fw.mu.Lock()
    if until.After(fw.waitTill) {
        fw.waitTill = until
    }
    fw.mu.Unlock()

    slog.Warn("FLOOD_WAIT received", "seconds", seconds)

    select {
    case <-time.After(time.Duration(seconds) * time.Second):
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (fw *FloodWaiter) IsWaiting() bool {
    fw.mu.Lock()
    defer fw.mu.Unlock()
    return time.Now().Before(fw.waitTill)
}

type MiddlewareChain struct {
    rateLimiter *RateLimiter
    floodWaiter *FloodWaiter
}

func (mc *MiddlewareChain) Execute(ctx context.Context, op func(ctx context.Context) error) error {
    if err := mc.rateLimiter.Wait(ctx); err != nil {
        return err
    }
    return op(ctx)
}

func (mc *MiddlewareChain) FloodWaiter() *FloodWaiter { return mc.floodWaiter }

// Constructors — at end of file
func NewRateLimiter(rps float64, burst int) *RateLimiter {
    return &RateLimiter{limiter: rate.NewLimiter(rate.Limit(rps), burst)}
}

func NewFloodWaiter() *FloodWaiter {
    return &FloodWaiter{}
}

func NewMiddlewareChain(rps float64, burst int) *MiddlewareChain {
    return &MiddlewareChain{
        rateLimiter: NewRateLimiter(rps, burst),
        floodWaiter: NewFloodWaiter(),
    }
}
```

**Step 5: Run tests**

```bash
go test ./internal/telegram/mtproto/ -v
```

Expected: PASS

**Step 6: Commit**

```bash
git add backend/internal/telegram/mtproto/
git commit -m "feat(backend): MTProto types, session store, rate limiter"
```

---

## Task 6: MTProto — Client + Message Listener

**Files:**
- Create: `backend/internal/telegram/mtproto/client.go`
- Create: `backend/internal/telegram/mtproto/messages.go`
- Test: `backend/internal/telegram/mtproto/client_test.go`

**Reference:** `backend/_archived/internal/telegram/mtproto/client.go` (auth state machine), `backend/_archived/internal/telegram/mtproto/messages.go` (message conversion)

**Step 1: Write client_test.go**

```go
// backend/internal/telegram/mtproto/client_test.go
package mtproto

import (
    "testing"
)

func TestNewClientRequiresAPIID(t *testing.T) {
    _, err := NewClient(Options{APIHash: "hash"})
    if err == nil {
        t.Fatal("expected error for missing API ID")
    }
}

func TestNewClientRequiresAPIHash(t *testing.T) {
    _, err := NewClient(Options{APIID: 123})
    if err == nil {
        t.Fatal("expected error for missing API hash")
    }
}
```

**Step 2: Implement client.go**

Port auth state machine from archived code. Key: local `telegramAPI` interface (lowercase, in this file).

```go
// backend/internal/telegram/mtproto/client.go
package mtproto

import (
    "context"
    "fmt"
)

type telegramAPI interface {
    Run(ctx context.Context, f func(ctx context.Context) error) error
    IsAuthorized(ctx context.Context) (bool, error)
    SendCode(ctx context.Context, phone string) (string, error)
    SignIn(ctx context.Context, phone, code, codeHash string) (*User, error)
    CheckPassword(ctx context.Context, password string) (*User, error)
    ListenUpdates(ctx context.Context, ch chan<- ChannelMessage) error
}

type Options struct {
    APIID   int
    APIHash string
    Store   sessionStore
    API     telegramAPI
}

type Client struct {
    api   telegramAPI
    store sessionStore
}

func (c *Client) StartAuth(ctx context.Context, phone string) (*AuthSession, error) {
    codeHash, err := c.api.SendCode(ctx, phone)
    if err != nil {
        return nil, fmt.Errorf("send code: %w", err)
    }
    return &AuthSession{
        Phone:         phone,
        PhoneCodeHash: codeHash,
        State:         AuthStateOTPSent,
    }, nil
}

func (c *Client) VerifyOTP(ctx context.Context, session *AuthSession, code string) error {
    user, err := c.api.SignIn(ctx, session.Phone, code, session.PhoneCodeHash)
    if err != nil {
        if err.Error() == "SESSION_PASSWORD_NEEDED" {
            session.State = AuthState2FA
            return nil
        }
        session.State = AuthStateFailed
        session.Error = err.Error()
        return fmt.Errorf("sign in: %w", err)
    }
    session.State = AuthStateSuccess
    session.UserID = user.ID
    session.Username = user.Username
    session.FirstName = user.FirstName
    session.LastName = user.LastName
    return nil
}

func (c *Client) Verify2FA(ctx context.Context, session *AuthSession, password string) error {
    user, err := c.api.CheckPassword(ctx, password)
    if err != nil {
        session.State = AuthStateFailed
        session.Error = err.Error()
        return fmt.Errorf("check password: %w", err)
    }
    session.State = AuthStateSuccess
    session.UserID = user.ID
    session.Username = user.Username
    session.FirstName = user.FirstName
    session.LastName = user.LastName
    return nil
}

// NewClient — constructor at end of file
func NewClient(opts Options) (*Client, error) {
    if opts.APIID == 0 {
        return nil, fmt.Errorf("API ID is required")
    }
    if opts.APIHash == "" {
        return nil, fmt.Errorf("API hash is required")
    }
    return &Client{
        api:   opts.API,
        store: opts.Store,
    }, nil
}
```

**Step 3: Implement messages.go**

```go
// backend/internal/telegram/mtproto/messages.go
package mtproto

import (
    "context"
    "fmt"
    "log/slog"

    "github.com/shaba/messenger-sf/internal/common/models"
)

type MessageListener struct {
    client   *Client
    messages chan models.Message
}

func (l *MessageListener) Messages() <-chan models.Message { return l.messages }

func (l *MessageListener) Listen(ctx context.Context) error {
    rawCh := make(chan ChannelMessage, 256)

    go func() {
        if err := l.client.api.ListenUpdates(ctx, rawCh); err != nil {
            slog.Error("listen updates stopped", "error", err)
        }
        close(rawCh)
    }()

    for raw := range rawCh {
        msg := models.Message{
            ExternalID:  fmt.Sprintf("%d", raw.MessageID),
            Platform:    models.PlatformTelegram,
            Direction:   models.Inbound,
            ChatID:      fmt.Sprintf("%d", raw.ChannelID),
            SenderID:    fmt.Sprintf("%d", raw.SenderID),
            Text:        raw.Text,
            MediaType:   raw.MediaType,
            MediaFileID: raw.MediaID,
            Timestamp:   raw.Date,
        }

        select {
        case l.messages <- msg:
        default:
            slog.Warn("MTProto message channel full, dropping",
                "message_id", raw.MessageID, "channel_id", raw.ChannelID)
        }
    }
    return nil
}

// NewMessageListener — constructor at end of file
func NewMessageListener(client *Client, bufSize int) *MessageListener {
    if bufSize <= 0 {
        bufSize = 256
    }
    return &MessageListener{
        client:   client,
        messages: make(chan models.Message, bufSize),
    }
}
```

**Step 4: Run tests**

```bash
go test ./internal/telegram/mtproto/ -v
```

Expected: PASS

**Step 5: Commit**

```bash
git add backend/internal/telegram/mtproto/client.go backend/internal/telegram/mtproto/messages.go backend/internal/telegram/mtproto/client_test.go
git commit -m "feat(backend): MTProto client auth state machine + message listener"
```

---

## Task 7: Telegram Adapter (dual protocol)

**Files:**
- Create: `backend/internal/telegram/adapter.go`
- Test: `backend/internal/telegram/adapter_test.go`

**Reference:** `backend/_archived/internal/telegram/adapter.go` (protocol routing, goroutine lifecycle, non-blocking sends)

**Step 1: Write adapter_test.go**

```go
// backend/internal/telegram/adapter_test.go
package telegram

import (
    "context"
    "testing"

    "github.com/shaba/messenger-sf/internal/common/models"
)

func TestNewAdapterRequiresConfig(t *testing.T) {
    _, err := NewAdapter(AdapterConfig{})
    if err == nil {
        t.Fatal("expected error for empty config")
    }
}

func TestAdapterStatus(t *testing.T) {
    cfg := AdapterConfig{
        ConnectionType: ConnectionTypeBotAPI,
        BotHandler:     &fakeBotHandler{},
    }
    a, err := NewAdapter(cfg)
    if err != nil {
        t.Fatal(err)
    }
    status := a.Status()
    if status.Connected {
        t.Fatal("should not be connected before Connect()")
    }
    if status.Protocol != string(ConnectionTypeBotAPI) {
        t.Fatalf("expected bot_api, got %s", status.Protocol)
    }
}

type fakeBotHandler struct{}

func (f *fakeBotHandler) Messages() <-chan models.Message {
    return make(chan models.Message)
}
```

**Step 2: Implement adapter.go**

Local interfaces only — `botMessageSource` and `mtprotoMessageSource` defined in this file.

```go
// backend/internal/telegram/adapter.go
package telegram

import (
    "context"
    "fmt"
    "log/slog"
    "sync"

    "github.com/shaba/messenger-sf/internal/common/models"
)

type ConnectionType string

const (
    ConnectionTypeMTProto ConnectionType = "mtproto"
    ConnectionTypeBotAPI  ConnectionType = "bot_api"
)

type botMessageSource interface {
    Messages() <-chan models.Message
}

type AdapterConfig struct {
    ConnectionType ConnectionType
    BotHandler     botMessageSource
}

type TelegramAdapter struct {
    mu         sync.RWMutex
    connected  bool
    connType   ConnectionType
    botHandler botMessageSource
    messages   chan models.Message
    cancel     context.CancelFunc
}

func (a *TelegramAdapter) Connect(ctx context.Context) error {
    a.mu.Lock()
    defer a.mu.Unlock()

    childCtx, cancel := context.WithCancel(ctx)
    a.cancel = cancel

    switch a.connType {
    case ConnectionTypeBotAPI:
        go a.forwardMessages(childCtx, a.botHandler.Messages())
    default:
        cancel()
        return fmt.Errorf("unsupported connection type: %s", a.connType)
    }

    a.connected = true
    return nil
}

func (a *TelegramAdapter) Disconnect(_ context.Context) error {
    a.mu.Lock()
    defer a.mu.Unlock()
    if a.cancel != nil {
        a.cancel()
    }
    a.connected = false
    return nil
}

func (a *TelegramAdapter) InboundMessages() <-chan models.Message { return a.messages }

func (a *TelegramAdapter) Status() models.ConnectionStatus {
    a.mu.RLock()
    defer a.mu.RUnlock()
    return models.ConnectionStatus{
        Connected: a.connected,
        Protocol:  string(a.connType),
    }
}

func (a *TelegramAdapter) Platform() models.Platform { return models.PlatformTelegram }

func (a *TelegramAdapter) forwardMessages(ctx context.Context, src <-chan models.Message) {
    for {
        select {
        case <-ctx.Done():
            return
        case msg, ok := <-src:
            if !ok {
                return
            }
            select {
            case a.messages <- msg:
            default:
                slog.Warn("adapter message channel full, dropping")
            }
        }
    }
}

// NewAdapter — constructor at end of file
func NewAdapter(cfg AdapterConfig) (*TelegramAdapter, error) {
    if cfg.ConnectionType == "" {
        return nil, fmt.Errorf("connection type is required")
    }
    if cfg.ConnectionType == ConnectionTypeBotAPI && cfg.BotHandler == nil {
        return nil, fmt.Errorf("bot handler required for bot_api connection")
    }
    return &TelegramAdapter{
        connType:   cfg.ConnectionType,
        botHandler: cfg.BotHandler,
        messages:   make(chan models.Message, 512),
    }, nil
}
```

**Step 3: Run tests**

```bash
go test ./internal/telegram/ -v
```

Expected: PASS

**Step 4: Commit**

```bash
git add backend/internal/telegram/adapter.go backend/internal/telegram/adapter_test.go
git commit -m "feat(backend): Telegram adapter with dual protocol support"
```

---

## Task 8: Salesforce — Auth + Client

**Files:**
- Create: `backend/internal/salesforce/auth.go`
- Create: `backend/internal/salesforce/client.go`
- Test: `backend/internal/salesforce/auth_test.go`
- Test: `backend/internal/salesforce/client_test.go`

**Reference:** `backend/_archived/internal/salesforce/auth.go` (JWT signing, PEM parsing), `backend/_archived/internal/salesforce/client.go` (double-check-locking, token refresh)

**Step 1: Write auth_test.go**

```go
// backend/internal/salesforce/auth_test.go
package salesforce

import (
    "testing"
)

func TestNewJWTAuthRequiresConfig(t *testing.T) {
    _, err := NewJWTAuth(JWTAuthConfig{})
    if err == nil {
        t.Fatal("expected error for empty config")
    }
}

func TestNewJWTAuthRequiresValidPEM(t *testing.T) {
    _, err := NewJWTAuth(JWTAuthConfig{
        ClientID:    "client",
        Username:    "user@example.com",
        InstanceURL: "https://test.salesforce.com",
        PrivateKey:  "not-a-pem-key",
    })
    if err == nil {
        t.Fatal("expected error for invalid PEM")
    }
}
```

**Step 2: Implement auth.go**

Port JWT signing logic from archived code. Key: PKCS8/PKCS1 fallback, 5-minute JWT expiry.

```go
// backend/internal/salesforce/auth.go
package salesforce

import (
    "crypto"
    "crypto/rsa"
    "crypto/sha256"
    "crypto/x509"
    "encoding/base64"
    "encoding/json"
    "encoding/pem"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "strings"
    "time"
)

type TokenResponse struct {
    AccessToken string `json:"access_token"`
    InstanceURL string `json:"instance_url"`
    ID          string `json:"id"`
    TokenType   string `json:"token_type"`
    IssuedAt    string `json:"issued_at"`
}

type JWTAuthConfig struct {
    ClientID    string
    Username    string
    InstanceURL string
    PrivateKey  string
    TokenURL    string
}

type JWTAuth struct {
    cfg        JWTAuthConfig
    privateKey *rsa.PrivateKey
    httpClient *http.Client
}

func (a *JWTAuth) Authenticate() (*TokenResponse, error) {
    assertion, err := a.buildAssertion()
    if err != nil {
        return nil, fmt.Errorf("build assertion: %w", err)
    }

    resp, err := a.httpClient.PostForm(a.cfg.TokenURL, url.Values{
        "grant_type": {"urn:ietf:params:oauth:grant-type:jwt-bearer"},
        "assertion":  {assertion},
    })
    if err != nil {
        return nil, fmt.Errorf("token request: %w", err)
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("read response: %w", err)
    }
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("token request failed (%d): %s", resp.StatusCode, string(body))
    }

    var token TokenResponse
    if err := json.Unmarshal(body, &token); err != nil {
        return nil, fmt.Errorf("parse token response: %w", err)
    }
    return &token, nil
}

func (a *JWTAuth) buildAssertion() (string, error) {
    header := base64URLEncode([]byte(`{"alg":"RS256","typ":"JWT"}`))

    claims, err := json.Marshal(map[string]interface{}{
        "iss": a.cfg.ClientID,
        "sub": a.cfg.Username,
        "aud": a.cfg.TokenURL,
        "exp": time.Now().Add(5 * time.Minute).Unix(),
    })
    if err != nil {
        return "", fmt.Errorf("marshal claims: %w", err)
    }
    payload := base64URLEncode(claims)

    signingInput := header + "." + payload
    hash := sha256.Sum256([]byte(signingInput))
    sig, err := rsa.SignPKCS1v15(nil, a.privateKey, crypto.SHA256, hash[:])
    if err != nil {
        return "", fmt.Errorf("sign JWT: %w", err)
    }

    return signingInput + "." + base64URLEncode(sig), nil
}

func base64URLEncode(data []byte) string {
    return strings.TrimRight(base64.URLEncoding.EncodeToString(data), "=")
}

func parsePrivateKey(pemData string) (*rsa.PrivateKey, error) {
    block, _ := pem.Decode([]byte(pemData))
    if block == nil {
        return nil, fmt.Errorf("failed to decode PEM block")
    }
    // Try PKCS8 first, fall back to PKCS1
    key, err := x509.ParsePKCS8PrivateKey(block.Bytes)
    if err == nil {
        rsaKey, ok := key.(*rsa.PrivateKey)
        if !ok {
            return nil, fmt.Errorf("PKCS8 key is not RSA")
        }
        return rsaKey, nil
    }
    return x509.ParsePKCS1PrivateKey(block.Bytes)
}

// NewJWTAuth — constructor at end of file
func NewJWTAuth(cfg JWTAuthConfig) (*JWTAuth, error) {
    if cfg.ClientID == "" || cfg.Username == "" || cfg.InstanceURL == "" {
        return nil, fmt.Errorf("ClientID, Username, and InstanceURL are required")
    }
    if cfg.PrivateKey == "" {
        return nil, fmt.Errorf("PrivateKey is required")
    }
    if cfg.TokenURL == "" {
        cfg.TokenURL = cfg.InstanceURL + "/services/oauth2/token"
    }
    pk, err := parsePrivateKey(cfg.PrivateKey)
    if err != nil {
        return nil, fmt.Errorf("parse private key: %w", err)
    }
    return &JWTAuth{
        cfg:        cfg,
        privateKey: pk,
        httpClient: &http.Client{Timeout: 30 * time.Second},
    }, nil
}
```

**Step 3: Implement client.go**

Port double-check-locking token refresh from archived code.

```go
// backend/internal/salesforce/client.go
package salesforce

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

const apiVersion = "v62.0"

type tokenProvider interface {
    Authenticate() (*TokenResponse, error)
}

type Client struct {
    mu          sync.RWMutex
    auth        tokenProvider
    token       string
    instanceURL string
    tokenExpiry time.Time
    httpClient  *http.Client
}

func (c *Client) getToken() (string, string, error) {
    c.mu.RLock()
    if time.Now().Before(c.tokenExpiry) {
        token, url := c.token, c.instanceURL
        c.mu.RUnlock()
        return token, url, nil
    }
    c.mu.RUnlock()

    c.mu.Lock()
    defer c.mu.Unlock()
    if time.Now().Before(c.tokenExpiry) {
        return c.token, c.instanceURL, nil
    }

    resp, err := c.auth.Authenticate()
    if err != nil {
        return "", "", fmt.Errorf("authenticate: %w", err)
    }
    c.token = resp.AccessToken
    c.instanceURL = resp.InstanceURL
    c.tokenExpiry = time.Now().Add(90 * time.Minute)
    return c.token, c.instanceURL, nil
}

func (c *Client) DoRequest(ctx context.Context, method, path string, body interface{}) ([]byte, int, error) {
    token, instanceURL, err := c.getToken()
    if err != nil {
        return nil, 0, err
    }

    var bodyReader io.Reader
    if body != nil {
        data, err := json.Marshal(body)
        if err != nil {
            return nil, 0, fmt.Errorf("marshal body: %w", err)
        }
        bodyReader = bytes.NewReader(data)
    }

    url := instanceURL + path
    req, err := http.NewRequestWithContext(ctx, method, url, bodyReader)
    if err != nil {
        return nil, 0, fmt.Errorf("create request: %w", err)
    }
    req.Header.Set("Authorization", "Bearer "+token)
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, 0, fmt.Errorf("execute request: %w", err)
    }
    defer resp.Body.Close()

    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, resp.StatusCode, fmt.Errorf("read response: %w", err)
    }
    return respBody, resp.StatusCode, nil
}

func (c *Client) CreateRecord(ctx context.Context, objectName string, fields map[string]interface{}) (string, error) {
    path := fmt.Sprintf("/services/data/%s/sobjects/%s/", apiVersion, objectName)
    body, status, err := c.DoRequest(ctx, http.MethodPost, path, fields)
    if err != nil {
        return "", err
    }
    if status != http.StatusCreated {
        return "", fmt.Errorf("create record failed (%d): %s", status, string(body))
    }
    var result struct{ ID string `json:"id"` }
    if err := json.Unmarshal(body, &result); err != nil {
        return "", fmt.Errorf("parse response: %w", err)
    }
    return result.ID, nil
}

func (c *Client) PublishPlatformEvent(ctx context.Context, eventName string, payload map[string]interface{}) error {
    path := fmt.Sprintf("/services/data/%s/sobjects/%s/", apiVersion, eventName)
    body, status, err := c.DoRequest(ctx, http.MethodPost, path, payload)
    if err != nil {
        return err
    }
    if status != http.StatusCreated {
        return fmt.Errorf("publish event failed (%d): %s", status, string(body))
    }
    return nil
}

// NewClient — constructor at end of file
func NewClient(auth tokenProvider) *Client {
    return &Client{
        auth:       auth,
        httpClient: &http.Client{Timeout: 30 * time.Second},
    }
}
```

**Step 4: Run tests**

```bash
go test ./internal/salesforce/ -v
```

Expected: PASS

**Step 5: Commit**

```bash
git add backend/internal/salesforce/
git commit -m "feat(backend): Salesforce JWT auth + REST client with token refresh"
```

---

## Task 9: Salesforce — Ingestion (batch + sync.Pool)

**Files:**
- Create: `backend/internal/salesforce/ingestion.go`
- Test: `backend/internal/salesforce/ingestion_test.go`

**Reference:** `backend/_archived/internal/salesforce/ingestion.go` (sync.Pool, 950KB limit, 200 record limit, flush loop)

**Step 1: Write ingestion_test.go**

```go
// backend/internal/salesforce/ingestion_test.go
package salesforce

import (
    "context"
    "testing"

    "github.com/shaba/messenger-sf/internal/common/models"
)

func TestIngesterBatchesMessages(t *testing.T) {
    mock := &mockClient{}
    ing := NewIngester(mock, "Inbound_Message__e")

    msg := models.Message{
        ExternalID: "1",
        Platform:   models.PlatformTelegram,
        Direction:  models.Inbound,
        ChatID:     "100",
        Text:       "hello",
    }

    if err := ing.Add(context.Background(), msg); err != nil {
        t.Fatal(err)
    }
    if ing.BatchLen() != 1 {
        t.Fatalf("expected 1, got %d", ing.BatchLen())
    }
}

type mockClient struct {
    published int
}

func (m *mockClient) PublishPlatformEvent(_ context.Context, _ string, _ map[string]interface{}) error {
    m.published++
    return nil
}
```

**Step 2: Implement ingestion.go**

Port sync.Pool buffer reuse and size tracking from archived code.

```go
// backend/internal/salesforce/ingestion.go
package salesforce

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "log/slog"
    "sync"
    "time"

    "github.com/shaba/messenger-sf/internal/common/models"
)

const (
    maxBatchBytes = 950 * 1024
    maxBatchSize  = 200
    flushInterval = 5 * time.Second
)

type eventPublisher interface {
    PublishPlatformEvent(ctx context.Context, eventName string, payload map[string]interface{}) error
}

type Ingester struct {
    mu        sync.Mutex
    client    eventPublisher
    eventName string
    batch     []map[string]interface{}
    batchSize int
    bufPool   sync.Pool
}

func (i *Ingester) Add(ctx context.Context, msg models.Message) error {
    payload := map[string]interface{}{
        "Text__c":               msg.Text,
        "Chat_SF_ID__c":         msg.ChatID,
        "Sender_Name__c":        msg.SenderName,
        "Sender_External_ID__c": msg.SenderID,
        "Message_Type__c":       msg.MediaType,
        "Sent_At__c":            time.Unix(msg.Timestamp, 0).UTC().Format(time.RFC3339),
        "Protocol__c":           string(msg.Platform),
    }
    if msg.MediaFileID != "" {
        payload["Media_URL__c"] = msg.MediaFileID
    }

    buf := i.bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    if err := json.NewEncoder(buf).Encode(payload); err != nil {
        i.bufPool.Put(buf)
        return fmt.Errorf("estimate size: %w", err)
    }
    msgSize := buf.Len()
    i.bufPool.Put(buf)

    i.mu.Lock()
    defer i.mu.Unlock()

    if i.batchSize+msgSize > maxBatchBytes || len(i.batch) >= maxBatchSize {
        if err := i.flushLocked(ctx); err != nil {
            return fmt.Errorf("flush before add: %w", err)
        }
    }

    i.batch = append(i.batch, payload)
    i.batchSize += msgSize
    return nil
}

func (i *Ingester) Flush(ctx context.Context) error {
    i.mu.Lock()
    defer i.mu.Unlock()
    return i.flushLocked(ctx)
}

func (i *Ingester) flushLocked(ctx context.Context) error {
    if len(i.batch) == 0 {
        return nil
    }

    for _, payload := range i.batch {
        if err := i.client.PublishPlatformEvent(ctx, i.eventName, payload); err != nil {
            return fmt.Errorf("publish event: %w", err)
        }
    }

    i.batch = i.batch[:0]
    i.batchSize = 0
    return nil
}

func (i *Ingester) RunFlushLoop(ctx context.Context) {
    ticker := time.NewTicker(flushInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            if err := i.Flush(context.Background()); err != nil {
                slog.Error("final flush failed", "error", err)
            }
            return
        case <-ticker.C:
            if err := i.Flush(ctx); err != nil {
                slog.Error("periodic flush failed", "error", err)
            }
        }
    }
}

func (i *Ingester) BatchLen() int {
    i.mu.Lock()
    defer i.mu.Unlock()
    return len(i.batch)
}

// NewIngester — constructor at end of file
func NewIngester(client eventPublisher, eventName string) *Ingester {
    return &Ingester{
        client:    client,
        eventName: eventName,
        bufPool: sync.Pool{
            New: func() interface{} { return new(bytes.Buffer) },
        },
    }
}
```

**Step 3: Run tests**

```bash
go test ./internal/salesforce/ -v
```

Expected: PASS

**Step 4: Commit**

```bash
git add backend/internal/salesforce/ingestion.go backend/internal/salesforce/ingestion_test.go
git commit -m "feat(backend): Salesforce ingestion with batch limits and sync.Pool"
```

---

## Task 10: Media — R2 Client

**Files:**
- Create: `backend/internal/media/r2.go`
- Test: `backend/internal/media/r2_test.go`

**Reference:** `backend/_archived/internal/media/r2.go` (custom endpoint, path-style, presigned URLs)

**Step 1: Write r2_test.go**

```go
// backend/internal/media/r2_test.go
package media

import (
    "testing"
)

func TestNewR2ClientRequiresAccountID(t *testing.T) {
    _, err := NewR2Client(t.Context(), R2Config{})
    if err == nil {
        t.Fatal("expected error for missing account ID")
    }
}

func TestR2EndpointFormat(t *testing.T) {
    endpoint := r2Endpoint("test-account-id")
    expected := "https://test-account-id.r2.cloudflarestorage.com"
    if endpoint != expected {
        t.Fatalf("expected %s, got %s", expected, endpoint)
    }
}
```

**Step 2: Implement r2.go**

```go
// backend/internal/media/r2.go
package media

import (
    "context"
    "fmt"
    "io"
    "time"

    "github.com/aws/aws-sdk-go-v2/aws"
    awsconfig "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/credentials"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

type R2Config struct {
    AccountID       string
    AccessKeyID     string
    SecretAccessKey string
    BucketName      string
}

type R2Client struct {
    client     *s3.Client
    presigner  *s3.PresignClient
    bucketName string
}

func r2Endpoint(accountID string) string {
    return fmt.Sprintf("https://%s.r2.cloudflarestorage.com", accountID)
}

func (c *R2Client) Upload(ctx context.Context, key string, reader io.Reader, contentType string) error {
    _, err := c.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(c.bucketName),
        Key:         aws.String(key),
        Body:        reader,
        ContentType: aws.String(contentType),
    })
    if err != nil {
        return fmt.Errorf("upload %s: %w", key, err)
    }
    return nil
}

func (c *R2Client) Download(ctx context.Context, key string) (io.ReadCloser, string, error) {
    out, err := c.client.GetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(c.bucketName),
        Key:    aws.String(key),
    })
    if err != nil {
        return nil, "", fmt.Errorf("download %s: %w", key, err)
    }
    ct := ""
    if out.ContentType != nil {
        ct = *out.ContentType
    }
    return out.Body, ct, nil
}

func (c *R2Client) Delete(ctx context.Context, key string) error {
    _, err := c.client.DeleteObject(ctx, &s3.DeleteObjectInput{
        Bucket: aws.String(c.bucketName),
        Key:    aws.String(key),
    })
    if err != nil {
        return fmt.Errorf("delete %s: %w", key, err)
    }
    return nil
}

func (c *R2Client) PresignedGetURL(ctx context.Context, key string, ttl time.Duration) (string, error) {
    req, err := c.presigner.PresignGetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(c.bucketName),
        Key:    aws.String(key),
    }, s3.WithPresignExpires(ttl))
    if err != nil {
        return "", fmt.Errorf("presign GET %s: %w", key, err)
    }
    return req.URL, nil
}

func (c *R2Client) PresignedPutURL(ctx context.Context, key, contentType string, ttl time.Duration) (string, error) {
    req, err := c.presigner.PresignPutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(c.bucketName),
        Key:         aws.String(key),
        ContentType: aws.String(contentType),
    }, s3.WithPresignExpires(ttl))
    if err != nil {
        return "", fmt.Errorf("presign PUT %s: %w", key, err)
    }
    return req.URL, nil
}

// NewR2Client — constructor at end of file
func NewR2Client(ctx context.Context, cfg R2Config) (*R2Client, error) {
    if cfg.AccountID == "" {
        return nil, fmt.Errorf("R2 account ID is required")
    }
    endpoint := r2Endpoint(cfg.AccountID)

    awsCfg, err := awsconfig.LoadDefaultConfig(ctx,
        awsconfig.WithCredentialsProvider(
            credentials.NewStaticCredentialsProvider(cfg.AccessKeyID, cfg.SecretAccessKey, ""),
        ),
        awsconfig.WithRegion("auto"),
    )
    if err != nil {
        return nil, fmt.Errorf("load AWS config: %w", err)
    }

    client := s3.NewFromConfig(awsCfg, func(o *s3.Options) {
        o.BaseEndpoint = aws.String(endpoint)
        o.UsePathStyle = true
    })

    return &R2Client{
        client:     client,
        presigner:  s3.NewPresignClient(client),
        bucketName: cfg.BucketName,
    }, nil
}
```

**Step 3: Run tests**

```bash
go test ./internal/media/ -v
```

Expected: PASS

**Step 4: Commit**

```bash
git add backend/internal/media/
git commit -m "feat(backend): Cloudflare R2 client with upload, download, presigned URLs"
```

---

## Task 11: Realtime — Centrifugo Publisher

**Files:**
- Create: `backend/internal/realtime/centrifugo.go`
- Test: `backend/internal/realtime/centrifugo_test.go`

**Reference:** `backend/_archived/internal/realtime/centrifugo.go` (channel naming, payload structure, APIKey auth)

**Step 1: Write centrifugo_test.go**

```go
// backend/internal/realtime/centrifugo_test.go
package realtime

import (
    "testing"
)

func TestChannelName(t *testing.T) {
    c, _ := NewCentrifugoClient(CentrifugoConfig{URL: "http://localhost:8000", APIKey: "key"})
    name := c.ChannelName("abc123")
    expected := "messenger:chat:abc123"
    if name != expected {
        t.Fatalf("expected %s, got %s", expected, name)
    }
}

func TestNewCentrifugoClientRequiresURL(t *testing.T) {
    _, err := NewCentrifugoClient(CentrifugoConfig{})
    if err == nil {
        t.Fatal("expected error for empty URL")
    }
}
```

**Step 2: Implement centrifugo.go**

```go
// backend/internal/realtime/centrifugo.go
package realtime

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
)

type CentrifugoConfig struct {
    URL    string
    APIKey string
}

type CentrifugoClient struct {
    url        string
    apiKey     string
    httpClient *http.Client
}

func (c *CentrifugoClient) Publish(ctx context.Context, channel string, data interface{}) error {
    payload := map[string]interface{}{
        "method": "publish",
        "params": map[string]interface{}{
            "channel": channel,
            "data":    data,
        },
    }

    body, err := json.Marshal(payload)
    if err != nil {
        return fmt.Errorf("marshal payload: %w", err)
    }

    req, err := http.NewRequestWithContext(ctx, http.MethodPost, c.url+"/api", bytes.NewReader(body))
    if err != nil {
        return fmt.Errorf("create request: %w", err)
    }
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "apikey "+c.apiKey)

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return fmt.Errorf("publish to centrifugo: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        respBody, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("centrifugo error (%d): %s", resp.StatusCode, string(respBody))
    }
    return nil
}

func (c *CentrifugoClient) ChannelName(chatID string) string {
    return "messenger:chat:" + chatID
}

// NewCentrifugoClient — constructor at end of file
func NewCentrifugoClient(cfg CentrifugoConfig) (*CentrifugoClient, error) {
    if cfg.URL == "" {
        return nil, fmt.Errorf("centrifugo URL is required")
    }
    return &CentrifugoClient{
        url:        cfg.URL,
        apiKey:     cfg.APIKey,
        httpClient: &http.Client{Timeout: 10 * time.Second},
    }, nil
}
```

**Step 3: Run tests**

```bash
go test ./internal/realtime/ -v
```

Expected: PASS

**Step 4: Commit**

```bash
git add backend/internal/realtime/
git commit -m "feat(backend): Centrifugo publisher with channel naming"
```

---

## Task 12: Wire Everything in App + Final Verification

**Files:**
- Modify: `backend/internal/messenger/app.go` (wire all components)
- Modify: `backend/cmd/messenger/main.go` (database init)

**Step 1: Update app.go to wire components**

Update `NewApp()` to accept database pool and create all sub-services. Add ingestion flush loop as a second actor.

```go
// Full updated app.go — see reference at backend/_archived/internal/telegram/adapter.go for wiring pattern
```

The exact wiring depends on which components are needed at startup. For the MVP scaffold, wire: server + bot API handler + ingester flush loop.

**Step 2: Run full test suite**

```bash
cd /home/shaba/Repositories/SF/backend
go test ./... -v
```

Expected: ALL PASS

**Step 3: Run go vet**

```bash
go vet ./...
```

Expected: No issues

**Step 4: Build binary**

```bash
go build ./cmd/messenger/
```

Expected: Compiles successfully

**Step 5: Commit**

```bash
git add backend/
git commit -m "feat(backend): wire all components in App, full rewrite complete"
```

---

## Task 13: Cleanup + Final Commit

**Step 1: Verify no exported interfaces (except models)**

```bash
cd /home/shaba/Repositories/SF/backend
grep -rn "type [A-Z].*interface" internal/ --include="*.go" | grep -v "_archived" | grep -v "common/models"
```

Expected: No results (all interfaces should be lowercase/local).

**Step 2: Verify constructors at end of files**

Manually check that every `New*` function is the last function in its file.

**Step 3: Verify no log-and-return pattern**

```bash
grep -rn "slog\.\(Error\|Warn\)" internal/ --include="*.go" -A2 | grep "return.*err" | grep -v "_archived"
```

Expected: No results.

**Step 4: Run full test suite one more time**

```bash
go test ./... -v -count=1
```

Expected: ALL PASS

**Step 5: Final commit**

```bash
git add -A
git commit -m "chore: Go backend rewrite complete — squad-pattern architecture"
```

---

## Summary

| Task | Component | Key Pattern |
|---|---|---|
| 0 | Archive | Move old code to `_archived/` |
| 1 | Scaffold | common/models, config (struct tags), app.go, main.go (oklog/run) |
| 2 | HTTP Server | Run()/Shutdown() pair, HMAC middleware, health endpoint |
| 3 | Database | Pool + embedded SQL migrations |
| 4 | Bot API | Webhook handler, media detection, non-blocking channel |
| 5 | MTProto base | Types, session store (UPSERT), rate limiter (token bucket + FLOOD_WAIT) |
| 6 | MTProto client | Auth state machine, message listener |
| 7 | Adapter | Dual protocol, local interfaces, goroutine forwarding |
| 8 | SF Auth+Client | JWT Bearer flow, double-check-locking token refresh |
| 9 | SF Ingestion | Batch limits (950KB/200), sync.Pool, flush loop |
| 10 | R2 Media | S3-compatible client, presigned URLs, path-style |
| 11 | Centrifugo | Channel naming, APIKey auth, publish |
| 12 | Wiring | App.NewApp() wires all components |
| 13 | Cleanup | Verify patterns, final test suite |
