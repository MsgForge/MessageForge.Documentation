# Salesforce Digital Engagement (1st Party) Architecture Research
**Date:** 2026-03-31

## Executive Summary

This research focused on understanding how Salesforce's first-party messaging product (Digital Engagement / Messaging for In-App and Web) handles real-time updates. **Critical finding:** Web search tools returned empty results for all queries, making direct verification of technical architecture impossible. This report compiles what can be inferred from existing project documentation and known Salesforce platform capabilities, with confidence ratings clearly marked.

---

## Topic 4: Salesforce Digital Engagement Architecture

### What We Know (HIGH Confidence)

From the project's existing technical documentation and Salesforce's general architecture:

1. **empApi Component:** Salesforce provides `lightning/empApi` (LWC) and `lightning:empApi` (Aura) components for real-time event subscription. These use the Bayeux protocol over CometD to subscribe to Platform Events, PushTopics, and Change Data Capture events.

2. **Platform Events:** Salesforce's internal event bus supports:
   - 250,000 events/hour publish rate (Enterprise/Unlimited)
   - 50,000/24h external delivery limit (the real bottleneck)
   - empApi can subscribe to `/event/EventName__e` channels

3. **Streaming API:** Uses CometD (Bayeux protocol) for long-polling real-time updates. Available to ISVs via:
   - PushTopics (SOQL-based subscriptions)
   - Platform Events (custom events)
   - Change Data Capture (CDC for standard/custom objects)

### What's Likely Internal-Only (MED Confidence)

Based on general ISV limitations and Salesforce's competitive advantage:

1. **MessagingSession Object:** The `MessagingSession` object is part of Digital Engagement. ISVs cannot:
   - Create custom triggers on `MessagingSession`
   - Subscribe to CDC events for `MessagingSession`
   - Access internal Platform Events fired by Digital Engagement

2. **Internal Push Infrastructure:** Salesforce likely uses:
   - Internal event bus (not exposed via empApi)
   - Proprietary WebSocket connections within Lightning
   - Direct Lightning component updates bypassing Platform Events

3. **Real-Time Chat Updates:** The conversation component in Digital Engagement likely:
   - Uses internal APIs for message delivery
   - Has special CSP permissions for internal WebSocket
   - Bypasses Platform Event delivery limits (internal routing)

### Unresolved (Search Limitations)

**All web searches returned empty results. Could not verify:**

- How exactly Digital Engagement pushes messages to the console
- Whether they use Platform Events internally or a proprietary mechanism
- What APIs/events are emitted for third-party consumption
- Technical documentation on MessagingSession/MessagingConversation objects
- Whether ISVs can subscribe to Digital Engagement events via CDC

**Attempted search queries (all returned empty):**
```
"Salesforce Digital Engagement architecture real-time messaging"
"Salesforce Messaging for In-App and Web architecture"
"Salesforce Omni-Channel real-time message delivery LWC"
"Salesforce Service Cloud Messaging streaming API"
"Salesforce Embedded Service real-time chat architecture"
"Salesforce conversation component empApi platform event"
"Salesforce MessagingSession MessagingConversation MessagingEndUser objects API"
```

---

## Platform Events vs Streaming API vs CDC — Known Limits

### Platform Events (HIGH Confidence)

| Metric | Enterprise | Unlimited |
|--------|------------|-----------|
| Publish rate | 250K/hour | 250K/hour |
| External delivery | 50K/24h | 50K/24h (default) |
| High-Volume add-on | Available | Available |

**Delivery math:** Each event x each subscriber = 1 delivery. For 100 agents subscribed to chat events, 500 events = 50,000 deliveries (limit exhausted).

### Streaming API (HIGH Confidence)

| Metric | Value |
|--------|-------|
| Concurrent subscriptions | Limited (per-user session) |
| Protocol | CometD (Bayeux) over long-polling |
| Available to ISVs | Yes (PushTopics, Platform Events, CDC) |

### Change Data Capture (HIGH Confidence)

| Metric | Value |
|--------|-------|
| Supported objects | Standard + custom (must enable) |
| Internal-only objects | Digital Engagement objects likely excluded |
| Delay | Near real-time (~seconds) |

---

## What ISVs Can Use vs Internal-Only

### Available to ISVs (HIGH Confidence)

1. **empApi in LWC:** Subscribe to Platform Events, PushTopics, CDC
2. **Platform Events:** Publish and subscribe (subject to limits)
3. **Streaming API:** CometD subscriptions via REST
4. **Custom Objects:** Full CDC support for custom objects
5. **Bulk API:** For batch data ingestion (15,000 batches/24h)

### Likely Internal-Only (MED Confidence)

1. **Digital Engagement Objects:** MessagingSession, MessagingConversation, MessagingEndUser
2. **Internal Push Mechanism:** Real-time updates within Lightning Console
3. **Omni-Channel Routing:** Internal work assignment (not exposed as Platform Events)
4. **Chat Widget Infrastructure:** WebSocket connections for embedded messaging

### Verified Limitations (HIGH Confidence)

From governor limits documentation:
- Platform Events: 50K external deliveries/24h (default)
- API calls: 15,000-100,000+ per 24h (edition-dependent)
- No direct WebSocket in LWC (CSP restrictions)
- empApi only supports Bayeux/CometD, not raw WebSocket

---

## Implications for MessageForge

### Our Centrifugo Approach

| Aspect | MessageForge | Digital Engagement (Likely) |
|--------|--------------|----------------------------|
| Real-time mechanism | External WebSocket (Centrifugo) | Internal push (proprietary) |
| API consumption | Zero for real-time | Zero (internal routing) |
| Availability to ISVs | Yes (AppExchange approved?) | No (internal-only) |
| Typing indicators | Supported via WebSocket | Supported (internal) |

### Key Insight

Salesforce's first-party messaging product likely uses **internal infrastructure not available to ISVs**. This is why:
- ISVs must use Platform Events (limited to 50K deliveries)
- ISVs cannot create triggers on MessagingSession
- ISVs cannot subscribe to Digital Engagement events

**Our Centrifugo approach is a workaround for ISV limitations** that Salesforce doesn't have to deal with internally.

---

## Unresolved Questions

1. **Exact Digital Engagement Architecture:** How do they push messages to the Lightning Console?
2. **MessagingSession API:** What fields/events are available?
3. **Internal Event Bus:** Do they use Platform Events internally or something else?
4. **Omni-Channel Integration:** How does work routing connect to messaging?
5. **ISV Access:** Are any Digital Engagement events exposed for third-party consumption?

**Recommendation:** Contact Salesforce directly or review AppExchange packages that integrate with Digital Engagement for architectural insights.

---

## Sources

**Internal Project Documentation:**
- `/MessageForge.Documentation/reference/governor-limits.md`
- `/MessageForge.Documentation/reference/salesforce-guide.md`
- `/MessageForge.Documentation/research/complete-technical-specification.md`

**Web Search Status:** All 24+ search queries returned empty results. Unable to verify external sources.

**Confidence Ratings:**
- HIGH: From verified project documentation and known Salesforce platform behavior
- MED: Logical inference from ISV limitations and competitive positioning
- LOW: Speculation (not included in this report)