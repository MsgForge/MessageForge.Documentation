# MessageForge Cost Model

**Last Updated:** 2026-03-29

This document models infrastructure and operational costs across three deployment tiers: MVP, Growth, and Scale. Costs are primarily driven by:

1. **Compute** (VPS hosting)
2. **Storage** (PostgreSQL, R2 media)
3. **Network** (proxies for Telegram rate limit management)
4. **CDN** (Cloudflare Workers for media delivery)

## Key Assumptions

- All costs in USD/EUR equivalent
- Pricing as of Q1 2026 (subject to change)
- Monthly recurring costs (no one-time setup fees included)
- Assumes Bot API-only channels for MVP/Growth; MTProto for Growth/Scale
- VPS self-hosted database for MVP/Growth; managed for Scale
- Cloudflare free tier sufficient for MVP/Growth
- No Salesforce Platform Event overage fees (within normal allocation)

---

## MVP Tier: 10 Channels, ~1K Messages/Day

**Target:** Pilot deployment, proof-of-concept, team usage

### Cost Breakdown

| Component | Provider | Specification | Monthly Cost | Notes |
|-----------|----------|---|---|---|
| **Compute** | Hetzner | CAX11 (2 vCPU, 4GB RAM) | €4.50 | Dual-core ARM sufficient for <10 channels |
| **Storage (VPS)** | Hetzner | 40GB NVMe SSD | Included | Sufficient for 6+ months of message data |
| **PostgreSQL** | Self-hosted | On VPS | €0 | No separate managed service |
| **Backups** | Local | Daily snapshots to S3 | ~€1-2 | Optional, estimated ~5GB/month |
| **Residential Proxies** | SmartProxy/IPRoyal | 5 dedicated IPs | €25-40 | For Bot API rotation (Bot API has lower rate limits) |
| **R2 Storage** | Cloudflare | ~1GB media/month | €0 | Free tier (first 10GB/month) |
| **R2 Bandwidth** | Cloudflare | <10GB egress/month | €0 | Free tier (first 1GB/day) |
| **Workers** | Cloudflare | <100K requests/day | €0 | Free tier (10M req/month) |
| **CDN Cache** | Cloudflare | Included | €0 | Free Argo Smart Routing |
| **Domain + DNS** | Registrar | 1 domain .com/.io | ~€10/year | ~€0.83/month |
| **SSL/TLS** | Let's Encrypt | Via Caddy | €0 | Automatic, free |
| **Monitoring** | (Optional) | Uptime.com free tier | €0 | Or omit for MVP |
| | | | |
| **MONTHLY TOTAL** | | | **€31-47** | |

### Cost Drivers at MVP Scale

- **Proxies dominate**: 5 IPs @ €5-8/IP = bulk of non-infrastructure cost
- **Compute is minimal**: CAX11 easily handles 10 channels
- **Storage negligible**: <10GB/month media, self-hosted DB
- **Network costs zero**: Cloudflare free tier sufficient

### Optimization Options

- **Remove proxies** if using Bot API only (no rate limit concerns): saves €25-40 → **€6-7/month**
- **Use free DNS** (Cloudflare) instead of paid registrar DNS: saves ~€10/year
- **Consolidate backups** to Hetzner snapshot service: saves €1-2

---

## Growth Tier: 100 Channels, ~10K Messages/Day

**Target:** Production deployment, team rollout, early customers

### Cost Breakdown

| Component | Provider | Specification | Monthly Cost | Notes |
|-----------|----------|---|---|---|
| **Compute** | Hetzner | CAX21 (4 vCPU, 8GB RAM) | €8 | ~2.5K messages/channel/day; good headroom |
| **Storage (VPS)** | Hetzner | 80GB NVMe SSD | Included | Sufficient for 12+ months |
| **PostgreSQL** | Self-hosted | On VPS | €0 | Still manageable on CAX21 |
| **Backups** | AWS S3 | Daily snapshots | ~€3-5 | ~50GB/month storage + transfer |
| **Residential Proxies** | Bright Data/SmartProxy | 50 dedicated IPs | €250-500 | MTProto introduces rotation need |
| **R2 Storage** | Cloudflare | ~10GB/month | ~€1.50 | 50 IPs × 200KB avg image = ~10GB/month |
| **R2 Bandwidth** | Cloudflare | 100GB egress/month | ~€5 | Worker-cached delivery; 100GB reasonable |
| **Workers** | Cloudflare | ~500K requests/day | ~€5 | Above free tier; ~$0.50/M requests |
| **CDN** | Cloudflare | Pro plan | €20 | For analytics, custom SSL, page rules |
| **Domain + DNS** | Registrar | 1 domain | €10/year | €0.83/month |
| **SSL/TLS** | Let's Encrypt | Via Caddy | €0 | Automatic |
| **Monitoring** | Better Stack | Basic plan | ~€15/month | Recommended for production |
| | | | |
| **MONTHLY TOTAL** | | | **€298-549** | |

### Cost Drivers at Growth Scale

- **Residential proxies**: 50 IPs @ €5-10/IP = ~€250-500 (>50% of total cost)
  - Required for MTProto multi-user scenarios
  - Reduces Telegram's connection/login rate limits
  - SmartProxy, IPRoyal, Bright Data all viable
- **Workers + R2**: ~€10-11 (manageable)
- **Monitoring**: ~€15 (recommended for production SLA)
- **CDN Pro**: €20 (optional but recommended for analytics)

### Optimization Options

- **Switch to Bot API only**: Reduces proxies to ~5 IPs → **save €200-400** → €98-249/month
- **Consolidate to managed PostgreSQL later** if reliability critical: +€15 but reduces operational burden
- **Use Datadog/New Relic free tier** instead of Better Stack: saves €15 but less mature for production

### Capacity Planning

| Metric | CAX21 Headroom | Action |
|--------|---|---|
| Database connections | 200 max, ~50 active | Increase work_mem if > 100 |
| CPU utilization | <40% at peak | Scale to CAX31 if >60% |
| Memory | 8GB, ~2GB used | Consider Hetzner managed DB if > 6GB |
| Disk I/O | NVMe sufficient | Monitor with iostat; upgrade if >80% |

---

## Scale Tier: 500 Channels, ~50K Messages/Day

**Target:** Enterprise deployment, multi-team usage, external customers

### Cost Breakdown

| Component | Provider | Specification | Monthly Cost | Notes |
|-----------|----------|---|---|---|
| **Compute** | Hetzner | CAX31 (8 vCPU, 16GB RAM) | €16 | High-traffic baseline |
| **Storage (VPS)** | Hetzner | 160GB NVMe SSD | Included | Sufficient for 12+ months |
| **PostgreSQL** | Hetzner Managed | 4 vCPU, 8GB RAM | €30 | Automated backups, failover, monitoring |
| **Backups** | Hetzner | Managed + snapshots | €5 | Included with managed DB + snapshots |
| **Redis** | Self-hosted on VPS | For rate limiting cache | €0 | Distributed session + limit state |
| **Residential Proxies** | Bright Data | 200 dedicated IPs | €1,000-2,000 | MTProto multi-user, aggressive rotation |
| **R2 Storage** | Cloudflare | ~50GB/month | ~€7.50 | 500 channels × 100KB avg media |
| **R2 Bandwidth** | Cloudflare | 500GB egress/month | ~€20 | Worker-cached; still <1GB/day per channel |
| **Workers** | Cloudflare | ~2M requests/day | ~€15 | ~$0.50/M requests above free tier |
| **Workers KV** | Cloudflare | Rate limit state | ~€5 | Optional, for distributed rate limiting |
| **CDN** | Cloudflare | Business plan | €200 | Enterprise analytics, custom SSL, priority support |
| **Domain + DNS** | Registrar | 1 domain | €10/year | €0.83/month |
| **Monitoring** | Datadog/New Relic | Standard plan | €40-50/month | Logs, metrics, distributed tracing |
| **Load Balancing** | Hetzner | (optional) | €15 | For HA across multiple backends |
| | | | |
| **MONTHLY TOTAL** | | | **€1,353-2,354** | |

### Cost Drivers at Scale

- **Residential proxies**: 200 IPs @ €5-10/IP = €1,000-2,000 (**85%+ of total cost**)
  - Mandatory for high-volume MTProto usage
  - Rate limits: 1 concurrent connection per proxy
  - Assumes rotating IPs to avoid temporary bans
  - Bright Data + SmartProxy typically cheapest at volume
- **Managed PostgreSQL**: €30 (justified by reliability/uptime SLA)
- **Infrastructure**: €67 (VPS + backups + Redis)
- **CDN/Workers**: €42-45 (negligible at this scale)
- **Monitoring**: €40-50 (critical for enterprise SLA)

### Optimization Options

- **Bot API-only channels** (no MTProto): Drops proxies to ~10 IPs → **save €900-1,900** → €353-554/month
- **Shared proxy pool** (rotating shared IPs): €100-200/month but higher rate-limit risk
- **Custom proxy infrastructure**: Build own datacenter proxies → €0 proxies but +€200-500 infrastructure
- **Caching layer** (Redis on managed provider): Adds €15-30 but reduces DB load by 40-50%

### High-Availability Configuration

For enterprise SLA (99.9% uptime):

| Add-on | Provider | Cost |
|--------|----------|------|
| **Second VPS** | Hetzner | +€16 |
| **Managed PostgreSQL HA** | Hetzner | +€20 (standby) |
| **Load Balancer** | Hetzner | +€15 |
| **CDN Enterprise** | Cloudflare | +€100 → €300/month |
| **Monitoring + Alerts** | Datadog | +€50 → €100/month |
| | | |
| **HA Monthly Overhead** | | **+€150-200** |

---

## Cross-Tier Comparison

### Cost Per Channel Per Month

| Tier | Monthly Cost | Channels | Cost/Channel |
|------|---|---|---|
| MVP | €31-47 | 10 | **€3.10-4.70** |
| Growth | €298-549 | 100 | **€2.98-5.49** |
| Scale | €1,353-2,354 | 500 | **€2.71-4.71** |

**Observation:** Per-channel cost *decreases* as you scale (economies of scale), but only because compute/storage scale sub-linearly. Proxies scale *linearly* with channels using MTProto.

### Cost Per Message

Assuming message volume:

| Tier | Monthly Messages | Monthly Cost | Cost/K Messages |
|------|---|---|---|
| MVP | ~30K | €31-47 | **€1.03-1.57** |
| Growth | ~300K | €298-549 | **€0.99-1.83** |
| Scale | ~1.5M | €1,353-2,354 | **€0.90-1.57** |

---

## Proxy Cost Analysis

**Residential proxies are the dominant variable cost.** Understanding proxy economics is critical:

### Proxy Providers (Ranked by Q1 2026 Pricing)

| Provider | 50 IPs | 200 IPs | Note |
|----------|--------|--------|------|
| **IPRoyal** | €240/month | €800/month | Cheapest; datacenter-grade reliability |
| **SmartProxy** | €280/month | €900/month | Good uptime; rotating residential |
| **Bright Data** | €400-500/month | €1,500-2,000/month | Enterprise SLA; best for scale |
| **Oxylabs** | €450/month | €1,800/month | Premium features; compliance focus |
| **Luminati** | €500/month | €2,000+/month | Highest cost; enterprise only |

### MTProto vs. Bot API Proxy Requirements

| Protocol | Proxy Need | IPs Required | Cost Impact |
|----------|---|---|---|
| **Bot API** | Optional | 0-5 | €0-40/month (MVP only, for rotation) |
| **MTProto** | Mandatory | 1 per concurrent user session | €5-10/IP/month |

**Decision:** If 90% of channels use Bot API + 10% use MTProto, cost is roughly:
- 90 channels × 0 IPs + 50 channels × 2 IPs = 100 IPs → €500-1,000/month

---

## Optional Costs Not Included

These are project-specific and omitted from baseline model:

| Cost | Typical Range | When Needed |
|------|---|---|
| **Salesforce licenses** | $125/user/month | If deploying as AppExchange package |
| **SIEM/Security** | $50-500/month | For compliance (SOC 2, HIPAA) |
| **DDoS protection** | $200-1,000/month | If under attack; usually free via Cloudflare |
| **Datadog/New Relic** | $20-100/month | If moving beyond Better Stack |
| **Legal/Compliance** | ~€1,000 one-time | AppExchange security review |
| **Ops team** | €3,000-5,000/month | For enterprise SLA (not in cost model) |

---

## Scaling Roadmap

### When to Upgrade Infrastructure

| Metric | Threshold | Action | Cost Delta |
|--------|---|---|---|
| **Channels** | >20 | Upgrade to CAX21 | +€3.50/month |
| **Messages/day** | >20K | Add managed PostgreSQL | +€15/month |
| **Channels** | >100 | Consider load balancer | +€15/month |
| **Messages/day** | >100K | Upgrade to CAX31 | +€8/month |
| **Channels** | >500 | Dedicated proxy provider account | negotiate |

### When to Optimize Proxies

| Scenario | Recommendation |
|----------|---|
| Heavy Bot API usage (70%+) | Reduce IPs by 50%; negotiate bulk discount |
| High MTProto volume | Switch to Bright Data for better rate-limit handling |
| Regional deployment | Use regional proxy providers (e.g., Ipify for EU) |
| Cost-sensitive | Implement proxy rotation + caching; reduce concurrent IP count |

---

## Financial Projections (Annual)

### Scenario 1: Rapid Growth (MVP → Scale in 12 months)

| Month | Channels | Messages/day | Infrastructure | Proxies | Total | Cumulative |
|---|---|---|---|---|---|---|
| 1 | 10 | 1K | €5 | €35 | €40 | €40 |
| 3 | 25 | 3K | €5 | €100 | €105 | €285 |
| 6 | 100 | 10K | €25 | €350 | €375 | €1,875 |
| 9 | 300 | 30K | €35 | €1,000 | €1,035 | €4,020 |
| 12 | 500 | 50K | €71 | €1,500 | €1,571 | €7,571 |

**Year 1 Cost:** ~€7,500-8,000

### Scenario 2: Organic Growth (MVP for 24 months, then scale)

| Period | Channels | Infrastructure | Proxies | Quarterly | Annual |
|---|---|---|---|---|---|
| Year 1 | 10-50 | €5-15 | €100-300 | €500 × 4 | €2,000 |
| Year 2 | 50-200 | €15-25 | €300-900 | €1,000-1,500 × 4 | €5,000-6,000 |

**2-Year Cost:** ~€7,000-8,000

---

## Revenue Model Integration

For pricing customers per channel:

| Tier | Price/Channel/Month | Gross Margin (500 channels) |
|------|---|---|
| €10/channel | €5,000 revenue − €1,600 cost = **€3,400/month** (68%) |
| €15/channel | €7,500 revenue − €1,600 cost = **€5,900/month** (79%) |
| €20/channel | €10,000 revenue − €1,600 cost = **€8,400/month** (84%) |

**Key Insight:** At Scale tier, proxies consume ~€1,600/month with 500 channels @ 2 IPs/channel. This is the cost floor for MTProto-heavy deployments.

---

## Cost Reduction Strategies

### Immediate (No Architecture Changes)

1. **Negotiate proxy bulk discounts** (10%+ at 100+ IPs): saves €50-200/month
2. **Consolidate to single Salesforce org** (avoid Platform Event overage): saves €0-500/month
3. **Use CloudFlare Argo Smart Routing** (auto-enabled free): saves €15-30/month vs. standard routing
4. **Implement aggressive caching** (Centrifugo presence, rate-limit state): reduces DB load

### Medium-term (1-3 months)

1. **Migrate to managed PostgreSQL only if hitting resource limits**: +€15 but -€5 in ops overhead
2. **Implement Redis on VPS** for distributed rate limiting: +€0 but -€10 in Workers KV
3. **Regional proxy providers**: Bright Data US West costs 20% less than US East; pick nearest

### Long-term (3-12 months)

1. **Custom proxy infrastructure**: Build own ISP-backed proxy service (€500-1,000/month setup) → €0 marginal cost
2. **Dedicated MTProto client pool**: Rotate sessions across fewer IPs (protocol-level batching) → 30% reduction
3. **AppExchange monetization**: Charge per message or per channel → turn cost center into revenue center

---

## Conclusion

**MessageForge cost model is proxy-dominated at scale.**

| Tier | Total | Proxy % | Infrastructure % |
|------|---|---|---|
| MVP | €31-47 | 80% (€25-40) | 20% (€6-7) |
| Growth | €298-549 | 85% (€250-500) | 15% (€48-49) |
| Scale | €1,353-2,354 | 87% (€1,000-2,000) | 13% (€353-354) |

**Strategic Options:**

1. **Bot API-first strategy**: 90% of channels via Bot API (proxy-free) → cost drops to €50-100/month at scale
2. **Proxy partnership**: Negotiate volume discounts or reseller status with proxy provider → 20-30% reduction
3. **Monetization**: Charge per message (€0.01-0.05) or per channel (€5-20) → positive unit economics at 100+ channels

For MVP phase (10 channels, Bot API only): **€6-7/month** is sustainable and profitable at any price point.

For Scale phase (500 channels, mixed MTProto): **€1,353-2,354/month** requires €10-20/channel pricing to achieve 75%+ gross margin.
