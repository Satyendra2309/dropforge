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

## 3. Context diagram — next lesson
## 4. Service boundaries — next lesson
## 5. API contracts — next lesson
