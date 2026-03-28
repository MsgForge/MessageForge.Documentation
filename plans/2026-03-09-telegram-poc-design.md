# Telegram PoC — Design Document

**Date:** 2026-03-09
**Status:** Approved
**Goal:** Standalone demo app — run it, enter API keys, login to Telegram, browse chats, send messages.

## Architecture

```
poc/telegram-demo/
├── main.go              # Entry point
├── go.mod               # Standalone module
├── Makefile             # `make run` builds React + runs Go
├── internal/
│   ├── server.go        # HTTP server, routes, static file serving
│   ├── state.go         # In-memory AppState (config, clients, auth)
│   ├── handlers_config.go
│   ├── handlers_auth.go
│   ├── handlers_chat.go
│   └── telegram/
│       ├── mtproto.go   # gotd/td client wrapper
│       └── botapi.go    # telegram-bot-api wrapper
└── web/                 # React Router app (Vite)
    ├── package.json
    ├── vite.config.ts
    └── src/
        ├── main.tsx
        ├── App.tsx          # AuthGuard + Router + Nav
        ├── api.ts           # fetch wrapper
        ├── pages/
        │   ├── Setup.tsx    # API ID, Hash, Bot Token form
        │   ├── Login.tsx    # Phone → OTP → 2FA steps
        │   ├── Chats.tsx    # Dialog list
        │   └── Chat.tsx     # Message history + send
        └── components/
            ├── Nav.tsx      # Persistent top nav with status dots
            └── AuthGuard.tsx
```

## Smart Routing (Auto-Onboarding)

React `<AuthGuard>` calls `GET /api/status` on every navigation:

```
Response: { stage: "needs_config" | "needs_auth" | "ready" }

needs_config → redirect to /setup
needs_auth   → redirect to /login
ready        → redirect to /chats
```

- User opens `localhost:8080` → lands on the correct page automatically
- Completing setup → auto-redirect to /login
- Completing login → auto-redirect to /chats
- Nav shows all steps but disables unreachable ones

## API Endpoints

| Method | Endpoint | Request Body | Response | Description |
|--------|----------|-------------|----------|-------------|
| GET | `/api/status` | — | `{stage, mtproto: {configured, connected, user?}, bot: {configured, connected}}` | App state for routing |
| POST | `/api/config` | `{api_id, api_hash, bot_token?}` | `{stage}` | Save config, init clients |
| POST | `/api/auth/phone` | `{phone}` | `{phone_code_hash}` | Send OTP |
| POST | `/api/auth/code` | `{phone, phone_code_hash, code}` | `{stage}` (ready or needs_2fa) | Verify OTP |
| POST | `/api/auth/2fa` | `{password}` | `{stage, user}` | SRP 2FA |
| GET | `/api/chats` | `?source=all\|mtproto\|bot` | `[{id, name, type, last_message, source}]` | Chat list |
| GET | `/api/chats/:id/messages` | `?source=mtproto\|bot&limit=50` | `[{id, sender, text, date}]` | History |
| POST | `/api/send` | `{chat_id, text, source}` | `{message_id}` | Send message |

Error format: `{error: "message", code: "ERROR_CODE"}`

## State Management

All in-memory, single `AppState` struct in Go:

```go
type AppState struct {
    mu          sync.RWMutex
    Config      *AppConfig        // API ID, Hash, Bot Token
    MTProto     *MTProtoClient    // gotd/td client
    BotAPI      *BotAPIClient     // telegram-bot-api client
    AuthSession *AuthSession      // phone, code hash, state
    Stage       string            // "needs_config" | "needs_auth" | "ready"
}
```

No database, no cookies, no persistence. Restart = start over.

## Libraries

| Library | Purpose |
|---------|---------|
| `github.com/gotd/td` | MTProto 2.0 client |
| `github.com/go-telegram-bot-api/telegram-bot-api/v5` | Bot API client |
| `net/http` (stdlib) | HTTP server |
| React 19 + React Router 7 | SPA frontend |
| Vite | Build tool |

## UI Pages

### /setup
- Form: API ID (number), API Hash (string), Bot Token (string, optional)
- "Connect" button
- Status badges showing what's configured

### /login
- Step 1: Phone number input → "Send Code"
- Step 2: OTP code input (6 digits) → "Verify"
- Step 3 (if needed): 2FA password → "Submit"
- Shows logged-in user info when complete

### /chats
- List of recent dialogs from both MTProto and Bot API
- Each item: avatar placeholder, name, last message preview, source badge
- Click → navigate to /chats/:id

### /chats/:id
- Scrollable message history
- Source toggle (MTProto / Bot API) for sending
- Text input + send button
- Auto-scroll to bottom

## Styling

Minimal custom CSS. Clean, functional, no component library. Light theme.

## How to Run

```bash
cd poc/telegram-demo
make run
# Opens http://localhost:8080
```

`make run` does:
1. `cd web && npm install && npm run build`
2. `go run .`
