# Payment System · Architecture Template

> **Representative products**: Stripe, Alipay, WeChat Pay, PayPal, all kinds of "checkout / aggregated payment" platforms
> **One-line definition**: in an unreliable network and an untrustworthy world, move "money from A to B" down to the exact cent — never double, never drop, every entry reconcilable and traceable.

---

## 1. One-line definition

A payment system = **a money-state machine where "correctness trumps everything"** + **a ledger that always balances.**

Its biggest difference from the systems you know: **everything else chases speed; this one chases "correct" first.** It would rather be slow, rather reject a transaction, than miscalculate a single cent or charge twice. Its soul isn't performance — it's three words: **idempotency, consistency, reconcilability.**

## 2. The business essence: what problem is it solving

A payment system is the **trusted intermediary for the flow of money**: it stands between users, merchants, and banks / card networks, letting a sum of money travel safely and deterministically from payer to payee.

What it sells isn't technology — it's **trust**. Users dare to hand it their card number; merchants dare to wait for it to settle. One miscalculated ledger, one double charge, and that trust collapses.

Where the money comes from: a fee on every transaction, cross-border / currency-conversion fees, value-added services (installments, fraud control, reconciliation reports).

> **The key fact: money can't be created out of thin air, and can't vanish into thin air.** This physics-like conservation law dictates nearly every architectural trade-off in a payment system — it can't, like an ordinary system, "just lose it and recompute."

## 3. Core requirements and constraints

**Functional requirements:**
- [ ] Initiate payment (multiple channels: bank card / balance / third-party wallet)
- [ ] Refund / partial refund
- [ ] Accounts and balances (who has how much money)
- [ ] Reconciliation (cross-check every entry against the bank / channel)
- [ ] Clearing and settlement (actually move the money to the merchant)

**Non-functional requirements / quality attributes (worlds apart from an ordinary system here):**
| Quality attribute | Target | Why it matters for this kind of system |
|---|---|---|
| **Correctness** | Exact to the cent | This is the floor of all floors, above everything |
| **Idempotency** | Duplicate requests never double-charge | The network retries; without idempotency you overcharge |
| **Consistency** | Strong consistency of funds | Balances and ledgers can never show money out of nowhere |
| **Auditability** | Every entry traceable | Regulatory requirement, dispute evidence, after-the-fact audits |
| **Availability** | 99.99%+ | But **correctness comes before availability**: when in doubt, rather reject |

**Key constraints (boundaries you cannot cross):**
- 🔴 **Conservation of funds**: at any instant the books must balance; no intermediate state may make money "disappear."
- 🔴 **External channels are unreliable and asynchronous**: a bank may time out, or return a result hours later — "**unknown state**" is the norm, not the exception.
- 🔴 **Compliance is a hard constraint**: PCI-DSS (card data), anti-money-laundering, regulatory reporting — not a "deal with it later."
- 🔴 **Timeout ≠ failure**: when a request times out, it may have succeeded, or may have failed — this is the hardest part of payments.

## 4. The big picture

```
   User / Merchant
       │ initiate payment
       ▼
┌──────────────────┐
│ Payment gateway  │  onboarding, signature check, tokenization (card number never lands)
│ / checkout       │
└────────┬─────────┘
         ▼
┌──────────────────────────────────────────────────────────────┐
│  Payment orchestration (state-machine engine) — the core     │
│  • Create payment order (dedupe by idempotency key)          │
│  • Drive state: pending → processing → success/fail/unknown  │
│  • Route: which channel to use      • Fraud check            │
└───┬───────────────┬───────────────┬───────────────┬──────────┘
    ▼               ▼               ▼               ▼
┌─────────┐  ┌──────────────┐  ┌───────────┐  ┌──────────────────┐
│ Fraud   │  │ Channel      │  │ Ledger    │  │ Async notify /   │
│ control │  │ adapter      │  │ (double-  │  │ callback handler │
│         │  │ (bank/wallet)│  │ entry)    │  │ (trust queries)  │
└─────────┘  └──────┬───────┘  └─────┬─────┘  └──────────────────┘
                    │ async, may time out  │
                    ▼                  ▼
            ┌───────────┐    ┌────────────────┐
            │ External  │    │ Reconciliation │ daily line-by-line check
            │ bank /    │───▶│ system         │ against the channel; diffs
            │ card net  │    │ (final truth)  │ handled auto / manually
            └───────────┘    └────────────────┘
```

> The soul of the system is the **Ledger + the Reconciliation system**: the former uses "double-entry bookkeeping" to keep the books balanced at every instant; the latter uses "line-by-line cross-checking with the bank" to backstop everything asynchronous and unknown. **These two things exist so that "money" always adds up.**

## 5. Component responsibilities

- **Payment gateway / checkout**: onboard each client, verify signatures, **tokenize** the sensitive card number (use a token in place of the real number, so the number never enters the internal system). *Why it's needed*: keep compliance risk and sensitive data out at the very edge.
- **Payment orchestration / state machine**: the brain of the whole payment. It uses an **idempotency key** to dedupe requests, drives the payment order through its states, and routes the channel. *Why it's needed*: a payment is, at its core, a state machine with well-defined states that may not jump around arbitrarily.
- **Channel adapter**: translate the internal unified command into each bank's / wallet's protocol. *Why it's needed*: shield internal logic from external differences; external calls are asynchronous by nature and may time out.
- **Ledger**: record money movement with **double-entry bookkeeping** (every transaction records a debit and a credit at once, the two sides equal). *Why it's needed*: use "structure" to guarantee conservation of funds, auditability, and tamper-resistance (see Decision 2).
- **Fraud control / anti-fraud**: judge in real time whether a transaction is suspicious. *Why it's needed*: payments are a hotbed of fraud.
- **Reconciliation system**: every day, cross-check your own transaction log line by line against the bank's / channel's, and surface the diffs. *Why it's needed*: this is the ultimate backstop for "unknown state" — settle on the facts both sides agree on.
- **Async notification handler**: receive channel callbacks, but **never blindly trust a callback** — settle on the result of your own active query.

## 6. Key data flows

**Scenario 1: a card payment (the core path)**
```
1. User submits payment ──▶ gateway: signature check, tokenize the card number
2. ──▶ orchestration layer: dedupe by "idempotency key"
        already exists ──▶ return the previous result directly (don't re-initiate!)
        doesn't exist ──▶ create payment order (state = processing)
3. ──▶ fraud control: allow / block
4. ──▶ channel adapter ──▶ external bank (async, may time out)
5. Bank returns success ──▶ ledger records a pair of entries (debit: user, credit: merchant), state = success
6. ──▶ notify the merchant asynchronously; surrounding systems (points / shipping) subscribe to the event, eventually consistent
```

**Scenario 2: what if it times out? (the hardest payment scenario)**
```
At step 4 the bank takes forever / times out:
  ✗ Wrong approach: immediately mark it failed ──▶ the user may have been charged, yet you record failure → money vanishes
  ✓ Right approach: set the state to "unknown," kick off [query compensation]:
       after a while, actively ask the bank "did this one succeed or not?"
       still uncertain ──▶ leave it to that day's [reconciliation] as a backstop, settling on the bank's final log
```
> This is exactly why, in payments, **"unknown" must be a first-class state** — it can't be crudely lumped in with "failure."

## 7. Data model and storage choices

The core entities: `payment order (with a state machine)`; `ledger entries (debit/credit in pairs)`; `channel transaction log`; `reconciliation diffs`.

| Data | Storage type | Why |
|---|---|---|
| Payment orders / accounts | Relational (strong transactions) | Money operations need ACID, strong consistency |
| Ledger entries | Append-only / immutable log | Insert-only, never update or delete — guarantees auditability and tamper-resistance |
| Channel logs, reconciliation | Relational / columnar | Massive volume, aggregated and cross-checked by day |
| Idempotency keys | KV (with a unique constraint) | High-speed dedupe, prevents double charges |

> Teaching point: the ledger **only ever appends, and never updates or deletes** (a balance is *computed* by "summing all the entries," not stored as a single number that gets overwritten). This makes history tamper-proof, recomputable at any time, and auditable.

## 8. Key architectural decisions and trade-offs ⭐

**Decision 1: how do you guarantee idempotency? (the lifeline of payments) ⭐**
- No idempotency: on a network retry, the same payment gets initiated twice → **double charge**, a disaster.
- With idempotency: every request carries a unique "idempotency key," the server uses a unique constraint to guarantee "the same key is processed only once," and a duplicate request just returns the first result.
- **The lean**: **mandatory, no exceptions.** Idempotency is the seatbelt for any "write operation that may be retried."

**Decision 2: how do you record account balances? "Update a balance field" or "double-entry bookkeeping"? ⭐**
- Update a balance field (`UPDATE balance = balance - 100`): intuitive, but under concurrency it loses updates easily, can't be audited, can't be reconciled, and is hard to trace when something goes wrong.
- Double-entry bookkeeping (every transaction records paired debit/credit entries, the balance derived by summing the entries): **conserves funds by nature, is auditable, tamper-proof, and can precisely reconstruct any past instant.**
- **The lean**: a serious payment system always uses double-entry. The cost is a more complex model and more writes. **This is the model case of "letting the data structure itself guarantee correctness."**

**Decision 3: how do you handle timeout / failure?**
- Treat a timeout as failure: simple, but the user may already be charged → money vanishes, a serious incident.
- Introduce an "unknown" state + query compensation + reconciliation backstop: complex, but **you don't lose money.**
- **The lean**: you must distinguish "failure" from "unknown," and converge "unknown" to certainty via active querying and reconciliation.

**Decision 4: strong consistency at the core, eventual consistency at the periphery.**
- Strong consistency end to end: impossible, and unnecessary (and it would crush performance).
- **The lean**: **the money core (charging, bookkeeping) is strongly consistent**; **the periphery (sending an SMS, adding points, shipping notifications) is event-driven, eventually consistent.** Separating "what can't be wrong" from "what's fine to be a bit late" is the key judgment.

## 9. Scaling and bottlenecks

- **First bottleneck: write contention on hot accounts** (a big platform merchant's receiving account is hit by countless concurrent ledger writes). → Fix: serialize / queue at the account dimension, batch-merge ledger writes, split into sub-accounts.
- **Second bottleneck: reconciliation data explodes with transaction volume.** → Fix: sharding (database/table split) + offline batch processing + incremental reconciliation.
- **Third bottleneck: external channels rate-limit / jitter.** → Fix: multi-channel routing + degradation, auto-failover when one channel goes down.
- **The money core can't be naively sharded**: a cross-shard transfer drags in distributed transactions, so shard along a natural boundary like "account" and avoid cross-shard money operations.

## 10. Security and compliance highlights

- 🔴 **Card-data compliance (PCI-DSS)**: the real card number **never lands** — use tokenization / a third-party collector; the internal system only ever sees a token.
- **Anti-replay and signature verification**: every external request / callback must be signature-checked to prevent forgery.
- **Callbacks aren't trustworthy**: forging a "payment succeeded" callback is a common attack → **always settle on the result of your own active query to the bank.**
- **Fraud control / anti-fraud**: real-time detection and blocking of stolen cards, money laundering, and cash-out schemes.
- **Audit logs are undeletable**: who, when, changed what — keep a full trail to satisfy regulators.
- **Least privilege**: lock down the permissions on any interface that can move money to the tightest possible, and require multi-party approval for critical operations.

## 11. Common pitfalls / anti-patterns

- ❌ **Updating an account by "read balance - modify balance - write back"** → ✅ Double-entry bookkeeping + immutable entries; eliminate lost updates, stay auditable.
- ❌ **No idempotency, so a retry causes a double charge** → ✅ Idempotency key + unique constraint, the floor.
- ❌ **Treating an external callback as a trusted source** → ✅ Signature check + settle on your own query.
- ❌ **Marking a timeout as failure outright** → ✅ Introduce an "unknown" state, query compensation + reconciliation backstop.
- ❌ **Storing / logging sensitive card numbers in plaintext** → ✅ Tokenize; the card number never lands.
- ❌ **Chasing performance at the expense of correctness** → ✅ In payments correctness is absolutely first; being a bit slow, or rejecting one transaction, beats getting it wrong.

## 12. Evolution path: MVP → Growth → Maturity

| Stage | Scale | What the architecture looks like | What to worry about now |
|---|---|---|---|
| **MVP** | Just starting | **Plug in a mature third-party payment provider** (you never touch funds, never store cards), with a thin wrapper layer | Outsource compliance and security to the experts; just get the business running |
| **Growth** | Multi-channel / at scale | Build your own payment orchestration and state machine, multi-channel routing, your own ledger and daily reconciliation, fraud control | Idempotency, reconciliation, unknown-state handling — make "correct" rock-solid |
| **Maturity** | Platform-scale / cross-border | Clearing and settlement, multi-currency, regulatory reporting, hot-account optimization, active-active disaster recovery | Funds safety, compliance, disaster recovery, reconciliation at scale |

## 13. Reusable takeaways

- 💡 **Idempotency is the seatbelt for any "write operation that may be retried."** Not just payments — for any distributed write, ask: "would running it twice cause a problem?"
- 💡 **Guaranteeing correctness through the data structure itself beats guarding it with careful code.** Double-entry makes "the books always balance" a structural property, rather than relying on a programmer never slipping up.
- 💡 **Treat "unknown" as a first-class citizen.** In a distributed world timeout ≠ failure; acknowledging and actively converging uncertainty is far safer than pretending it doesn't exist.
- 💡 **Separate "what can't be wrong" from "what's fine a bit late"** — strong consistency at the core, eventual consistency at the periphery — the key judgment for getting both performance and correctness.
- 💡 **External input is never trustworthy** — verify callback signatures, and settle on your own query.

## 🎯 Quick quiz

<Quiz
  question="What is a payment system's lifeline for preventing 'a network retry causing a double charge'?"
  :options="['Locking', 'An idempotency key: the same request is processed only once', 'More logging']"
  :answer="1"
  explanation="Idempotency is the seatbelt for any 'write operation that may be retried' — a duplicate request with the same idempotency key just returns the first result, never double-charging."
/>

---

## References & Further Reading

> This template is compiled from the following **real open-source projects** and **engineering blogs**.

**🔧 Open-source prototypes (read the code directly):**
- [tigerbeetle/tigerbeetle](https://github.com/tigerbeetle/tigerbeetle) — a database built for financial transactions, with built-in accounts/transfers double-entry bookkeeping, embodying ledger consistency and high-performance OLTP.
- [juspay/hyperswitch](https://github.com/juspay/hyperswitch) — an open-source, composable payment platform (Rust): multi-channel routing / reconciliation / acquiring connectors, PCI-compliant.

**📖 Engineering blogs:**
- [Stripe: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency) — idempotency keys + client retries + exponential backoff, the foundational article on preventing double charges in payments.

---

> 📌 Remember a payment system in one line: **it's not "an interface that can charge a card," it's "a state machine that, in an uncertain world, still gets the books exactly right to the cent" — and every design choice answers "how do I make it never double, never drop, never wrong, and every entry auditable."**
