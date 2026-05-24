# Browser Extension · Architecture Template

> **Representative products**: Honey, Grammarly, all kinds of browser plugins (ad blocking, password management, web annotation)
> **One-line definition**: while a user browses "someone else's web page," inject an extra layer of capability into that page from inside the browser — and monetize that privileged position.

---

## 1. One-line definition

A browser extension = **a piece of code that "lives parasitically" inside someone else's web page** + **a background that stays resident in the browser and coordinates across tabs** + **a data and monetization engine on your own servers.**

The most counterintuitive thing architecturally: you **have no page of your own.** Your "frontend" runs on arbitrary websites the user may visit at any moment — sites you don't control (shopping sites, email, docs) — strictly bound by the extension platform's sandbox model. So the central tension of the whole architecture is: **you can see almost everything the user browses (enormous power), yet the platform, the user, and the law are all watching like hawks for "what exactly did you touch?" (enormous trust and privacy boundaries).** Whether the architecture is any good comes down to one question: "how do you make this injected layer both useful and trustworthy, with the least permission and the most restrained data collection?"

## 2. The business essence: what problem it solves

What users want is **one extra helper capability, right in the moment they're doing something, without leaving the current page:**
- Cashback/coupon (Honey): at checkout, automatically scour the web for coupons, one-click test which one saves the most, and give me cashback on the side;
- Writing assistance (Grammarly): as I type in any input field, catch errors and rewrite in real time;
- Tool enhancement: block ads, save passwords, annotate/translate web pages.

What they share: **its value happens in the instant "the user is operating inside another product."** It doesn't pull the user away to its own app; it rides alongside the user's existing workflow.

Where the money comes from (cashback being the canonical case):
- **Affiliate revenue share**: when a user clicks/orders through you, the merchant pays you a commission; you give part of it back to the user and keep part as profit;
- Subscriptions / premium features (common for writing tools: basic is free, advanced grammar, rewriting, and plagiarism checks cost money);
- Enterprise editions (team management, compliance, private dictionaries).

> **The key fact: the entire business model of cashback products rests on one technical act — "attribution." Whether you can correctly and compliantly prove "this order came from me" directly decides whether there's any revenue at all.** This one fact will repeatedly shape both the architecture and the ethical boundaries below.

## 3. Core requirements and constraints

**Functional requirements (what the system must be able to do):**
- [ ] Identify the current page: judge "is the user right now on the checkout page of some supported merchant / in some enhanceable input field?"
- [ ] Inject and read/write the page: insert UI (buttons, tooltips) into the page, read/write the page's DOM (read the price, auto-fill a coupon code).
- [ ] Cross-tab / cross-session state: login state, the order currently being tracked, pending cashback.
- [ ] Communicate with the backend: query the coupon store, query merchant rules, report attribution, sync the cashback ledger.
- [ ] Popup / standalone UI: the panel that pops up when you click the extension icon (view cashback balance, account, settings).
- [ ] Data collection pipeline: continuously collect, validate, and refresh coupons and merchant rules (crawling + crowdsourcing).

**Non-functional requirements / quality attributes (this is where architecture actually fights):**
| Quality attribute | Target | Why it matters for this kind of system |
|---|---|---|
| **Trust / provable privacy** | Highest priority | You can see almost every page the user visits; the day you're caught secretly exfiltrating data, the product dies. |
| **Zero damage to the page** | Don't slow down or mess up the host page | You're a guest; wreck someone's shopping cart and they uninstall *you*, not the store. |
| **Injection responsiveness** | Recognize the checkout page within milliseconds of its appearing | A step too slow and the user has already paid; the discount and attribution are both too late. |
| **Platform compliance** | Strictly conform to extension-platform policy | A policy violation = immediate takedown; this is the sword hanging over your head. |
| **Data freshness** | Coupons that "haven't expired and actually work" | Try five coupons and all are dead, and the user's trust in you drops to zero. |
| **Availability** | Highly available backend | If the backend goes down, the extension spins on the checkout page — i.e., you fail the user at their most critical moment. |

**Key constraints (boundaries you cannot cross):**
- 🔴 **You run inside the extension sandbox; your capabilities are defined by the platform and can be revoked at any time.** Which APIs you may use, which permissions you can get, how long your scripts live — it's all the platform's call.
- 🔴 **The platform's permission model evolves (and keeps getting stricter).** One policy upgrade (tightening background residency, tightening network-interception ability, mandating least privilege) can force you to tear down and rebuild the whole architecture — this is the **number-one platform risk.**
- 🔴 **The content script shares the same page environment as the host page, yet is deliberately isolated.** It can read the DOM, but (under the isolation mechanism) can't touch the page's own script variables; its power is greater than an ordinary web page's, yet far less than a native program's.
- 🔴 **"Being able to see everything" is itself the biggest risk surface.** The boundary of data collection isn't a technical question — it's a life-or-death one.
- Legal / ethical boundaries: once the way you obtain attribution crosses the line (overwriting attribution that someone else brought in), it triggers legal and reputational risk.

## 4. The big-picture architecture

```
   User's browser                                       │            Your servers (backend)
 ┌───────────────────────────────────────────────┐   │   ┌──────────────────────────────────┐
 │                                               │   │   │                                  │
 │  Host page (shopping site / email / docs      │   │   │   Access-layer API (auth / rate    │
 │   — not under your control)                    │   │   │   limiting)                        │
 │   ┌─────────────────────────────────────┐    │   │   └───────────────┬──────────────────┘
 │   │  Content Script                       │    │   │                   │
 │   │  • Injected into the page, reads/      │    │   │      ┌────────────┼─────────────┐
 │   │    writes the DOM                      │    │   │      ▼            ▼             ▼
 │   │  • Identify merchant / input field     │    │   │  ┌────────┐ ┌─────────┐ ┌──────────┐
 │   │  • Auto-fill coupon / inject hint UI    │    │   │  │ Coupon │ │Merchant │ │ User acct │
 │   │  • [Only does things tied to this      │    │   │  │ store /│ │rule store│ │/ cashback │
 │   │    page's DOM]                        │    │   │  │ dict.  │ │/attr.rule│ │  ledger   │
 │   └───────────────┬─────────────────────┘    │   │  └────┬───┘ └─────────┘ └──────────┘
 │                   │ message passing            │   │       │
 │                   ▼                            │   │       ▼
 │   ┌─────────────────────────────────────┐    │   │       │
 │   │  Background script / resident process │◀───┼───┼──── Data collection pipeline
 │   │  • Cross-tab / cross-session state    │    │   │     ┌──────────────────────────┐
 │   │  • The sole outbound network exit ───┼────┼───┼───▶│ Crawler + crowdsourced       │
 │   │  • Manages login state, coordinates   │    │   │     │ submissions + manual review │
 │   │    attribution reporting              │    │   │     │ → verify if coupons still    │
 │   └───────────────┬─────────────────────┘    │   │     │   work                       │
 │                   │                            │   │     └──────────────────────────┘
 │                   ▼                            │   │
 │   ┌─────────────────────────────────────┐    │   │
 │   │  Popup / standalone UI                │    │   │
 │   │  • View cashback balance, account,    │    │   │
 │   │    toggle settings                    │    │   │
 │   └─────────────────────────────────────┘    │   │
 └───────────────────────────────────────────────┘   │
        Browser sandbox boundary (platform's call)  Trust boundary (data leaves the user's machine)
```

> The soul isn't any single box — it's **that narrow channel running through "content script → background script → backend."** What happens on the page has to pass through one narrowing gate after another (read only the necessary DOM, go out to the network through the single background exit, report only the necessary fields) before it becomes a backend query or an attribution record. **The whole art of the architecture lies in keeping every segment of this channel as "touch less, send less, store less" as possible.**

## 5. Component responsibilities

Each key part below: **what it does + why it's needed** (a part with no "why" is over-engineering).

- **Content Script**: injected by the platform into the host page; the only part that can directly read/write the current page's DOM. It handles **identifying the current page** (is this a given merchant's checkout page? is this an enhanceable input field?), **injecting UI** (coupon hints, rewrite suggestions), and **performing actions** on the page (fill the coupon code into the field, click apply). *Why it's needed*: only it can touch the host page; the whole "value happens on someone else's page" thing can, physically, only be realized by it. **But it should be as "dumb" as possible** — doing only what's strongly tied to this page's DOM, never stuffing business logic and keys into it (see Decision 1).
- **Background script / resident process**: the extension's "brain" and its **sole outbound network exit.** It holds the **cross-tab, cross-session state** (login state, the order being tracked, pending cashback), coordinates the content scripts across tabs, and centralizes communication with the backend and attribution reporting. *Why it's needed*: content scripts are destroyed on page refresh and isolated from each other, so you need a role that "lives longer and sees more" to hold state and funnel network requests. Centralizing networking here makes it easy both to unify auth/rate-limiting and to audit clearly "which networks were touched."
- **Popup / standalone UI**: the panel that pops up when the user clicks the extension icon, showing cashback balance, account, and setting toggles. *Why it's needed*: this is the extension's **own, controllable** slice of UI — the right place for content "the user actively comes to view/operate"; it's a different thing from the UI injected into the host page, and the two shouldn't be conflated.
- **Backend: coupon store / dictionary**: holds massive coupons and merchant rules (for writing tools, grammar rules / dictionaries). *Why it's needed*: this data is large, frequently updated, and must be shared across all users and retrieved fast — putting it on the client both won't fit and can't be updated in time (see Decision 4).
- **Backend: merchant rule store / attribution rules**: what each merchant's page looks like, which input field the coupon code goes in, how attribution is recorded. *Why it's needed*: merchant pages vary endlessly and get redesigned often; turning "how to recognize, how to operate" into **backend-pushable rules/config** is the only way to adapt to a merchant's redesign without shipping a new extension version (see Decision 4).
- **Backend: user account / cashback ledger**: user identity, accumulated cashback, withdrawal records. *Why it's needed*: cashback is money — it must be strongly consistent, auditable, and never miscounted; this kind of data naturally belongs on the server.
- **Data collection pipeline (crawling + crowdsourcing + validation)**: continuously gathers coupons from everywhere, receives crowdsourced submissions from users, and **repeatedly verifies "does this coupon still work right now?"** *Why it's needed*: the core value of a coupon is "it actually works," yet coupons naturally expire and get pulled by merchants; without a pipeline that continuously refreshes and validates, the coupon store rots fast (see Decision 5).

## 6. Key data flows

A few scenarios that best capture this system's character.

**Scenario 1: auto-find coupons at checkout + apply the best + record attribution (the most central path for cashback)**
```
1. User enters a merchant's checkout page
   ──▶ Content script detects the URL / page features: matches a "supported merchant"
2. Content script ──message──▶ Background: "user is on merchant X's checkout page, get me coupons"
3. Background ──▶ Backend: query merchant X's current available coupon list + that merchant's page-operation rules
       (Backend: filter the coupon store for unexpired coupons that apply to this order, return them sorted by estimated discount)
4. Background ──message──▶ Content script: push down the coupon list + operation rules
5. Content script "tries coupons one by one" on the page:
       fill in the code → click apply → read the new total on the page → record how much this coupon saved
       ⟲ Loop through the candidate coupons (or stop once it's good enough)
6. Content script applies "the one that saved the most," hints to the user on the page "saved you ¥XX"
7. If the user goes on to place the order ──▶ Background records attribution (this order was driven by this extension)
   ──▶ Backend cashback ledger logs a "pending cashback"
       (After the merchant later confirms the order is valid, that cashback turns into "withdrawable")
```
> Note two things: ① **the dirty work of "trying coupons" — tightly coupled to the page, repeatedly reading the DOM — can only happen in the content script;** while the business judgment of "which coupons are worth trying, in what order" should be computed by the backend and pushed down. ② The attribution in step 7 is the lifeblood of the whole business model, and also where the ethical red line lies (see Section 10).

**Scenario 2: writing assistance — real-time enhancement of any input field**
```
User types in "any input field on any website"
   ──▶ Content script listens to the input, recognizes this as an enhanceable editing area
   ──▶ Content script hands the "text to check" to the background script
   ──▶ Background ──▶ Backend: do grammar/rewrite analysis (heavy models on the backend)
   ──▶ Result returned ──▶ Content script injects "suggestion underlines / rewrite cards" beside the field
   ──▶ User clicks accept ──▶ Content script writes the change back into the field's DOM

⚠️ Privacy point: what the user types in "any input field" may be passwords, private messages, medical records.
   Architecturally you must be explicit: which fields are NEVER collected (password fields, fields marked sensitive),
   and that what's transmitted is "the minimum text needed for the check," not "everything visible on the page."
```

**Scenario 3: coupon data collection and freshness (the continuous backend-side process)**
```
Source A: crawler periodically scrapes coupon sites / merchant promo pages ─┐
Source B: users at checkout crowdsource-submit "this code works"           ─┼─▶ Candidate coupon pool
Source C: partner channel imports                                          ─┘        │
                                                                                    ▼
                                                  Validator: try the rules to check validity
                                                  (expired? minimum spend? region-locked?)
                                                                                    │
                                            ┌───────────────────────┼───────────────────────┐
                                            ▼                       ▼                       ▼
                                       Mark valid              Mark expired           Uncertain → manual review
                                            │
                                            ▼
                                  Enter the "pushable" coupon store ──▶ served to Scenario 1's query
```
> This pipeline exists because **a coupon's value decays over time**: a code that works today may be pulled tomorrow. Freshness isn't a one-time problem; it's a "freshness-keeping" effort requiring continuous investment.

## 7. Data model and storage choices

Core entities: `user` ─ `account/cashback ledger` ─ `cashback record`; `merchant` ─ `coupon` / `page-operation rule`; `attribution event`; `crowdsourced submission`. For writing tools, swap "coupon/merchant rule" for "grammar rule / user dictionary."

| Data | Storage type | Why |
|---|---|---|
| User / account / cashback ledger | Relational | Money is involved; needs transactions, strong consistency, and auditability — must never miscount |
| Cashback / withdrawal records | Relational / append log | A monetary ledger is append-only and reconcilable; needs strict consistency and a clear trail |
| Coupon store (massive, frequently updated) | Document + search index | Large in volume, flexible in structure (different merchants have different fields), must filter fast by merchant |
| Merchant page-operation rules | Document / pushable config | Essentially "config pushed down to the client"; must be updatable without shipping a new version |
| Available coupons for hot merchants | In-memory KV cache | Must return in milliseconds at the moment of checkout, and the same merchant is queried repeatedly (read-heavy, write-light) |
| Attribution events / usage telemetry | Time-series / columnar | Massive, aggregated by time, used for reconciliation and analysis |
| Client-side local state (login state, settings) | Browser local storage | Small in size, lives with the user on their machine — shouldn't and needn't be uploaded to the server |

> Teaching point: **note the "what the client should store" row — it's the privacy boundary expressed in the data model.** Any state that "works fine without being sent to the server" (toggles, preferences, temporary tracking) stays on the user's machine; only data that "must be shared / must be strongly consistent (money)" goes to the server. Every copy of user data you can avoid storing on your servers shrinks your risk surface by one.

## 8. Key architectural decisions and trade-offs ⭐

**(The most valuable section of this template.)**

**Decision 1: put logic in the content script, or in the background script / backend? (the first-principles question of extension architecture)**
- Cram it all into the content script: it can manipulate the page directly, which looks "close at hand and convenient." But the cost is huge: ① content scripts are destroyed on page refresh and isolated from each other, so they can't hold state; ② sharing the page with the host, **logic and keys are exposed in a hostile environment** — easily spied on or interfered with by the host page or malicious scripts; ③ business logic hardcoded in the client means changing it once requires shipping a version, so iteration is glacial.
- Content script does only "the dirty DOM work," with business judgment moved up to the background script and backend: the content script only "reads the page, fills the page, injects UI"; "whether to fill, which coupon to fill, how to record attribution" goes to the background/backend.
- **Verdict**: **resolutely keep the content script as thin as possible.** It's the "hand" you reach into someone else's page with — it should do only what a hand does; the "brain" lives in the background script (cross-page state) and the backend (business rules). The cost is designing the message protocol between content script ↔ background ↔ backend, plus an extra hop of communication latency. But it's worth it: **the simpler the injection point, the safer, more maintainable, and faster to adapt to a merchant's redesign.**

**Decision 2: least privilege — don't request a single permission you can do without**
- Request the broad permission "access all websites + read all data": easy for the developer, can meddle with any page. But the cost is: ① stricter platform review and harder approval; ② users see "can read your data on all websites" at install time and hesitate or even refuse; ③ once you're breached, the attacker inherits this same broad permission — **you've blown your own risk surface up to the maximum.**
- Request permissions on demand and progressively: only request permission for a given merchant when the user actually uses it; prefer a narrow API over a broad one.
- **Verdict**: **least privilege is the bedrock of this kind of product, not an option.** Every permission you request is overdrawing user trust, enlarging your risk surface, and increasing platform risk. The cost is a more cumbersome implementation (handling the "permission not yet granted" state, guiding the user to authorize), but this is precisely how the number-one quality attribute, "trust," gets cashed out in the architecture.

**Decision 3: how much does the client do, how much does the backend do?**
- Lean client-side: put merchant identification, coupon filtering, and discount computation locally. The upside is it runs offline, doesn't depend on the network, and doesn't send browsing behavior to the server (better privacy). The cost is slow logic updates (requires shipping a version), limited client capability, and an inability to handle heavy computation.
- Lean backend: the client only collects minimal input and hands the judgment to the server. The upside is logic can be updated anytime, capability is strong, and adapting to a merchant's redesign is fast. The cost is **one more "send data to the server" privacy exposure,** plus a strong dependence on the network and backend availability.
- **Verdict**: **cut along the "privacy sensitivity" line.** Put the non-sensitive, frequently-rule-updated parts (coupon filtering, merchant adaptation) on the backend; keep the privacy-touching parts (what the user typed in a field, which sites they visited) on the client as much as possible, and upload only "the minimum fragment necessary to complete the task." **"Should this piece of data be sent out at all" always outranks "is this implementation convenient."**

**Decision 4: merchant-adaptation rules — hardcode into the extension, or push from the backend?**
- Hardcode: each merchant's page recognition and coupon-filling logic is written into the extension code. Simple and direct, but the moment a merchant redesigns you break, and you **can only fix it by shipping a new version** (and then waiting for platform review and for users to update).
- Backend-pushable config/rules: turn "how to recognize a merchant, which field the coupon goes in, how to record attribution" into data pushed by the backend, fetched when the extension starts.
- **Verdict**: **once you scale (supporting dozens to hundreds of merchants), pushable is a must.** Otherwise you'll be dragged to death by "merchants redesign at any time." The cost is designing a way to describe and push rules, and the client being able to safely execute these pushed rules (note: what's pushed is "config/selectors," NOT "arbitrary executable code" — otherwise you've opened a remote-injection backdoor on yourself).

**Decision 5: coupon freshness — verify in real time, or keep fresh with background batches?**
- Verify each coupon in real time when the user checks out: most accurate, but slow (the user is waiting), and it piles the verification load onto the most critical moment.
- A background pipeline continuously batch-verifies, tags coupons "valid/expired," and at checkout you take only the ones already verified as valid: fast, but there's a window of "just expired but not yet re-checked."
- **Verdict**: **background freshness-keeping primary, real-time as a supplement.** The background pipeline continuously scrubs the coupon pool clean; at checkout, prefer coupons "recently verified as valid," and feed the signal "found expired while trying" back into the pipeline (i.e., use real checkout behavior to help keep things fresh). The cost is operating a continuous data pipeline and accepting a small expiry window — but that's a far better trade than "making the user sit and wait for verification at checkout."

**Decision 6: how attribution is implemented, and where its boundary is (a decision where technical and ethical considerations coexist)**
- Aggressive approach: regardless of whether the traffic was originally brought in by someone else (another promoter, another channel), overwrite it all to "I brought it in" to maximize commission. Technically doable; high short-term revenue.
- Restrained approach: record attribution only when "the order was genuinely driven by this extension"; don't overwrite the user's existing attribution from others.
- **Verdict**: **you must choose restraint.** Overwriting others' attribution (commonly "attribution hijacking"), even if technically feasible, brings real legal risk, retaliation from partners, and — once exposed — reputational collapse; and reputation is the lifeline of a product that "can see everything you browse." **This is a rare case in architectural decision-making where an ethical constraint directly becomes a hard constraint: the business model must be built on attribution that can stand up, or the foundation of the whole edifice is illegal.**

## 9. Scaling and bottlenecks

Unlike an ordinary website: **the "scale" pressure on this kind of system comes both from user volume and from "the number and rate of change of the external world (merchants/sites) you have to adapt to."**

- **First bottleneck: changes to platform policy and the permission model.** (The most unusual, most easily underestimated bottleneck.)
  Fixes: ① architecturally **gather and isolate** the parts that "depend on a particular platform feature" so they're replaceable; ② don't bet your core capability on some API that may be tightened at any moment; ③ continuously track platform policy and migrate ahead of time. **This isn't a traffic problem — it's a "the ground you stand on can move at any time" problem; no amount of servers can scale your way out.**
- **Second bottleneck: the more and messier the merchants/sites, the more the maintenance of adaptation rules explodes.**
  Fixes: ① make adaptation rules backend-pushable (Decision 4), downgrading "fixing" from "shipping a version" to "editing config"; ② use more general page-recognition strategies to reduce per-merchant hardcoding; ③ wire in crowdsourcing/feedback so real usage data helps you spot broken adaptations.
- **Third bottleneck: backend queries (coupons/rules) get maxed out during checkout peaks.**
  Fixes: ① put hot merchants' available coupons in an in-memory KV cache (read-heavy, write-light — perfect for caching); ② optimize the coupon store's search index; ③ push down "what the client can judge" as much as possible to cut backend round-trips.
- **Fourth bottleneck: the data collection pipeline can't keep up with the rate coupons rot.**
  Fixes: ① scale the validator horizontally; ② tier freshness-keeping by "merchant heat" (re-check popular merchants frequently, cold ones infrequently); ③ use real checkout feedback to feed validation.
- **Cashback-ledger consistency isn't a bottleneck but is a high-pressure zone**: money is involved, so better slow than wrong. It's usually not a performance bottleneck but a "must never be wrong" correctness red line — hold it with relational storage + strict reconciliation.

## 10. Security and compliance essentials

This is the **heaviest section** for this kind of system, because its capabilities push naturally to the very edge of privacy.

- 🔴 **"Being able to see everything" is the biggest attack surface and the biggest liability.** The extension can observe almost every page the user visits. Architecturally you must make the **collection boundary** a first-class citizen: explicitly list what is **never collected** (password fields, payment card numbers, fields marked sensitive, pages unrelated to the function), transmit "the minimum data necessary to complete the task" rather than "send the whole page for convenience." This boundary must be **explainable and auditable** to both the user and the platform.
- 🔴 **The extension's super-strong permissions = once breached, the consequences are amplified.** Therefore: least privilege (Decision 2), the background script as the sole network exit for easy auditing, push only "config" not "arbitrary executable code" (otherwise you ship a built-in remote-code-injection vulnerability), and encrypt backend communication.
- 🔴 **The legal and ethical boundary of attribution.** Attribution hijacking (overwriting attribution others brought in) carries real legal risk; in both architecture and product strategy you must hold the line of "record only attribution genuinely driven by yourself" (Decision 6). This is not just compliance — it's the trust foundation of this kind of product.
- **The isolation boundary between the content script and the host page.** The content script must guard against being exploited or fooled by malicious scripts on the host page; injected UI shouldn't expose sensitive data anywhere the host page can read it.
- **"Informed consent" for data collection.** What's collected, what it's used for, whether it's sold/shared — all must be transparent, switchable off, and deletable; especially when monetization itself depends on user data, the boundary must be even clearer.
- **Platform risk is compliance risk.** Platform policy (data use, permissions, monetization methods) is itself the "law" you must obey; violate it = takedown.

## 11. Common pitfalls / anti-patterns

- ❌ **Cramming business logic and keys into the content script** → ✅ The content script does only "the dirty DOM work"; move logic up to the background and backend; the thinner the injection point, the safer (Decision 1).
- ❌ **Requesting the broad "access all websites" permission for convenience** → ✅ Request minimally, on demand, progressively; every permission overdraws trust and amplifies risk (Decision 2).
- ❌ **Letting "convenient to implement" override "should this data be sent at all"** → ✅ Keep privacy-sensitive data on the client as much as possible; upload only the minimum fragment to complete the task (Decision 3).
- ❌ **Hardcoding merchant-adaptation logic so it can only be fixed by shipping a version** → ✅ Make adaptation rules backend-pushable, downgrading fixes from "shipping a version" to "editing config" (Decision 4).
- ❌ **Cramming the entire coupon store into the client to save one request** → ✅ Massive, frequently-updated, shared data goes on the backend; the client takes only what it currently needs (Decisions 4/5).
- ❌ **Pushing "executable code" for the extension to run** → ✅ Push only data like config/selectors; pushing executable code is installing a remote-injection backdoor on yourself.
- ❌ **Overwriting others' attribution to earn more commission** → ✅ Record only attribution genuinely driven by yourself; attribution hijacking is a double landmine of legal and reputational risk (Decision 6).
- ❌ **Making injected UI steal the show and slow down the host page** → ✅ You're a guest; inject with restraint and never break the host page's core flow.

## 12. Evolution path: MVP → growth → maturity

| Stage | Scale | What the architecture looks like | What to worry about now |
|---|---|---|---|
| **MVP** | Validate the idea, 1 or a few merchants | A thin content script + a background script + a simple backend; coupon store maintained by hand; request only the minimum permission for the needed sites; attribution via the most direct, compliant method | **First validate "users really will use it, and the commission model really holds up"** — don't build a collection pipeline up front |
| **Growth** | Dozens to hundreds of merchants, millions of users | Merchant-adaptation rules become backend-pushable; build the data collection pipeline (crawling + crowdsourcing + validation); hot coupons go into cache; add transactions and reconciliation to the cashback ledger; round out the popup UI and account system | Decouple adaptation from "shipping a version," hold data freshness, keep attribution compliant |
| **Maturity** | Massive merchants / affiliate networks, tens of millions of users | A full cashback ledger + affiliate-network integration + a closed-loop crowdsourcing pipeline; a freshness pipeline tiered by heat; an architecture replaceable in the face of platform-policy change; strict privacy boundaries and audit capability | Platform risk and compliance, data trust, attribution at scale and its legality, the cost of the collection pipeline |

## 13. Reusable takeaways

- 💡 **When you "live parasitically" in an environment you don't control, make the part that touches that environment as thin as possible.** The simpler the content script, the safer — this is the same as "when depending on an external system, make the adaptation layer thin and guard the core logic on your own side."
- 💡 **Least privilege isn't a security clause — it's product strategy.** Every capability you request is overdrawing trust and amplifying your risk surface. First ask "can I do this without this permission?"
- 💡 **Partition client vs. server along "privacy sensitivity," not "implementation convenience."** "Should this data be sent out at all" always outranks "is this convenient to write."
- 💡 **Treat "data that rots over time" as an engineering effort requiring continuous freshness-keeping, not a one-time pour.** Coupons are like this, and so is any data that "goes stale the moment the external world changes" (prices, inventory, a competitor's page structure).
- 💡 **When the business model is built on one technical act (here, attribution), the legal and ethical boundary of that act is the hardest constraint in your architecture.** Being able to do it technically doesn't mean you should — sometimes the ethical red line *is* the system's hard requirement.
- 💡 **Beware that "the ground you stand on can move on its own."** Platform policy and third-party page structures are both outside your control; architecturally, gather, isolate, and make replaceable your dependencies on them.

## 🎯 Quick quiz

<Quiz
  question="When a browser plugin requests permissions, the correct principle is?"
  :options="['Request as many as possible, just in case', 'Least privilege: resolutely refuse any permission you can do without', 'How many does not matter — users will agree anyway']"
  :answer="1"
  explanation="Least privilege isn't just a security clause but a product strategy — every capability you request is overdrawing user trust and amplifying your attack surface."
/>

---

## References & Further Reading

> This template is compiled from the following **official docs** and **real open-source extensions**.

**📖 Official docs:**
- [MDN: Anatomy of a WebExtension](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Anatomy_of_a_WebExtension) — the separation of responsibilities between content script and background script, and message passing.
- [Chrome: Migrate to a service worker (Manifest V3)](https://developer.chrome.com/docs/extensions/develop/migrate/to-service-workers) — under MV3, the background is replaced by a service worker, runs on demand, and state must be persisted.

**🔧 Open-source prototypes (read the code directly):**
- [gorhill/uBlock](https://github.com/gorhill/uBlock) — a classic open-source extension, embodying the engineering practice of content filtering / page injection and privacy protection.

---

> 📌 Remember the browser extension in one line: **it's the peculiar thing that "lives inside someone else's page yet can see everything the user does." Every architectural trade-off ultimately answers: 'how do you make this injected layer both useful and trustworthy, with the least permission and the most restrained data?'**
