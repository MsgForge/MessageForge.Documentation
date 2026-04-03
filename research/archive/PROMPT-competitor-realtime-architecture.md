# Research Prompt: Competitor Real-Time Architecture Analysis

## Objective

Deep web research into HOW AppExchange messaging apps (SMS-Magic, Mogli, Heymarket, 360 SMS, Sinch Engage, Salesforce Digital Engagement) handle real-time message delivery, chat UI updates, and file display in their Salesforce integrations. We need to understand their architectural choices to validate or improve our Centrifugo-based approach.

**This is NOT a go/no-go analysis. We are building this. We need to learn from competitors to build it BETTER.**

---

## Context

**MessageForge** is a 2GP managed package connecting Telegram (and future messengers) to Salesforce CRM via Go middleware. Our architecture uses:
- **Go middleware** — receives messages from Telegram, saves to Salesforce via REST API
- **Centrifugo** — external WebSocket server for real-time push to LWC (zero SF API consumption)
- **Salesforce ContentVersion** — all media files stored in SF Files (ADR-20, decided)
- **LWC** — Lightning Web Components for chat UI inside Salesforce

**Our concern:** We use Centrifugo to avoid SF API limits for real-time updates. How do competitors solve this same problem? Do they use external WebSocket servers? Platform Events? Polling? Streaming API? Something else?

---

## Research Topics

### Topic 1: SMS-Magic (Conversive) — Real-Time Architecture

Search queries:
- "SMS-Magic Salesforce real-time message delivery architecture"
- "SMS-Magic Conversive LWC chat component how it works"
- "SMS-Magic Salesforce Platform Events messaging"
- "SMS-Magic Salesforce Streaming API push notifications"
- "SMS-Magic conversation view real-time updates"
- site:sms-magic.com "real-time" OR "live" OR "push" OR "streaming" OR "websocket"
- site:sms-magic.com "platform event" OR "streaming api" OR "empApi"

Find out:
- How does their chat UI update when a new message arrives?
- Do they use Platform Events, Streaming API, polling, or external WebSocket?
- How do they handle typing indicators / read receipts (if at all)?
- Do they have a middleware server or direct SF-to-provider integration?
- What happens when an agent has the chat open and a new message comes in?

### Topic 2: Mogli SMS & WhatsApp — Real-Time Architecture

Search queries:
- "Mogli SMS Salesforce real-time notifications"
- "Mogli Salesforce conversation component architecture"
- "Mogli SMS platform events streaming"
- site:mogli.com "real-time" OR "live updates" OR "push notification"
- "Mogli SMS Salesforce LWC chat"

Find out:
- Same questions as Topic 1
- Mogli is "100% native Salesforce" — how do they do real-time without external servers?
- Do they use empApi in LWC for Platform Events?
- What's the refresh mechanism — push or pull?

### Topic 3: Heymarket — Real-Time Architecture

Search queries:
- "Heymarket Salesforce integration architecture"
- "Heymarket real-time sync Salesforce"
- "Heymarket Salesforce chat component live updates"
- site:help.heymarket.com "real-time" OR "sync" OR "push" OR "webhook"
- "Heymarket Salesforce polling interval"

Find out:
- Heymarket syncs "every minute" (from their docs) — is this polling? Or has it improved?
- Do they have a real-time component or is it always delayed?
- Do they have their own external servers + sync, or is everything in SF?

### Topic 4: Salesforce Digital Engagement (1st Party) — Architecture

Search queries:
- "Salesforce Digital Engagement architecture real-time messaging"
- "Salesforce Messaging for In-App and Web architecture"
- "Salesforce Omni-Channel real-time message delivery LWC"
- "Salesforce Service Cloud Messaging streaming API"
- "Salesforce Embedded Service real-time chat architecture"
- "Salesforce conversation component empApi platform event"
- developer.salesforce.com "messaging" "real-time" "platform event"

Find out:
- How does Salesforce's OWN messaging product handle real-time updates?
- They likely use internal APIs not available to ISVs — document what those are
- What's available to ISVs vs what's internal-only?
- Do they use Platform Events, Change Data Capture, Streaming API, or internal push?

### Topic 5: Platform Events vs Streaming API vs Change Data Capture — Real Limits

Search queries:
- "Salesforce Platform Events daily limit Enterprise Unlimited 2025 2026"
- "Salesforce Platform Events high volume pricing"
- "Salesforce Streaming API concurrent subscribers limit"
- "Salesforce Change Data Capture vs Platform Events for chat"
- "Salesforce empApi LWC real-time updates performance"
- "Salesforce Platform Events vs external WebSocket ISV"
- "Salesforce high volume platform events pricing per event"

Find out:
- **Platform Events:** Exact daily delivery limits per edition. Cost of High-Volume add-on.
- **Streaming API:** Max concurrent subscribers. Can it replace WebSocket?
- **Change Data Capture (CDC):** Does it fire on custom object insert? Limits? Delay?
- **empApi in LWC:** Max concurrent subscriptions per user? Performance at scale?
- For 100 agents, 10,000 messages/day: which approach hits limits first?

### Topic 6: External WebSocket in AppExchange — Is It Allowed?

Search queries:
- "AppExchange security review external WebSocket connection"
- "AppExchange managed package external server WebSocket"
- "Salesforce ISV external real-time server AppExchange approved"
- "AppExchange CSP Trusted Sites WebSocket wss://"
- "Salesforce managed package connect external service real-time"

Find out:
- Can an AppExchange package connect LWC to an external WebSocket server (like Centrifugo)?
- What CSP/security requirements exist?
- Are there approved apps that do this? (Slack integration? Twilio? Others?)
- What does the security review process require for external connections?

### Topic 7: Comparative Architecture Table

Based on all findings, build this table:

| App | Real-Time Method | External Server? | Latency | SF Limits Consumed | Typing/Presence? |
|---|---|---|---|---|---|
| SMS-Magic | ? | ? | ? | ? | ? |
| Mogli | ? | ? | ? | ? | ? |
| Heymarket | ? | ? | ? | ? | ? |
| SF Digital Engagement | ? | ? | ? | ? | ? |
| 360 SMS | ? | ? | ? | ? | ? |
| Sinch Engage | ? | ? | ? | ? | ? |
| **MessageForge (ours)** | Centrifugo WebSocket | Yes (Hetzner) | ~50ms | Zero | Yes (planned) |

### Topic 8: Lessons for MessageForge

Based on competitor analysis:
- What can we learn from how they solved real-time updates?
- Is our Centrifugo approach unique? Is that an advantage or risk for AppExchange review?
- Should we have a FALLBACK to Platform Events if Centrifugo is unreachable?
- Do any competitors offer typing indicators / presence — or is that a differentiator for us?
- What's the standard UX pattern: instant push or "pull to refresh"?

---

## Output Format

Save to: `MessageForge.Documentation/research/competitor-realtime-architecture.md`

### Structure:
```markdown
# Competitor Real-Time Architecture Analysis
**Date:** [today]

## Executive Summary
[3-5 sentences: what we learned, how competitors handle real-time, what it means for us]

## Topic 1: SMS-Magic
### Findings
[Bullet points with source URLs]
### Confidence: HIGH/MED/LOW

## Topic 2: Mogli
[same format]

...

## Topic 7: Comparative Architecture Table
[The table from above, filled in]

## Topic 8: Lessons for MessageForge
### What We Should Copy
### What We Do Better
### Risks to Address
### Recommended Changes (if any)

## Unresolved Questions
[What we couldn't find — needs direct testing or contacting competitors]

## Sources
[All URLs referenced]
```

---

## Research Rules

1. **Use WebSearch and WebFetch extensively.** Every claim needs a source URL.
2. **Rate confidence:** HIGH = official docs/confirmed. MED = community/multiple sources. LOW = single source/speculation.
3. **If you can't find how a competitor works internally** — say so honestly. Don't guess. Note it as "Unresolved."
4. **Focus on ARCHITECTURE, not features.** We don't care about their feature list. We care about HOW they push messages to the UI in real-time.
5. **Check AppExchange listings, help docs, developer docs, Trailhead, StackExchange, Reddit, and GitHub** for each competitor.
6. **This is research, not implementation.** Don't write code. Write findings.
