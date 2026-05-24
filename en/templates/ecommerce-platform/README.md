# E-commerce Platform · Architecture Template

> **Representative products**: Amazon, Shopify, Taobao, all manner of online retail and trading platforms
> **One-line positioning**: Under massive concurrency, make sure "money for goods, square and clean" **never goes wrong** — and survive the flash flood of a mega-sale.

---

## 1. One-line positioning

An e-commerce platform = **a "read-heavy, write-light, reads-can-be-loose, writes-must-be-exact-to-the-penny" transaction machine** + **a peak-shaving rig built specifically to ride out the mega-sale flood**.

The most counterintuitive thing about it, architecturally: its biggest difference from an ordinary website isn't "more pages" — it's that **two opposite kinds of data live here at the same time**. Browsing products, reading recommendations, checking reviews — those are massive reads, and nobody complains if they're a little slow or a little stale. But **decrementing stock, placing orders, taking payment** — get any of *those* writes wrong and you've undersold revenue, oversold goods, or double-charged a customer; that means paying out money and getting sued. The core proposition of the whole architecture is exactly this: **treat these two kinds of data with completely different levels of rigor** — count every cent for money and stock, and cut browsing and recommendations all the slack in the world.

## 2. The essence of the business: what problem is it solving

What the platform does, put plainly, is one sentence: **let buyers hand over their money with confidence, let sellers ship the right goods accurately, and don't let a single cent or a single item go wrong in between.**

Its business reality brings three iron laws:

- **Money and goods must square, and must reconcile**: a user who paid must get goods and an order; a platform that took the money must be able to reconcile it against the payment provider. There's no room here for "eventually, roughly consistent."
- **Stock is finite and gets fought over**: the last item in stock cannot be sold to two people at once (overselling), or you get guaranteed complaints and payouts.
- **Traffic is wildly uneven; mega-sales are the norm**: calm most of the time, but the instant a mega-sale or flash sale hits, momentary traffic can spike tens or hundreds of times — the system either holds, or crashes and makes the news.

**Key fact: in e-commerce, a "bad read" earns a couple of curses from the user; a "bad write" costs the platform cold, hard cash.** This one fact dictates nearly every architectural trade-off downstream — above all, "**why you can't wrap the entire checkout flow in one big transaction.**"

## 3. Core requirements and constraints

Split the requirements into two kinds. **Telling "function" apart from "quality" is an architect's first fundamental skill.**

**Functional requirements (what the system must be able to do):**
- [ ] Browse and search: roam the product catalog, search keywords, view recommendations and reviews.
- [ ] Shopping cart: add to cart, change quantity, check out.
- [ ] Place order: validate, decrement stock, generate an order.
- [ ] Payment: initiate payment, confirm receipt, roll back on failure.
- [ ] Inventory management: decrement, restock, prevent overselling.
- [ ] Fulfillment / logistics: ship, track, deliver.
- [ ] Pricing and promotions: pricing, coupons, spend-and-save, flash-sale prices.
- [ ] Users and accounts: login, addresses, order history.

**Non-functional requirements / quality attributes (this is where architecture really fights):**
| Quality attribute | Target | Why it matters for this kind of system |
|---|---|---|
| **Strong consistency for money** | 100% correct, reconcilable | A penny miscounted is an incident; payments and accounting must withstand line-by-line audit. |
| **Inventory correctness** | Never oversell | Selling what you can't ship means guaranteed complaints and payouts. |
| **Browse latency** | Product/search pages return fast | Browse clunkily and the user closes the tab and leaves; conversion drops. |
| **Peak throughput** | Withstand a mega-sale flood tens of times the norm | Crash on the big day and it's direct lost sales and a brand incident. |
| **Availability** | High availability on the core transaction path | If checkout and payment go down, you're effectively closed for business. |

**Key constraints (boundaries you cannot cross):**
- 🔴 **Different data demands different strengths of consistency.** This is the number-one design principle: money and stock need strong consistency; browsing and recommendations can be eventually consistent. Lump them together and you'll either be slow to death or wrong.
- 🔴 **The "last item" of stock is a natural point of contention.** A hot product's single stock row gets fought over by countless requests at once — an unavoidable physical bottleneck.
- 🔴 **The mega-sale flood is both unpredictable and extreme.** The system must be able to shed peaks, throttle, and degrade — not brute-force its way to a crash.
- 🔴 **Every money-touching operation must be idempotent.** Networks retry, users hammer buttons — and none of that can be allowed to cause a duplicate order or a duplicate charge.

## 4. The big-picture architecture

```
                                   User (Web / App)
                                        │
                          ┌─────────────▼─────────────┐
                          │  Edge layer: Gateway / CDN │  auth, throttle, peak-shave, degrade
                          └─────────────┬─────────────┘
                                        │
         ┌─────────── Read path (read-heavy, cache-heavy) ┴ Write path (write-light, strong consistency) ──┐
         │                                                                          │
         ▼                                                                          ▼
┌────────────────┐  ┌──────────┐  ┌──────────┐                          ┌────────────────────┐
│ Catalog service│  │ Search   │  │ Recommend│                          │   Shopping cart svc │
│ (massive reads,│  │ (dedi-   │  │ (eventu- │                          └─────────┬──────────┘
│  cached)       │  │  cated   │  │  ally    │                                    │ checkout
└───────┬────────┘  │  engine) │  │  consis.)│                                    ▼
        │ multi-tier└──────────┘  └──────────┘             ┌──────────────────────────────┐
        ▼  cache                                           │ Checkout orchestration (Saga  │
┌────────────────┐                                         │ coordinator): step forward,   │
│ In-memory KV   │  ◀── cache warm-up                      │ compensate per step on failure│
│ cache          │                                         └───┬────────┬─────────┬────────┘
└────────────────┘                                             │        │         │
                          ┌──── Peak-shaving: async order queue ┘        │         │
                          │                                              ▼         ▼
                          ▼                                     ┌──────────┐  ┌──────────┐
                  ┌──────────────┐      ┌──────────────┐        │ Order svc│  │ Payment  │
                  │ Inventory svc│      │ Price/promo  │        │(state    │  │ svc      │
                  │ no-oversell/ │      └──────────────┘        │ machine) │  │(strongly │
                  │ segmented    │                             └────┬─────┘  │ consis.) │
                  │ stock        │                                  │        └────┬─────┘
                  └──────────────┘                                  ▼             ▼
                                                    ┌──────────────────┐  ┌──────────────┐
                                                    │ Fulfillment /    │  │ Reconciliation│
                                                    │ logistics        │  │ system       │
                                                    │ (ship, track)    │  │ (line-by-line)│
                                                    └──────────────────┘  └──────────────┘
```

> The soul of this diagram is the **write path** on the right: checkout orchestration → inventory → order → payment. However big the browse/search/recommend side on the left gets, it's all fundamentally "reads" — solvable by throwing machines at a cache and scaling out. What's genuinely hard, what genuinely can't go wrong, is this transaction chain that has to be **both correct and flood-proof.**

## 5. Component responsibilities

Going through each key piece in the diagram: **what it does + why you need it** (a piece with no "why" is over-engineering).

- **Edge layer (Gateway / CDN)**: auth, throttling, pushing static assets and images to the edge; during a mega-sale it's the first gate for **peak-shaving and degradation**. *Why you need it*: the flood must be shaped and intercepted at the outermost layer — don't let it slam straight into core services.
- **Catalog service**: manages product info, **read-heavy, write-light**, holds up under load via multi-tier caching. *Why you need it*: browsing products is the highest-traffic entry point; you must put a cache in front of the database to shield it.
- **Search**: supports keywords, filtering, and sorting with a dedicated search system. *Why you need it*: product search is "find by content," which a general-purpose database does poorly — it needs a dedicated inverted-index capability.
- **Recommendations**: computes "what you might want to buy" from behavior; can be **eventually consistent and a little stale**. *Why you need it*: it lifts conversion, but it's fundamentally "icing on the cake" — a bit slow or a bit stale doesn't affect money or goods.
- **Shopping cart service**: holds what the user intends to buy until checkout. *Why you need it*: there's a time gap between adding to cart and ordering, which needs a lightweight, fast read/write staging area.
- **Checkout orchestration (Saga coordinator)**: drives the chain "decrement stock → create order → call payment → confirm/roll back" **forward step by step, and on any failed step compensates by rolling back in reverse order**. *Why you need it*: checkout spans multiple services, yet you can't lock all of them in one big transaction (too heavy, can't take the load) — so you use a Saga to break it into a long flow that "advances step by step and can roll back step by step."
- **Inventory service**: decrements and restocks, **holding the line against overselling**; for hot products it defuses contention with segmented stock or queuing. *Why you need it*: inventory is the physical precondition for "money for goods"; overselling equals a payout, full stop.
- **Order service**: maintains the order's **state machine** (pending payment → paid → shipped → completed / canceled / refunded), guaranteeing states only transition legally. *Why you need it*: the order is the transaction's "record of fact"; every state change must be traceable and must not jump around illegally.
- **Payment service**: interfaces with the money, **strongly consistent + idempotent**, ensuring "the money is charged exactly once, and reconciles against the order." *Why you need it*: this is the link that can least afford to be wrong on the entire chain — any duplicate or loss is a financial loss.
- **Price / promotions**: computes the final transaction price (list price, coupons, spend-and-save, flash-sale price). *Why you need it*: price touches money, and promo rules change often, so it needs to be computed centrally and auditably.
- **Fulfillment / logistics**: arranges shipping once the order is confirmed, and provides tracking. *Why you need it*: it turns "an online transaction" into "goods arriving offline," and this part is naturally asynchronous.
- **Reconciliation system**: cross-checks orders, payments, and accounting line by line, finding and repairing inconsistencies. *Why you need it*: since the transaction chain uses "eventual consistency" (Saga), you must have a backstop mechanism to **guarantee it truly does become consistent in the end** — reconciliation is the last line of defense for the money.

## 6. Key data flows

Picking the 3 scenarios that best capture what makes this system distinctive.

**Scenario 1: Browse to add-to-cart (the read-heavy, cache-heavy path)**
```
1. User roams home/category ──▶ gateway ──▶ catalog service
2. Check [in-memory KV cache] first ── hit ──▶ return directly (the vast majority of requests go here)
                              └ miss ─▶ fall back to DB ─▶ write result into cache ─▶ return
3. Search keyword ──▶ dedicated search engine (inverted index) returns results
4. "Picks for you" on the page ──▶ recommendation service (data can be slightly stale; eventual consistency is fine)
5. User taps "Add to cart" ──▶ shopping cart service stages it
```
> Note: this entire path **barely touches a write on the core database** — it leans entirely on caches and dedicated engines to hold up. The more people browse, the more valuable the cache. This is "reads can be loose" in action: a few seconds of staleness or a slightly-off recommendation doesn't matter.

**Scenario 2: Placing an order (the write path — Saga advances step by step, rolls back on failure)**
```
User taps "Submit order" ──▶ Checkout orchestration (Saga coordinator) starts:

  Step ① Decrement stock ──▶ inventory service: reservation succeed?
        ├─ fail (out of stock) ──▶ end immediately, tell user "sold out"
        └─ success ─▶ continue
  Step ② Create order ──▶ order service: generate order, state = "pending payment"
        └─ fail ─▶ compensate: restock what step ① reserved ──▶ end
  Step ③ Call payment ──▶ payment service: initiate charge (idempotent, carries unique order ID)
        ├─ success ─▶ order state → "paid", reservation converts to a formal decrement ──▶ trigger fulfillment
        └─ fail/timeout ─▶ compensate: order → "canceled", restock ──▶ end

  ★ Every step can be retried independently; on failure, compensate in [reverse order]. No single big transaction
    spans all services at any point.
```
> Architecture point: **replace "one big distributed transaction" with "a compensable long flow (Saga)."** A big transaction would lock stock, order, and payment for a long time, dragging the whole system down under high concurrency; the Saga breaks it into "fast-in, fast-out per step, roll back if wrong," trading **eventual consistency + reconciliation** for throughput.

**Scenario 3: Mega-sale flash sale (peak-shaving + queuing + throttling)**
```
A million people grab for 1,000 units ──▶ gateway [throttle]: most requests are blocked/ticketed right at the door
        │ the requests let through
        ▼
  Async order queue (peak-shaving) ──▶ flatten the momentary flood into a steady rate the system can digest
        │ dequeued one by one
        ▼
  Inventory service: do [segmented stock / queued decrement] for this hot product, scattering single-row contention
        │ the ones who got it
        ▼
  Proceed through the normal Saga checkout flow; those who didn't ──▶ immediately return "sold out" (fail fast, don't make them wait)
```
> Architecture point: **don't brute-force the flood — "shape" it.** Throttle at the door, the queue flattens it in the middle, segmented stock dissipates heat at the bottom, and degradation drops the cargo to save the ship right before a crash — the four-piece combo turns "a peak tens of times the norm" into "a stream the system can calmly digest."

## 7. Data model and storage choices

Core entities: `Product` ─ `Inventory (per SKU)`; `User` ─ `Cart` ─ `Order` ─ `Order item`; `Order` ─ `Payment record`; `Promotion/price rules`. Among them, the **order** carries a state machine, and **payment records** plus **accounting ledger** are the basis for financial reconciliation.

| Data | Storage type | Why |
|---|---|---|
| Orders / payments / accounting ledger | Relational (strong transactions) | Touches money; needs strong consistency, transactions, line-by-line reconciliation |
| Inventory (decrementable counter) | Strongly consistent store with atomic decrement | Anti-oversell relies on atomic ops; hot items get segmented on top |
| Catalog (massive reads) | Relational base + in-memory KV cache | Read-heavy, write-light; the cache absorbs the vast majority of reads |
| Search index | Dedicated search system (inverted index) | "Find by content" is something a general-purpose database does poorly |
| Cart | In-memory KV cache (can be persisted) | Frequent read/write, simple structure, low consistency demands |
| Recommendation results | Precomputed + cached | Can be eventually consistent and slightly stale; shouldn't hit the DB in real time |
| Product images | Object storage + CDN | Large, immutable, fetched by reference; push to the edge to speed up |
| Mega-sale peak-shaving | Async message queue | Flatten the momentary flood into a digestible steady rate |

> Teaching point (the most important in this section): **consistency is not a "global switch" — it's a "dial graded per data type."** Turn the dial to "strong consistency + reconcilable" for money, to "anti-oversell" for inventory, while browsing, recommendations, and reviews can happily turn down to "eventually consistent." **Demanding one single strength from all your data means either dying slow or going wrong — and that is exactly the essence of e-commerce architecture.**

## 8. Key architectural decisions and trade-offs ⭐

**(The most valuable section of this template.)** Almost every fork in the e-commerce road answers the same question: **"how strong should this data's consistency be, and how do we keep it from going wrong under the flood."**

**Decision 1: How do you decrement stock without overselling?**
- **Pessimistic locking** (lock the row before decrementing): never oversells, but under high concurrency everyone queues for the same lock — **a hot product becomes a bottleneck the instant it's locked**, and throughput collapses.
- **Optimistic locking** (decrement with a version number, retry on conflict): lock-free, high throughput, but a hot product has an extremely high conflict rate, so **huge numbers of requests retry over and over**, dragging things down just the same.
- **Reserve + async confirm** (quickly pre-claim first, convert to a formal decrement on payment success, restock on payment timeout): fast response, can take the load, at the cost of managing the fiddly "reservation-timeout-restock" logic.
- **Segmented stock** (split 1,000 units into 10 segments of 100, scattering requests across different segments to decrement): turns "single-row contention" into "multi-row parallelism," hugely relieving hotspots. The cost is potential imbalance across segments (some empty out while others have stock left), requiring rebalancing.
- **Leaning**: **optimistic locking / atomic decrement is plenty for ordinary products; for hot / flash-sale products, deploy the "reserve + segment + queue" combo.** The core idea is to **turn one frantically-fought-over point into many points that can run in parallel.** The cost is complexity and the operational overhead of "reservation restock" — but this is the only way to keep hotspots from overselling.

**Decision 2: Consistency of orders and payments — distributed transaction, or Saga + reconciliation?**
- **Distributed transaction (the two-phase-commit family)**: cleanest semantics — all succeed or all fail. But it **locks resources across multiple services for a long time**; under high concurrency throughput is abysmal, and one stuck participant jams everything. **E-commerce can't afford it.**
- **Saga + eventual consistency + reconciliation**: break checkout into a chain of compensable steps "decrement stock → create order → pay," each fast-in/fast-out, rolling back in reverse order on failure, with a reconciliation system as the backstop guaranteeing eventual consistency.
- **Leaning**: **almost always pick Saga + reconciliation.** Trade "eventual consistency + a reliable reconciliation system" for high throughput. The cost is a brief "inconsistency window" in the middle (order created but not yet paid), plus the obligation to **build out a reconciliation system** as the money's last line of defense — and that investment is not one to skimp on.

**Decision 3: Idempotency design — how do you prevent duplicate orders / duplicate payments?**
- No idempotency: one network retry, one jittery double-tap from the user, and you **create a duplicate order, charge twice** — direct financial loss.
- With idempotency: every checkout/payment request carries a **unique identifier (idempotency key)**; when the server sees a duplicate of the same key, it just returns last time's result instead of executing again.
- **Leaning**: **every write operation that touches money or stock must be idempotent — no exceptions.** Because retries are the norm in distributed systems (a timeout leaves you unsure whether it succeeded, so you can only resend). The cost is maintaining storage and dedup logic for idempotency keys — but against the risk of financial loss, that cost is negligible.

**Decision 4: Read/write separation — should the massive browse reads and the critical transaction writes be split apart?**
- Don't split: reads and writes pile onto the same database, and **the massive browse reads contend for resources with the critical transaction writes**, dragging each other down.
- Split: hold up reads (catalog, search, recommendations) with caches and read replicas, and leave writes (inventory, order, payment) to the strongly-consistent primary store.
- **Leaning**: **must split.** Reads and writes are "two different species" in e-commerce — reads want fast and cheap, writes want accurate and consistent. **Hand the read path to caches and dedicated engines, and let the primary store devote itself to the writes that can't be wrong.** The cost is that the data you read may lag slightly (eventual consistency) — but the browse scenario tolerates that completely.

**Decision 5: How do you withstand the mega-sale flood?**
- Brute force (mindlessly add machines): extremely costly, and adding machines has a ceiling — stateful pieces like the database can't scale infinitely, so **come crunch time it still crashes.**
- Shape it (peak-shaving + throttling + degradation + cache warm-up): flatten the peak with a queue, block at the door with throttling, drop non-core features in a crisis via degradation, and warm hot data into the cache in advance.
- **Leaning**: **"shape" the flood, don't "brute-force" it.** Before the sale, **warm the cache** (preload bestseller data into the cache); at the entrance, **throttle** (queue or fail fast when over capacity); on the core chain, use an **async queue to shave peaks**; in a crisis, **degrade** (temporarily switch off non-core features like recommendations/reviews to protect checkout and payment). The cost is a degraded experience (queuing, temporarily shrunken features) — but that's the wise trade of "protect the core transaction" over "crash everything."

**Decision 6: Order state — manage it with a state machine, or change fields scattered all over?**
- Scattered changes: change the order state wherever it's convenient. Simple, but **states jump around** (e.g., "refunded" jumping back to "shipped"), hard to trace and bug-prone.
- State machine: explicitly define "which states exist, which transitions are allowed," and reject any illegal jump outright.
- **Leaning**: **the order must use a state machine.** It's the transaction's record of fact; every state change must be legal and traceable. The cost is having to design the states and transition rules clearly up front — but what you get in return is control and auditability across the whole transaction lifecycle.

## 9. Scaling and bottlenecks

E-commerce bottlenecks follow a clear pattern: **first you jam on "hot writes," then on "the flood's volume," and finally on "the cost of reads."**

- **First bottleneck: single-row stock contention on hot products.** A bestseller's "last batch" gets fought over by countless requests on the same row, and lock contention beats throughput into the floor.
  Fixes: ① segmented stock — split one row into many segments decremented in parallel; ② queuing — let requests fighting for the same product be processed in queued order; ③ reserve + async confirm — shorten how long each request holds the resource; ④ hotspot detection + dedicated scaling.
- **Second bottleneck: the mega-sale flood.** Momentary traffic tens of times the norm; synchronous checkout will blow the chain wide open.
  Fixes: ① async checkout (requests enqueue first, quickly telling the user "queued / accepted"); ② queue peak-shaving to flatten the peak; ③ entrance throttling — fail fast over capacity instead of waiting around; ④ degradation — cut non-core features in a crisis to protect the core; ⑤ cache warm-up — don't let a cold cache fall back to the database and crush it at peak.
- **Third bottleneck: massive reads on catalog and search.** Browse traffic is the largest; hitting the database directly is a guaranteed crash.
  Fixes: ① multi-tier caching (edge/CDN + in-memory KV) to block the vast majority of reads before the database; ② hand search to a dedicated search system, don't brute-search with a general-purpose database; ③ read/write separation, with read replicas taking the reads.
- **Fourth bottleneck: the reconciliation pressure brought by the consistency window.** Once you use Saga / eventual consistency, you'll continuously produce a small stream of "order, payment, and accounting don't match" cases.
  Fix: build a **reconciliation system** — cross-check line by line, auto-repair what's fixable, alert for human handling on what isn't. This is the normalized infrastructure for financial safety once you're at scale.

## 10. Security and compliance essentials

- 🔴 **Payment security is priority one.** The money chain must be strongly consistent, auditable, and reconcilable line by line; handle sensitive payment data per compliance requirements, and never put a core charging decision on the client.
- **Idempotency is security**: a duplicate request must not cause a duplicate charge or duplicate shipment — the idempotency key discussed earlier is both a correctness measure and a security measure against financial loss.
- **Price-tampering protection**: the transaction price **must be recomputed by the server**; never trust a price sent from the client — otherwise someone tampers with the request and orders for a penny.
- **Anti-fraud / anti-scalping**: mega-sale flash sales attract scripts and scalpers, so you need risk control (purchase limits, human verification, behavioral detection) to intercept abnormal traffic at the entrance — protecting both fairness and your stock and system.
- **The server doesn't trust the client**: stock, price, promotion eligibility, ordering permissions — all are decided authoritatively by the server. Everything sent by the client can be tampered with.
- **Overselling is an incident**: inventory atomicity and anti-overselling are not just a correctness problem but a compliance-and-reputation risk of "sold it, can't ship it."

## 11. Common pitfalls / anti-patterns

- ❌ **Wrapping the whole chain "decrement stock → create order → pay → ship" in one big transaction** → ✅ break it into a compensable Saga long flow, each step fast-in/fast-out, rolling back in reverse order on failure.
- ❌ **Calling payment synchronously at checkout and blocking while the payment provider answers** → ✅ make payment async and idempotent; don't let one external call jam the entire chain.
- ❌ **Skipping idempotency, so a network retry / user double-tap causes a duplicate order or duplicate charge** → ✅ every money/goods write carries an idempotency key for dedup.
- ❌ **Decrementing stock directly with `UPDATE stock = stock - 1`** → ✅ use atomic decrement + anti-oversell; segment/queue hot products so a single row doesn't become a lock-contention bottleneck.
- ❌ **Demanding one single strength of strong consistency from all data (requiring even recommendations and reviews to be real-time consistent)** → ✅ grade by data: strong consistency for money, anti-oversell for stock, eventual consistency for browsing/recommendations.
- ❌ **Brute-forcing the mega-sale by mindlessly adding machines** → ✅ peak-shaving + throttling + degradation + cache warm-up — "shape" the flood instead of "catching it head-on."
- ❌ **Trusting the price/stock/promotion sent from the client** → ✅ have the server recompute all amounts and eligibility.
- ❌ **Using eventual consistency but building no reconciliation** → ✅ eventual consistency must be paired with a reconciliation backstop, or "eventually" may never arrive.

## 12. Evolution path: MVP → growth → maturity

Architecture grows up. **Don't force the maturity-stage diagram onto an MVP.**

| Stage | User/scale magnitude | What the architecture looks like | What to worry about now |
|---|---|---|---|
| **MVP** | Validate the idea | **Monolithic store**: catalog, cart, order, payment all in one app, one database, checkout maybe done in a single transaction | First validate "whether this business even works"; don't split microservices and roll out Saga from day one |
| **Growth** | Tens of thousands ~ millions of orders | **Carve out the core domains** (inventory / order / payment standalone), introduce caching to block reads, a dedicated search system, read/write separation; switch checkout to Saga + idempotency | Find bottlenecks, protect transaction correctness, and hold up the read path with caching |
| **Maturity** | Tens of millions and up / has mega-sales | **Mega-sale special ops**: segmented stock, async checkout + peak-shaving queue, throttling and degradation, cache warm-up; build a reconciliation system; multi-active disaster recovery on key chains | Anti-oversell, peak-shaving, financial reconciliation, disaster recovery, cross-team coordination |

## 13. Reusable takeaways

- 💡 **Consistency is "a dial graded per data type," not "a global switch."** First ask each piece of data "what happens if it's wrong / stale," then decide how strong its consistency should be. This idea applies to any system where "read and write demands differ wildly."
- 💡 **Don't string a long flow together with one big transaction — use "a compensable long flow + reconciliation."** When you span multiple services and also take high concurrency, eventual consistency + a backstop check is almost always more realistic than one big distributed transaction.
- 💡 **Turn "one frantically-fought-over point" into "many points that can run in parallel."** The idea behind segmented stock generalizes to any hotspot-contention scenario — sharding, bucketing, queuing are all fundamentally "heat-sinking the hotspot."
- 💡 **"Shape" the flood, don't "brute-force" it.** Throttling, queuing, peak-shaving, and degradation are a general "traffic-shaping" toolkit, useful in any system that hits peaks.
- 💡 **Every write that can be retried must be idempotent.** In the distributed world retries are the norm; idempotency is the universal discipline that ensures "retries don't cause trouble" — especially anywhere money is involved.

## 🎯 Quick quiz

<Quiz
  question="Under high concurrency, what's the most critical means of preventing overselling?"
  :options="['Throw a few more servers at it', 'Decrement stock atomically, not via «read remaining - subtract - write back»', 'Put the stock count in a cache']"
  :answer="1"
  explanation="A non-atomic «read-modify-write» will inevitably oversell under high concurrency; only an atomic operation guarantees the last item is successfully decremented by exactly one request."
/>

---

## References & Further Reading

> This template is compiled from the following **authoritative pattern libraries** and **official engineering docs**.

**📖 Patterns / engineering docs:**
- [Pattern: Saga (microservices.io)](https://microservices.io/patterns/data/saga.html) — maintain cross-service data consistency with a sequence of local transactions + compensating transactions.
- [Stripe: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency) — idempotency keys to prevent duplicate charges in order / payment.
- [Stripe: Idempotent requests (API docs)](https://docs.stripe.com/api/idempotent_requests) — the precise semantics of idempotency keys, mapping to dedup for orders / inventory.

---

> 📌 Remember an e-commerce platform in one line: **it isn't "a website with lots of products" — it's "a transaction machine where money and goods can't go wrong, and which also has to withstand the flood." Every architectural trade-off ultimately answers "how seriously must this data be taken, and will it crash on mega-sale day."**
