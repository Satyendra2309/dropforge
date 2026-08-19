# Dropforge — High-Level Design

Living document, built section by section during Phase M0. Each section lands
after its lesson.

## 1. Requirements

### Functional
- Buyers: browse products; see upcoming drops with countdown; join the waiting
  room when a drop opens; receive a queue turn; hold an item; pay within the
  hold window (5 min); get notified of the outcome.
- Admins: create products; schedule drops (stock count, go-live time).

### Non-functional
- **Invariant (never breaks, even mid-crash): units sold ≤ units stocked.**
  Zero oversells — an oversell costs real refund money, not just UX.
- Fairness: waiting-room turns are first-come-first-served.
- Spike tolerance: survive the drop-open stampede without falling over.
- Bot resistance: one human ≈ one queue slot.
- SLO: join-queue p99 < 500 ms during peak; drop page p99 < 200 ms (cached).

## 2. Capacity estimates (flagship drop: 10,000 buyers, 100 items)

| Quantity | Estimate | Reasoning |
|---|---|---|
| Peak join rate | ~800 rps | 80% of 10k buyers click join within 10 s of open |
| Steady browse rate | ~5 rps | normal day; the spike is ~160× steady state |
| Successful orders | 100 per drop | stock is the ceiling — total write volume is tiny |
| Rejections | 9,900 per drop | 99% of buyers must lose; rejection must be the cheapest path |
| Live ticker connections | 10,000 concurrent WebSockets | connection state/memory, a separate axis from rps |
| Order storage | ~KBs per drop | storage is a non-problem |

### What bounds this system
Dropforge is **concurrency-bound**: the hard part is 10k simultaneous claims on
100 units, not data volume. Design consequences, in priority order:
1. Size for the spike, not the average (the waiting room absorbs it).
2. Keep the write path (reserve → hold → pay) tiny and protected.
3. Make rejection the cheapest operation in the system — losers never touch
   the database.

## 3. Context diagram

```mermaid
flowchart TB
    buyer([Buyer browser]) --> gw
    admin([Admin]) --> gw
    stripe([Stripe webhooks]) --> gw
    sim([Drop simulator - load and chaos]) --> gw

    subgraph edge [Edge]
        gw[gateway<br/>auth, rate limit, routing]
    end

    gw --> catalog[catalog]
    gw --> drop[drop<br/>schedule, waiting room, ticker]
    gw --> order[order]
    gw --> user[user/auth]
    gw --> payment[payment]

    drop --> inventory[inventory<br/>stock, holds]
    order --> inventory
    payment -->|Stripe API| stripeapi([Stripe])

    subgraph events [Kafka]
        bus[[drop.opened / order.placed / payment.succeeded / hold.expired]]
    end

    drop -. produce/consume .-> bus
    order -. produce/consume .-> bus
    inventory -. produce/consume .-> bus
    payment -. produce/consume .-> bus
    notification[notification] -. consume .-> bus

    catalog --- pgc[(Postgres)]
    inventory --- pgi[(Postgres)]
    order --- pgo[(Postgres)]
    payment --- pgp[(Postgres)]
    user --- pgu[(Postgres)]
    drop --- redis[(Redis<br/>queue, cache, locks)]
```

Every service also emits metrics/logs/traces to the observability stack
(Prometheus, Loki, Tempo) — omitted above for readability.

## 4. Service boundaries and their defense

**Rule: a service boundary is defined by the data it owns and the invariant it
protects — not by nouns in the problem statement.** Split where consistency
requirements change: atomic-together stays together; eventual-consistency may
be separated.

| Service | Owns | Why it is a separate service |
|---|---|---|
| inventory | stock counts, purchase holds | protects THE invariant (no oversell); highest-contention atomic writes. Holds live here because a held unit IS stock state — splitting that across services splits the invariant |
| order | order lifecycle records | write-light, history-heavy lifecycle tracking; no shared invariant with stock |
| catalog | products, prices | read-heavy, cacheable, staleness-tolerant — opposite consistency needs from inventory |
| drop | drop schedule, waiting-room queue, live ticker | traffic-shaping state (who may try, when), distinct from stock state (who succeeds) |
| payment | charges, Stripe integration | quarantines an external dependency, its failure modes, and its secrets |
| notification | delivery attempts | pure async consumer; nobody waits on it; failures isolate to a DLQ |
| user | identities, credentials, roles | identity is its own domain and security boundary |
| gateway | (no domain data) | edge concerns — authn enforcement, rate limiting, routing — implemented once, not seven times |

Deliberately absent: a cart service. Flash drops have no cart — the purchase
hold plays that role. A boundary drawn around a noun with no invariant behind
it is how distributed monoliths start.

## 5. API contracts — next lesson
