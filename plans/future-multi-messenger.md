# Multi-Messenger Architecture Plan

## Design Principle

MVP targets Telegram, but the Go middleware is architected so adding WhatsApp, Facebook Messenger, or any other platform requires only implementing a new adapter — no core restructuring.

## Go Interface Design

```go
// messenger.go — Core abstraction

type Platform string

const (
    PlatformTelegram  Platform = "telegram"
    PlatformWhatsApp  Platform = "whatsapp"   // future
    PlatformMessenger Platform = "messenger"   // future
)

// Message is the universal message struct all adapters produce/consume
type Message struct {
    ID            string
    Platform      Platform
    ChatID        string
    SenderID      string
    SenderName    string
    Text          string
    MediaType     string       // "photo", "video", "voice", "document", ""
    MediaFileID   string       // Platform-specific file reference
    ReplyToID     string
    Timestamp     time.Time
    Direction     Direction    // Inbound | Outbound
    RawPayload    json.RawMessage  // Original platform-specific JSON
}

type Direction string
const (
    Inbound  Direction = "inbound"
    Outbound Direction = "outbound"
)

// MessengerAdapter is what each platform must implement
type MessengerAdapter interface {
    // Platform returns the platform identifier
    Platform() Platform

    // Connect establishes the connection (MTProto session, webhook, etc.)
    Connect(ctx context.Context, config ConnectionConfig) error

    // Disconnect gracefully shuts down
    Disconnect(ctx context.Context) error

    // InboundMessages returns a channel of incoming messages
    InboundMessages() <-chan Message

    // SendMessage sends a message through this platform
    SendMessage(ctx context.Context, msg OutboundMessage) error

    // DownloadMedia downloads a file and returns a reader
    DownloadMedia(ctx context.Context, fileID string) (io.ReadCloser, MediaMeta, error)

    // Status returns the connection health
    Status() ConnectionStatus
}

// ConnectionConfig holds platform-specific config
type ConnectionConfig struct {
    Platform     Platform
    Credentials  map[string]string  // "phone", "api_id", "api_hash", "bot_token", etc.
    ProxyURL     string             // For MTProto IP isolation
    SalesforceID string             // Salesforce Connection record ID
}
```

## Telegram Adapter (MVP)

```go
// telegram_adapter.go — implements MessengerAdapter for BOTH MTProto and Bot API

type TelegramAdapter struct {
    mtprotoClient *telegram.Client     // gotd/td
    botAPI        *botapi.BotClient    // HTTP Bot API client
    mode          string               // "mtproto" | "bot_api"
    messages      chan Message
    // ...
}

func (t *TelegramAdapter) Platform() Platform { return PlatformTelegram }

// Connect initializes either MTProto or Bot API depending on config
func (t *TelegramAdapter) Connect(ctx context.Context, cfg ConnectionConfig) error {
    switch {
    case cfg.Credentials["bot_token"] != "":
        return t.connectBotAPI(ctx, cfg)
    case cfg.Credentials["phone"] != "":
        return t.connectMTProto(ctx, cfg)
    default:
        return errors.New("no valid credentials provided")
    }
}
```

## Future Adapter Skeleton

```go
// whatsapp_adapter.go — future
type WhatsAppAdapter struct {
    // WhatsApp Business API client
    messages chan Message
}

func (w *WhatsAppAdapter) Platform() Platform { return PlatformWhatsApp }
// ... implement all MessengerAdapter methods
```

## Database Schema Supports Multi-Platform

Both PostgreSQL and Salesforce schemas use generic `Messenger_*` / `messenger_*` naming (not `Telegram_*`) and include a `platform` / `Messenger_Platform__c` field. Adding a new platform requires:

- New Go adapter file (implements `MessengerAdapter`)
- New platform constant
- No schema changes

## Future Platforms

| Platform | API Type | Notes |
|---|---|---|
| WhatsApp Business API | REST webhooks | Cloud API or On-Premises. Rate limits per phone number |
| Facebook Messenger | REST webhooks | Graph API. Page-scoped user IDs |
| Instagram Direct | REST webhooks | Via Facebook Graph API. Business accounts only |
| Viber | REST webhooks | Viber Bot API. Similar to Telegram Bot API |
| Custom webhooks | Configurable | For proprietary internal messaging systems |
