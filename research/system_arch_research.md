# System Architecture & Deep Research Context: Telegram-to-Salesforce Integration

## 1. Project Overview & Core Goals
**Goal:** Develop an AppExchange-compliant plugin for Salesforce to handle integrations with Telegram (and eventually other messengers). 
**Concept:** A secure middleware built in Go (Golang) acts as the bridge between the Telegram API (handling both MTProto userbots and standard Bot API) and the Salesforce ecosystem. 
**Primary Use Case:** The project serves as a local activities and events aggregator (e.g., parsing regional channels and chats across Cyprus to extract event data) and uses standard bots for user interaction.
**Initial Hypothesis:** Store all data—including MTProto authorization sessions, chats, messages, and media—natively within Salesforce to maintain a strict, secure data perimeter.

## 2. Technical Stack
* **Middleware Language:** Go (Golang) – chosen for high concurrency (goroutines) and efficient resource usage, compiled for ARM architectures.
* **Telegram MTProto Libraries:**
  * `gotd/td`: Low-level, highly reliable core engine.
  * `GoTGProto`: Wrapper for simplified session management and handler routing.
* **CRM Platform:** Salesforce (Apex, Lightning Web Components - LWC).

## 3. Telegram API Constraints to Handle
* **Bot API:** Strict rate limits for outgoing messages (~30 msg/sec for mass mailing, ~1 msg/sec for DMs, ~20 msg/min for groups/channels).
* **MTProto (Userbots):** Strict monitoring by Telegram requiring robust handling of `FloodWait` errors and enforced request pacing to avoid account bans during parsing.

## 4. Architectural Bottlenecks & Strategic Pivots
The initial hypothesis of storing everything natively inside Salesforce presents critical bottlenecks regarding AppExchange requirements and platform limits. The architecture has been revised to address the following:

### A. AppExchange Security Review (Composite App)
* **Risk:** Because the architecture relies on an external Go server, Salesforce classifies this as a "Composite App."
* **Requirement:** The Go API must pass independent penetration testing (e.g., OWASP ZAP, Burp Suite). All transit data between Salesforce and the Go server must use TLS 1.2+ encryption. The Go server must prove it handles transient data securely without unauthorized logging.

### B. MTProto Session Storage Latency
* **Risk:** MTProto frequently updates authorization keys and salts. Routing the `StoreSession` method to write directly to a Salesforce database via REST API for every update introduces high latency, race conditions, and frequent connection drops.
* **Pivot:** Active, "hot" session bytes for `gotd` must be stored locally next to the Go worker (e.g., in a local PostgreSQL or Redis database). Salesforce should only store integration metadata (e.g., connection status, Salesforce user ID to Telegram account mapping).

### C. Media Storage Costs & Limits
* **Risk:** Salesforce data and file storage (`ContentVersion` / `ContentDocument`) are highly constrained and expensive. Saving Telegram images, videos, and voice notes directly to Salesforce will rapidly exhaust client storage limits.
* **Pivot:** Implement an external S3-compatible storage layer (AWS S3, Cloudflare R2). The Go server intercepts media from Telegram, uploads it to S3, and passes only the pre-signed/public URL to Salesforce for UI rendering.

### D. Salesforce API Request Limits
* **Risk:** Telegram generates high-frequency events (typing statuses, read receipts, rapid messages). A 1-to-1 REST API call to Salesforce for every event will exhaust the client's daily API limits in hours.
* **Pivot:** Implement batching on the Go server OR utilize Salesforce Platform Events. Pushing events to the Salesforce event bus allows internal Apex triggers/Flows to process incoming data asynchronously without burning standard API quotas.

---

## 5. Targeted Questions for Deep Research

Based on the context above, please provide a comprehensive technical analysis and architectural recommendations for the following areas:

### A. Telegram API & Rate Limits (MTProto + Bot API)
1. **Multi-Tenancy Risks:** What are the strict rate limits and hidden thresholds for MTProto (userbots) when acting as a high-volume data parser and CRM gateway? How can we prevent cascading account bans when handling multiple sessions?
2. **FloodWait Handling:** What are the architectural best practices for queuing and pacing requests in a highly concurrent Go environment to gracefully handle `FloodWait` errors without dropping the MTProto connection?
3. **Bot API Quotas:** How should the middleware design its internal message queue to globally throttle outbound messages and comply with Telegram Bot API limits across multiple Salesforce instances?

### B. Salesforce Complexities & App Lifecycle
4. **AppExchange Onboarding:** What is the exact step-by-step process for creating, packaging, and publishing a "Composite App" (Managed Package) in Salesforce from scratch?
5. **Security Review Compliance:** What specific penetration testing reports, data-in-transit encryption standards, and architectural proofs does the Salesforce Security Review mandate for an external Go server handling messaging data?
6. **Inbound Data Ingestion:** To avoid hitting daily API limits, what is the most cost-effective and real-time method for ingesting high-frequency Telegram data into Salesforce: Platform Events, standard REST API batching, or the Bulk API?
7. **Authentication:** How should Salesforce securely authenticate webhook payloads or event streams pushed from the Go middleware to ensure zero unauthorized data entry?

### C. Media & Attachment Handling
8. **S3 UI Integration:** What is the optimal architecture for natively displaying external media (AWS S3 / Cloudflare R2) within Lightning Web Components (LWC) in Salesforce, ensuring a seamless user experience without consuming Salesforce file storage?
9. **Secure URL Lifecycle:** How should the system manage access to private attachments? Is it better to generate pre-signed URLs on-the-fly via Apex when the user opens the chat, or utilize a secure CDN wrapper?

### D. Go Application Hosting & DevOps Environment
10. **Infrastructure Selection:** Which cloud hosting providers offer the optimal environment for a network-heavy Go/MTProto application? The focus should be on low latency to Telegram's European data centers and pristine IP reputation to avoid automated Telegram anti-spam triggers.
11. **ARM Architecture Optimization:** Given the requirement to compile the Go binary for ARM (supporting local development on Apple Silicon / Raspberry Pi up to production deployments), what are the best scalable cloud options (e.g., AWS Graviton, Hetzner ARM cloud) and CI/CD considerations?
12. **Session Storage:** Which local database engine (e.g., PostgreSQL, Redis, or embedded options like bbolt) is highly recommended for storing active MTProto session states and keys right next to the Go worker, minimizing latency during cryptographic salt updates?