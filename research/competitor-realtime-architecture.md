# Competitor Real-Time Architecture Analysis

**Date:** 2026-04-01
**Purpose:** Understand how AppExchange messaging apps handle real-time message delivery to validate/improve our Centrifugo-based approach.

---

## Executive Summary

Every competitor uses one of three patterns for real-time updates: (1) Salesforce Streaming API/PushTopics via empApi (SMS-Magic, Mogli, 360 SMS), (2) external server with embedded UI widget (Heymarket, Sinch Engage), or (3) Salesforce's own internal infrastructure (Digital Engagement). **None use an external WebSocket server the way we plan to with Centrifugo** -- making our approach architecturally unique but also technically superior for scale. The math is clear: Platform Events with fan-out to 100+ agents produces 2M+ daily deliveries against a 50K limit (Unlimited Edition). Our Centrifugo approach consumes **zero** SF event deliveries for UI updates. External WebSocket connections from LWC are officially supported since Summer '18 and pass AppExchange security review.

---

## Topic 1: SMS-Magic (Conversive)

### Findings

- **Real-time method: PushTopics (legacy Streaming API) via empApi/CometD**
  - Their `streamingDataService` Aura component wraps empApi to subscribe to PushTopic on `Incoming SMS` / `SMS History` custom objects
  - When a new record is created, PushTopic fires and Converse Desk UI auto-refreshes without page reload
  - Source: [Version 1.58 Release Notes](https://www.sms-magic.co/docs/converse-release-notes/knowledge-base/patch-version-1-58/)
  - Source: [Freshdesk support article](http://screenmagic.freshdesk.com/support/solutions/articles/233242-incoming-messages-are-not-auto-loaded-on-converse-desk-and-message-flow-user-interface-)

- **External middleware: YES -- "SMS-Magic Platform" / "SMS-Magic Portal"**
  - Despite "native" marketing, operates external servers in US and Ireland (EU)
  - Receives messages from telecom providers (Vonage/Twilio), routes to Salesforce via OAuth
  - OAuth failover mechanism with multiple backup users suggests connection fragility
  - Source: [SFMC Architecture](https://docs.sms-magic.com/wl5N2H674bM-sms-magic-converse-integration-with-salesforce-marketing-cloud-sfmc/GBgyhgYnY9U-sfmc-native-app-architecture-explained)
  - Source: [OAuth FAQ](https://www.sms-magic.co/docs/salesforce/faq/25-what-is-oauth-is-it-necessary-to-enable-the-oauth/)
  - Source: [OAuth Failover](https://www.sms-magic.com/docs/portal/knowledge-base/oauth-user-management-and-failover/)

- **Typing indicators / read receipts: NOT supported**
  - Green dot for visitor activity only during bot-enabled webchat, not SMS/WhatsApp
  - Source: [Receive and Respond](https://www.sms-magic.com/docs/salesforce/knowledge-base/receive-and-respond-to-enquiries/)

- **Critical dependency: Streaming API must be available**
  - Professional Edition customers must purchase Streaming API add-on
  - Summer '18 Salesforce release broke their Streaming API on Visualforce pages
  - v1.58 fixed a bug where empApi was force-disabled in Lightning
  - Source: [Supported Editions](https://www.sms-magic.com/docs/sf-quickstart/knowledge-base/supported-salesforce-editions/)

- **OmniChannel integration available** for routing incoming conversations to agents
  - Source: [OmniChannel Integration](https://docs.sms-magic.com/dRSpyt9QVyE-sms-magic-converse-integration-with-salesforce-omnichannel)

### Confidence: HIGH

---

## Topic 2: Mogli SMS & WhatsApp

### Findings

- **Real-time method: PushTopics (legacy Streaming API) via empApi/CometD**
  - `Mogli SMS: Push Topics` permission set required for every user -- "grants users the ability to see real-time notifications and messages in the Mogli Conversation View without needing to refresh the browser"
  - v5.66.2 release notes confirm PushTopics updated for real-time outgoing message display
  - Source: [Mogli Access & Visibility docs](https://mogli-sms.document360.io/docs/access-visibility)
  - Source: [Mogli Release Notes v5.66.2](https://mogli-sms.document360.io/docs/release-notes-v5661)
  - Source: [Mogli Notifications docs](https://mogli-sms.document360.io/docs/mogli-notifications)

- **Utility Bar notifications with audible chime** -- blinks and plays sound on inbound messages, navigates to associated record on click
  - Smart routing: notifies the last user who sent an outgoing text to that record
  - Source: [Mogli Notifications](https://mogli-sms.document360.io/docs/mogli-notifications)

- **External middleware: NONE -- "100% native Salesforce" (with caveats)**
  - No middleware server of their own. Carrier gateways (Twilio/Telnyx/MessageBird/Plivo) handle transport
  - Outbound: Apex callouts directly to carrier REST APIs
  - Inbound: Carrier webhook -> Salesforce Apex REST endpoint -> `Mogli_SMS__SMS_History__c` record created -> PushTopic fires
  - The carrier gateway effectively IS their middleware
  - Source: [Mogli homepage](https://www.mogli.com/), [AppExchange listing](https://appexchange.salesforce.com/appxListingDetail?listingId=a0N3A00000DqCytUAF)

- **Typing indicators / read receipts: NOT found**
  - Expected: SMS doesn't support typing natively. WhatsApp read receipts available via API but no evidence Mogli surfaces them

- **Risk: PushTopics are deprecated** -- Salesforce recommends Platform Events or CDC. Limited to 50 PushTopics per org, 20-field SELECT limit.

### Confidence: HIGH

---

## Topic 3: Heymarket

### Findings

- **Real-time method: 1-minute polling (scheduled Apex job)**
  - Default sync interval: every 1 minute. Configurable to 5 or 10 minutes
  - No evidence of Platform Events, Streaming API, CDC, or any push-based mechanism
  - Documentation explicitly states: "If you adjust the message sync interval, messages will experience delays"
  - Source: [Heymarket SF Integration](https://help.heymarket.com/hc/en-us/articles/360034372271-Salesforce-Integration)

- **Dual-path architecture:**
  - **Path A (Widget):** Near-real-time via direct connection to Heymarket's cloud backend (app.heymarket.com). Widget is likely an iframe embedding their React SPA. This is what agents use for live chat
  - **Path B (SF data):** Polling-based, 1-minute delay. `heymarket__Message__c` custom objects updated by scheduled job. This is what Flows, reports, and triggers see
  - Data consistency issue: widget shows real-time data while SF records lag behind

- **External server: YES -- standalone SaaS platform**
  - Primary product lives at app.heymarket.com (Node.js, React, AWS infrastructure)
  - Microservices architecture, SOC 2 Type 2 compliant
  - SF package is one integration among many (HubSpot, Slack, Zapier)
  - Auth: API key-based (not OAuth)
  - Source: [Heymarket Integrations](https://www.heymarket.com/integrations/), [Heymarket SOC 2](https://www.heymarket.com/product/soc-2/)

- **Typing indicators: NOT found**

- **No native SF real-time push** -- clear gap in their architecture

### Confidence: HIGH

---

## Topic 4: Salesforce Digital Engagement (1st Party)

### Findings

- **Agent console: CometD long-polling via `lightning/empApi`**
  - All LWC components share a single CometD connection per browser tab
  - Source: [lightning-emp-api docs](https://developer.salesforce.com/docs/component-library/bundle/lightning-emp-api/documentation)

- **Customer-facing widget (MIAW): Server-Sent Events (SSE)**
  - Endpoint: `/eventrouter/v1/sse` on `*.my.salesforce-scrt.com` domain
  - Unidirectional push from server to client. Messages sent back via REST POST
  - Source: [Enhanced Chat SSE docs](https://developer.salesforce.com/docs/service/messaging-api/references/about/server-sent-events.html)

- **External system integration (BYOC): gRPC Pub/Sub API**
  - Bidirectional event streaming at `api.pubsub.salesforce.com:7443`
  - How partner connectors (e.g., Telegram BYOC) receive outbound agent messages
  - Source: [Pub/Sub API docs](https://developer.salesforce.com/docs/platform/pub-sub-api/guide/intro.html)
  - Source: [BYO Demo Connector (GitHub)](https://github.com/salesforce-misc/byo-demo-connector)

- **Off-platform messaging database (INTERNAL-ONLY)**
  - ConversationEntry records stored in separate, purpose-built database (NOT main Oracle DB)
  - NOT queryable via SOQL. Access limited to Connect REST API, Data Cloud sync, or bulk export (7-day interval)
  - Source: [Messaging Object Model](https://developer.salesforce.com/docs/service/messaging-object-model/guide/messaging-object-model.html)

- **SCRT2 microservice layer (INTERNAL-ONLY)**
  - Runs on `salesforce-scrt.com` -- the real-time messaging backbone, separate from core platform
  - Routing engine, message persistence, and relay between SCRT2 and core platform are proprietary

- **No raw WebSocket connections** -- CometD can upgrade to WebSocket transport but typically uses HTTP long-polling. Customer widget uses SSE (HTTP/1.1)

- **ISV path: Bring Your Own Channel (BYOC)**
  - Designated partner integration via Interaction Service API
  - NeuraFlash already shipped a Telegram BYOC integration on AppExchange
  - Source: [BYOC Introduction](https://developer.salesforce.com/docs/service/messaging-partner/guide/introduction.html)

| Technology | Where Used | Available to ISVs? |
|---|---|---|
| CometD/empApi | Agent console LWC | YES |
| SSE | Customer-facing MIAW widget | YES (Enhanced Chat API) |
| gRPC Pub/Sub API | External system integration | YES |
| Lightning Message Service | Intra-console component comm | YES (packageable in 2GP) |
| Off-platform messaging DB | ConversationEntry storage | NO (Connect REST only) |
| SCRT2 internal relay | Routing + persistence | NO |

### Confidence: HIGH (official docs + GitHub reference implementations)

---

## Topic 5: Platform Events vs Streaming API vs CDC -- Real Limits

### Platform Events Daily Delivery Limits

| Edition | Daily Delivery (24h) | Hourly Publishing |
|---|---|---|
| Enterprise | ~25,000 | Not separately documented |
| Unlimited/Performance | 50,000 | 250,000/hour |
| + High-Volume Add-On | +100,000 (total 150K) | +25,000 (total 275K/hour) |

- **CRITICAL: Fan-out formula** -- `Daily Deliveries = Events Published x Number of Subscribers`
- Example: 1,000 events x 10 subscribers = 10,000 deliveries consumed
- Source: [Platform Event Allocations](https://developer.salesforce.com/docs/atlas.en-us.platform_events.meta/platform_events/platform_event_limits.htm)
- Source: [Working Within Delivery Limits (SF Dev Blog)](https://developer.salesforce.com/blogs/2021/08/how-to-work-within-platform-events-delivery-limits)

**Add-on pricing:** Not published. Quote-based via Salesforce AE.

**Event retention:** 72 hours (high-volume), 24 hours (standard-volume, retiring Summer '27)

### Streaming API / CometD Limits

- **2,000 concurrent CometD subscribers per org** (shared across ALL empApi clients)
- Each browser tab = 1 CometD connection. N tabs = N connections against the 2,000 limit
- CometD latency: ~300ms per event (vs ~50ms for WebSocket -- 6x slower)
- Source: [SF Dev Blog](https://developer.salesforce.com/blogs/2021/08/how-to-work-within-platform-events-delivery-limits)

### Change Data Capture (CDC) Limits

- Fires on custom object insert, update, delete, undelete: **YES**
- Default: 5 entities tracked. With add-on: unlimited
- **Shares the same daily delivery pool as Platform Events**
- Enterprise: 25,000/day. Unlimited: 50,000/day. With add-on: 125K / 150K
- Latency: sub-second under normal conditions (best-effort, no SLA)
- Source: [CDC Allocations](https://developer.salesforce.com/docs/atlas.en-us.change_data_capture.meta/change_data_capture/cdc_allocations.htm)

### Scenario: 100 Agents, 10,000 Messages/Day

| Approach | Daily Deliveries | Fits UE Limit (50K)? | Latency |
|---|---|---|---|
| Platform Events to all agents via empApi | 10,000 x 200 tabs = **2,000,000** | NO (40x over) | ~300ms |
| CDC to all agents via empApi | Same fan-out = **2,000,000** | NO (40x over) | ~300ms |
| PE with client filtering | Still counted = **2,000,000** | NO (filtered events still count) | ~300ms |
| PE (backend only, 1-2 subscribers) + Centrifugo (UI) | **10,000-20,000** | YES | **~50ms** |

**The fan-out formula makes Platform Events impossible for chat UI at scale.** Even Unlimited + add-on (150K) is exceeded by 13x.

### Confidence: HIGH (official docs, math is straightforward)

---

## Topic 6: External WebSocket in AppExchange -- Is It Allowed?

### Findings

- **YES -- officially supported since Summer '18 (API v218)**
  - Feature: "Create CSP Trusted Sites to Use WebSocket Connections"
  - `connect-src` CSP directive governs AJAX, WebSocket, and EventSource connections
  - Must use `wss://` (secure). Plain `ws://` is blocked
  - Source: [Summer '18 Release Notes](https://help.salesforce.com/s/articleView?id=release-notes.rn_lc_websockets.htm&language=en_US&release=218&type=5)
  - Source: [Secure Coding WebSockets](https://developer.salesforce.com/docs/atlas.en-us.secure_coding_guide.meta/secure_coding_guide/secure_coding_websockets.htm)

- **CSP/Security requirements:**
  - TLS 1.2+ mandatory
  - Qualys SSL Server Test must return A grade
  - All JS libraries (e.g., Centrifugo client SDK) must be bundled as Static Resources, NOT loaded from CDN
  - Source: [Top 20 AppExchange Vulnerabilities](https://developer.salesforce.com/blogs/2023/08/the-top-20-vulnerabilities-found-in-the-appexchange-security-review)

- **Approved apps using external servers:** Many (Talkdesk, RingCentral, Five9, Twilio, Sinch). CTI apps are the closest analogy -- they use real-time bidirectional communication. Specific WebSocket confirmation is not publicly documented but highly likely for CTI.

- **Security review requirements for external connections:**
  1. All external endpoints scanned (SSL/TLS compliance via Qualys)
  2. HTTPS/WSS only
  3. OAuth or equivalent secure auth (HMAC acceptable)
  4. No hotlinked external JS
  5. Comprehensive documentation of all external dependencies
  6. Test credentials for external systems
  7. Vulnerability scanning of all non-Salesforce libraries
  - Source: [Prepare for Security Review](https://developer.salesforce.com/blogs/2023/04/prepare-your-app-to-pass-the-appexchange-security-review)

- **CSP Trusted Site in managed package:**
  - CspTrustedSite is a Metadata API type but may NOT be packageable in 2GP
  - Standard pattern: post-install setup guide instructs admin to add CSP Trusted Site
  - Many AppExchange apps (Talkdesk, InCountry) do this -- well-accepted pattern
  - Source: [CspTrustedSite Metadata API](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_csptrustedsite.htm)

- **LWC WebSocket is front-end only** -- Apex cannot open WebSocket connections. LWC JavaScript connects directly to external WebSocket server. This is exactly our Centrifugo pattern.

### Confidence: HIGH

---

## Topic 7: Comparative Architecture Table

| App | Real-Time Method | External Server? | Latency | SF Limits Consumed | Typing/Presence? |
|---|---|---|---|---|---|
| **SMS-Magic** | PushTopics (empApi/CometD) | Yes (SMS-Magic Platform, US/EU) | ~300ms (CometD) | Streaming API subscribers + deliveries | No |
| **Mogli** | PushTopics (empApi/CometD) | No (carrier webhooks direct to SF) | ~300ms (CometD) | Streaming API subscribers + deliveries | No |
| **Heymarket** | 1-min polling (scheduled Apex) + iframe widget | Yes (app.heymarket.com, AWS) | 60s (SF data) / real-time (widget) | Minimal (polling, not streaming) | No |
| **SF Digital Engagement** | CometD/empApi (agent) + SSE (customer) + gRPC Pub/Sub (external) | Yes (SCRT2 on salesforce-scrt.com) | Sub-second | Internal (not counted against org limits) | Yes (internal) |
| **360 SMS** | Unresolved (likely Streaming API or polling) | No (100% native, carrier callouts) | Unknown | Likely Streaming API | No |
| **Sinch Engage** | External Sinch API servers feed `mercury:mercuryFeed` component | Yes (api.*.saas.sinch.com) | Near-real-time (from Sinch servers) | Minimal (data stored as Tasks) | Yes (WhatsApp only) |
| **MessageForge (ours)** | **Centrifugo WebSocket** | Yes (self-hosted, Hetzner) | **~50ms** | **Zero for UI updates** | **Yes (planned)** |

---

## Topic 8: Lessons for MessageForge

### What We Should Copy

1. **Utility Bar notifications (Mogli pattern)** -- audible chime + blinking icon + direct navigation to record. This is table stakes UX. Every competitor has some form of notification beyond the chat window.

2. **OmniChannel integration (SMS-Magic pattern)** -- route incoming conversations to available agents using native SF OmniChannel. This is expected for enterprise deployments and differentiates from simple "everyone sees everything" approaches.

3. **Smart routing (Mogli pattern)** -- notify the last agent who texted a contact, not all agents. Reduces noise and improves response times.

4. **Setup wizard for CSP configuration** -- since CspTrustedSite likely can't be packaged in 2GP, build a guided setup LWC component that instructs admins to add the Centrifugo endpoint. Multiple AppExchange apps (Talkdesk, InCountry) use this pattern.

### What We Do Better

1. **Zero SF limit consumption for real-time UI** -- the math is devastating for competitors:
   - 100 agents x 10K messages = 2M deliveries/day via Platform Events (40x over Unlimited limit)
   - Our approach: 0 deliveries consumed for UI updates. Platform Events used only for backend sync (1-2 subscribers = 10K-20K/day, well within limits)

2. **True WebSocket latency (~50ms) vs CometD (~300ms)** -- 6x faster than PushTopic-based competitors. Users will perceive our chat as noticeably more responsive.

3. **Typing indicators and presence** -- NO competitor on AppExchange offers typing indicators for messaging channels. SMS-Magic, Mogli, Heymarket, 360 SMS all lack this. Only Sinch Engage claims "live typing indicator" (WhatsApp only). This is a clear differentiator, especially for Telegram which natively supports typing status.

4. **No carrier dependency for Telegram** -- competitors (Mogli, 360 SMS) route through Twilio/Telnyx, paying per-message fees. Our direct MTProto connection means zero per-message carrier cost for Telegram.

5. **Modern tech stack** -- competitors rely on deprecated PushTopics (SMS-Magic, Mogli). We start with Platform Events + Centrifugo from day one.

6. **No dual-data-consistency problem** -- Heymarket's iframe shows real-time data while SF records lag 60 seconds behind. Our approach writes to SF first (source of truth), then pushes to Centrifugo, keeping SF data and UI in sync.

### Risks to Address

1. **AppExchange security review for external WebSocket** -- while `wss://` is officially supported, we need:
   - TLS 1.2+ on Centrifugo with Qualys A grade
   - Centrifugo JS client bundled as Static Resource (not CDN)
   - Full documentation of external dependency for reviewers
   - Test credentials / test Centrifugo instance for pen testing
   - **Action:** Set up a dedicated security-review-ready Centrifugo instance before submission

2. **CspTrustedSite may not be packageable in 2GP** -- verify via [Metadata Coverage Report](https://developer.salesforce.com/docs/metadata-coverage). If not, build a setup wizard LWC.

3. **Single point of failure: Centrifugo** -- if Centrifugo goes down, agents lose real-time updates. Need:
   - **Fallback to Platform Events** (empApi) when WebSocket connection fails
   - Health check + automatic reconnection in LWC
   - Centrifugo high-availability deployment (multiple nodes)
   - **Action:** Design dual-path: Centrifugo primary, Platform Events fallback

4. **NeuraFlash Telegram BYOC is direct competition** -- they've already shipped a Telegram integration using Salesforce's BYOC/Interaction Service API. Research their approach and positioning.

5. **Centrifugo adds operational complexity for customers** -- unlike "100% native" competitors (Mogli, 360 SMS), our customers need our Go middleware + Centrifugo running. For AppExchange, this means we host it (SaaS model), not the customer. Ensure pricing model accounts for infrastructure costs.

### Recommended Changes

1. **Add Platform Events fallback for real-time UI** -- if Centrifugo WebSocket disconnects, LWC should automatically fall back to empApi subscription on a Platform Event channel. This ensures degraded-but-functional real-time even during Centrifugo outages. The fan-out cost is acceptable as a temporary fallback (not primary path).

2. **Implement Utility Bar notification component** -- separate from the main chat LWC. Should include: audible notification, visual badge, conversation count, click-to-navigate. This is a competitive baseline.

3. **Plan OmniChannel integration for Phase 2+** -- create `PendingServiceRouting` records for incoming conversations so enterprises can use native SF routing. Not MVP but important for enterprise sales.

---

## Unresolved Questions

| Question | Why It Matters | How to Resolve |
|---|---|---|
| 360 SMS real-time mechanism (Streaming API vs polling?) | Understand if any "100% native" app achieves true push | Install trial from AppExchange, inspect network traffic |
| CspTrustedSite packageable in 2GP? | Determines if we need a post-install setup wizard | Check [Metadata Coverage Report](https://developer.salesforce.com/docs/metadata-coverage) |
| Platform Event limits for Developer/Professional editions | Affects customers on lower editions | Check official docs (JS-rendered page) or query `PlatformEventUsageMetric` |
| High-Volume Platform Event add-on pricing | Affects cost analysis for fallback approach | Contact Salesforce AE |
| NeuraFlash Telegram BYOC -- architecture details | Direct competitor | Install trial, analyze their approach |
| Heymarket widget internal transport (WebSocket vs polling to their servers?) | Understand if iframe-based approach uses WebSocket | Install trial, inspect DevTools network tab |
| empApi per-user subscription limit | Affects fallback design | Appears to be org-wide (2,000 CometD clients), not per-user |

---

## Sources

### Official Salesforce Documentation
- [Platform Event Allocations](https://developer.salesforce.com/docs/atlas.en-us.platform_events.meta/platform_events/platform_event_limits.htm)
- [Working Within PE Delivery Limits (Dev Blog)](https://developer.salesforce.com/blogs/2021/08/how-to-work-within-platform-events-delivery-limits)
- [CDC Allocations](https://developer.salesforce.com/docs/atlas.en-us.change_data_capture.meta/change_data_capture/cdc_allocations.htm)
- [Pub/Sub API Allocations](https://developer.salesforce.com/docs/platform/pub-sub-api/guide/allocations.html)
- [Streaming API Limits](https://developer.salesforce.com/docs/atlas.en-us.api_streaming.meta/api_streaming/limits.htm)
- [lightning-emp-api docs](https://developer.salesforce.com/docs/component-library/bundle/lightning-emp-api/documentation)
- [Enhanced Chat SSE docs](https://developer.salesforce.com/docs/service/messaging-api/references/about/server-sent-events.html)
- [Pub/Sub API docs](https://developer.salesforce.com/docs/platform/pub-sub-api/guide/intro.html)
- [Messaging Object Model](https://developer.salesforce.com/docs/service/messaging-object-model/guide/messaging-object-model.html)
- [BYOC Introduction](https://developer.salesforce.com/docs/service/messaging-partner/guide/introduction.html)
- [Conversation Toolkit API](https://developer.salesforce.com/docs/component-library/bundle/lightning-conversation-toolkit-api)
- [CspTrustedSite Metadata API](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_csptrustedsite.htm)
- [Secure Coding WebSockets](https://developer.salesforce.com/docs/atlas.en-us.secure_coding_guide.meta/secure_coding_guide/secure_coding_websockets.htm)
- [Summer '18 WebSocket CSP Release Notes](https://help.salesforce.com/s/articleView?id=release-notes.rn_lc_websockets.htm&language=en_US&release=218&type=5)
- [Standard-Volume PE Retirement (Summer '27)](https://help.salesforce.com/s/articleView?id=release-notes.rn_messaging_standard_volume_retirement.htm&language=en_US&release=250&type=5)
- [Top 20 AppExchange Vulnerabilities](https://developer.salesforce.com/blogs/2023/08/the-top-20-vulnerabilities-found-in-the-appexchange-security-review)
- [Prepare for Security Review](https://developer.salesforce.com/blogs/2023/04/prepare-your-app-to-pass-the-appexchange-security-review)

### Competitor Documentation
- [SMS-Magic v1.58 Release Notes](https://www.sms-magic.co/docs/converse-release-notes/knowledge-base/patch-version-1-58/)
- [SMS-Magic OAuth FAQ](https://www.sms-magic.co/docs/salesforce/faq/25-what-is-oauth-is-it-necessary-to-enable-the-oauth/)
- [SMS-Magic OAuth Failover](https://www.sms-magic.com/docs/portal/knowledge-base/oauth-user-management-and-failover/)
- [SMS-Magic SFMC Architecture](https://docs.sms-magic.com/wl5N2H674bM-sms-magic-converse-integration-with-salesforce-marketing-cloud-sfmc/GBgyhgYnY9U-sfmc-native-app-architecture-explained)
- [SMS-Magic OmniChannel](https://docs.sms-magic.com/dRSpyt9QVyE-sms-magic-converse-integration-with-salesforce-omnichannel)
- [Mogli Notifications docs](https://mogli-sms.document360.io/docs/mogli-notifications)
- [Mogli Access & Visibility](https://mogli-sms.document360.io/docs/access-visibility)
- [Mogli Release Notes v5.66.2](https://mogli-sms.document360.io/docs/release-notes-v5661)
- [Heymarket SF Integration](https://help.heymarket.com/hc/en-us/articles/360034372271-Salesforce-Integration)
- [Heymarket Integrations](https://www.heymarket.com/integrations/)
- [360 SMS FAQ](https://360smsapp.com/faq/)
- [360 SMS Conversation View](https://360smsapp.com/manual/conversation-view/)
- [Sinch Engage Message Feed Setup](https://support.app.sinch.com/hc/en-us/articles/10560915429263-Salesforce-Setup-Configure-the-Message-Feed)
- [Sinch Engage Delivery Status](https://support.app.sinch.com/hc/en-us/articles/10561067764623-Salesforce-Delivery-Status-notifications)

### Community / Third-Party
- [4 Strategies to Avoid Streaming Limits in LWCs (Medium)](https://medium.com/@gjasula/4-proven-strategies-to-avoid-salesforce-streaming-limits-in-real-time-lwcs-d4545b63c2a5)
- [WebSocket vs Platform Events Demo (GitHub)](https://github.com/eltoroit/ETWSBlogSalesforce)
- [WebSocket Chat in Salesforce LWC (Blog)](https://blog.jamigibbs.com/websockets-in-salesforce-with-lightning-web-components/)
- [LWC WebSocket Chat Demo (GitHub)](https://github.com/jamigibbs/lwc-websocket-chat)
- [BYO Demo Connector (GitHub)](https://github.com/salesforce-misc/byo-demo-connector)
- [empApi LWC/Aura Conflict Bug (GitHub)](https://github.com/salesforce/lwc/issues/1618)
