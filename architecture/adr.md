# Architectural Decision Records

| ADR | Decision | Rationale |
|---|---|---|
| **ADR-1** | Support BOTH MTProto and Bot API bidirectionally | Different use cases require different protocols. Parsing needs MTProto; user interaction needs Bot API. Both send and receive. |
| **ADR-2** | Remote PostgreSQL (not local/embedded) as Go-side data store | Proper hosted database for durability, backups, connection pooling. Not bbolt/SQLite — we need concurrent writes, queue processing, and multi-table relationships. |
| **ADR-3** | Cloudflare R2 + Workers for media | Salesforce storage is expensive (~2 GB/user). R2 has zero egress fees. Worker-based HMAC auth is more secure than pre-signed URLs for display. |
| **ADR-4** | Centrifugo for real-time LWC chat | Salesforce can't host WebSockets. Platform Event limits (50K delivery/24h) too restrictive for active chat. Centrifugo is Go-native, deploys alongside middleware. |
| **ADR-5** | Single corporate API ID + residential proxy pool | Per-user API IDs violate Telegram ToS. Shared API ID from single IP triggers bans. Proxy isolation per session is the proven strategy. |
| **ADR-6** | HMAC-SHA256 for webhook auth | Simpler than OAuth for high-velocity stateless pushes. Constant-time comparison prevents timing attacks. Pre-shared secret in Protected Custom Metadata. |
| **ADR-7** | Hybrid ingestion: Platform Events + Bulk API 2.0 + REST | No single API handles all cases. Events for live UI, Bulk for archives, REST for ad-hoc single updates. |
| **ADR-8** | Pub/Sub API (gRPC) for Go <-> Salesforce event streaming | Replaces legacy CometD. ~7-10x faster (compressed protobuf). Bidirectional streaming, pull-based flow control. Excellent Go gRPC support. |
| **ADR-9** | Hetzner (Europe) as primary hosting | Proximity to Telegram DCs in Amsterdam. Clean IPs. ARM Ampere instances available. Cost-effective for long-running Go processes. |
| **ADR-10** | Messenger-agnostic Go interface design | MVP is Telegram, but WhatsApp/Messenger/Viber planned. Adapter pattern: new platforms = new Go files, not restructuring core. |
| **ADR-11** | 2GP Managed Package (second-generation) | Salesforce recommends 2GP for all new packages. Better CI/CD integration, scratch org support, semantic versioning. |
| **ADR-12** | Generic `Messenger_*` naming for all objects/tables | Not `Telegram_*`. Supports multi-platform from day one without schema migrations when adding new platforms. |
| **ADR-13** | Centrifugo JWT auth via Apex-generated tokens | Centrifugo is stateless — needs cryptographic trust from Salesforce session context. HMAC-SHA256 default (simpler), RSA-SHA256 for cross-org deployments. Short-lived JWTs (15 min) with LWC refresh callback. |
| **ADR-14** | Direct-to-R2 outbound media upload via SigV4 presigned URLs | Avoids routing binary files through Salesforce storage tier (2 GB quota, DB bloat). LWC streams to R2 with near-zero memory footprint via Blob interface. |
| **ADR-15** | SRP v6a implementation for MTProto 2FA via gotd/td crypto/srp | PBKDF2 with 100K iterations must run on Go backend (browser JS would lock UI). Zero-knowledge proof — password never transmitted in plaintext. |
| **ADR-16** | Platform Events for outbound delivery failure sync | Real-time UI feedback for agents (red exclamation mark on failed messages) + durable persistence via dual-consumer pattern (LWC empApi + Apex Trigger). |
| **ADR-17** | sync.Pool + streaming JSON encoder for batch byte tracking | Prevents GC thrashing under high-throughput bursts. Individual struct-level byte tracking avoids full-slice re-serialization. 950KB threshold with 2-3 sec sliding window. |
