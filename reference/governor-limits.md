# Salesforce Governor Limits Reference

Governor limits are hard runtime quotas enforced by Salesforce because it's multi-tenant. If your code exceeds a limit, it throws an unhandleable exception and the transaction rolls back.

## Per Apex Transaction

Every single request (REST call, trigger execution, LWC controller method) is one transaction.

| Limit | Synchronous | Asynchronous |
|---|---|---|
| SOQL queries | 100 | 200 |
| DML statements (insert/update/delete) | 150 | 150 |
| Records retrieved by SOQL | 50,000 | 50,000 |
| Heap size | 6 MB | 12 MB |
| CPU time | 10,000 ms | 60,000 ms |
| Callouts (HTTP requests to external) | 100 | 100 |
| Callout timeout | 120 seconds | 120 seconds |

## Platform-Wide Limits (per org, per 24 hours)

| Limit | Value |
|---|---|
| API requests (REST/SOAP) | Varies by edition + licenses (Enterprise: 1,000/user/24h, min 15,000) |
| Platform Events published | Up to 250,000/hour (Performance/Unlimited) |
| Platform Events delivered (external) | 50,000/24h default — **the real bottleneck** |
| Bulk API batches | 15,000/24h rolling |
| Data storage | ~10 MB per user license |
| File storage | ~2 GB per user license |

### Platform Event Delivery Math

Each event x each subscriber = one delivery.

Example: 1 event published, 3 LWC components subscribed = 3 deliveries consumed.

With 50K/24h limit and active chat, this can exhaust at scale. At MVP scale (500-2K messages/day, 3-10 agents), PE daily deliveries stay within limits. LWC subscribes via `lightning/empApi` for real-time updates (ADR-21). For scale beyond PE limits, evaluate High-Volume PE add-on or Pub/Sub API (ADR-19).

## Why This Matters for Go Middleware

Every REST API call your Go server makes to Salesforce counts against the org's 24-hour API limit.

- **Bad:** One API call per incoming Telegram message -> busy chat group exhausts limit in hours
- **Good:** Batch messages via Platform Events or Bulk API
- **Best:** Use Pub/Sub API (gRPC) for real-time, Bulk API 2.0 for archives

## Edition Comparison

| Edition | API Calls/24h | Platform Events | File Storage | Price |
|---|---|---|---|---|
| Essentials | 15,000 | Limited | 1 GB | ~$25/user/mo |
| Professional | 15,000 | Limited | 10 GB | ~$80/user/mo |
| Enterprise | 100,000+ | 250K/hr publish | 10 GB | ~$165/user/mo |
| Unlimited | Unlimited | 250K/hr publish | Unlimited | ~$330/user/mo |

**Target: Enterprise minimum.** Professional is too constrained for active Telegram parsing.

## Practical Design Rules

1. **Never do 1 SOQL per record** — query collections, filter in memory
2. **Never do 1 DML per record** — collect into lists, single bulk DML
3. **Never make 1 callout per event** — batch, queue, aggregate
4. **Always check `Limits.getQueries()` vs `Limits.getLimitQueries()`** in complex flows
5. **Use async processing** (Queueable, Batch Apex) for heavy operations — doubles most limits
