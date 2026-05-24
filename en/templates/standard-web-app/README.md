# Standard Web App · Architecture Template

> **Representative products**: company websites, blogs, small-to-mid SaaS back-offices, internal tool sites — i.e., "the architecture 90% of projects actually need"
> **One-line definition**: a system that stores data well, reads and writes it by permission, and renders it into pages for users. It isn't sexy, but it can carry the overwhelming majority of scenarios you *think* need "distributed."

---

## 1. One-line definition

A standard web app = **one monolithic application** + **one database** + **(when needed) a layer of cache.**

The most counterintuitive thing architecturally is precisely that there **aren't that many bells and whistles.** The mistake modern engineers make most easily isn't designing a system too simply — it's **making the architecture complex before they've even hit a problem.** This template's core stance is one sentence: **the overwhelming majority of systems die of over-engineering, not under-engineering.** It exists to teach you something harder than "knowing how to do microservices" — **restraint**: seeing clearly just how far "a monolith + one database + cache" can carry you (the answer: far beyond what you imagine), so you can save your precious complexity budget for the day you truly need it.

## 2. The business essence: what problem it solves

The overwhelming majority of software, at its core, does one plain thing: **store information, read it and change it back by rules, and present it to the right people.**

- Company website / blog: store the content well, render it into pages for visitors (reads far outnumber writes);
- SaaS back-office / internal tool: let users log in and CRUD some business data (orders, customers, tickets, articles) by permission;
- Tool site: take input, process it, return a result.

It replaces the clumsy way of "Excel / paper forms / emailing back and forth."

Where the value and cost come from:
- Value: it saves the manual shuffling, centralizes management, and is queryable any time;
- Cost: **the biggest cost of this kind of system is often not the servers, but "the complexity you pay for a scale that doesn't exist"** — superfluous services, superfluous middleware, superfluous ops are all burning money and manufacturing failures.

> **The key fact: for 90% of projects, "can it carry the traffic" simply isn't the bottleneck; "can you get features right — and changed right — quickly and cheaply" is.** This one fact decides: simplicity is itself a core competitive advantage.

## 3. Core requirements and constraints

Split requirements into two kinds. **This is the architect's most important fundamental: distinguishing "function" from "quality."**

**Functional requirements (what the system must be able to do):**
- [ ] Authentication and authorization: who can log in, who can see/change what.
- [ ] CRUD: basic operations on the core business entities.
- [ ] Page rendering: turn data into an interface users can see.
- [ ] Handle user uploads (images, attachments).
- [ ] Some async/scheduled tasks (send email, generate reports, clean up data).

**Non-functional requirements / quality attributes (how *well* the system must do it):**
| Quality attribute | Target | Why it matters for this kind of system |
|---|---|---|
| **Changeability / dev speed** | Changing a feature should be fast and safe | This is this kind of system's true battlefield — the business is changing; whoever changes faster wins. |
| **Simplicity / maintainability** | A newcomer can grasp the whole picture in a few days | Simple = fewer failures, easier hiring, fearless changes. A severely underrated quality attribute. |
| **Availability** | 99.9% is usually enough | Don't chase 99.999% from the start — that costs exponentially more complexity. |
| **Response latency** | A few hundred ms for ordinary pages | Fast enough is enough; the vast majority of scenarios don't need extreme optimization. |
| **Ops cost** | The lower the better | A small team's manpower is the scarcest resource; the simpler the ops, the more energy goes into the business. |

**Key constraints (boundaries you cannot cross):**
- 🔴 **Small team, limited budget, manpower is the number-one scarce resource.** This is this kind of system's most important constraint, and it directly dictates "the simpler the better."
- 🔴 **Business requirements change continuously.** The architecture must serve "change fast and safely," not "design once, never change."
- 🔴 **The real scale is usually far smaller than you imagine.** You're not a tech giant — don't draw your diagram from a giant's blueprint.
- The complexity budget is finite: every component you add consumes the team's energy to understand, operate, and debug.

## 4. The big-picture architecture

```
                        User (browser / mobile)
                              │
                              ▼
                    ┌──────────────────┐        Static assets (JS/CSS/images)
                    │       CDN        │◀───────  served close to the user; don't make the
                    └────────┬─────────┘          app server wait on static files
                             │ dynamic request
                             ▼
                    ┌──────────────────┐
                    │  Load balancer    │   ← app layer is stateless, so you can run several, add freely
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
      ┌────────────┐ ┌────────────┐ ┌────────────┐
      │ App instance│ │ App instance│ │ App instance│   ← a monolith, internally the classic 3 layers:
      │ ┌────────┐ │ │ (stateless) │ │ (stateless) │      presentation → business logic → data access
      │ │present.│ │ └────────────┘ └────────────┘
      │ ├────────┤ │        │                              ┌───────────────┐
      │ │bus.log.│ │        │   ① check cache first ──────▶ │ In-memory KV  │
      │ ├────────┤ │        │   ◀── hit? return directly ── │     cache     │
      │ │data acc│ │        │   ② only on a miss, hit the DB └───────────────┘
      │ └────────┘ │        ▼
      └─────┬──────┘ ┌───────────────────────────┐
            └───────▶│      Database (primary)     │ ← usually the [first] and longest-lived bottleneck
                     │   (one is enough for most)  │
                     └─────────────┬─────────────┘
                                   │ (add later when scale grows) read replication
                                   ▼
                     ┌───────────────────────────┐
                     │      Read-only replica      │ ← when reads >> writes, offload reads here
                     └───────────────────────────┘

       ┌─────────────────────────────────────────────────────────────┐
       │  Side path: async tasks (send email, generate reports…)       │
       │  App ──enqueue──▶ [task queue] ──▶ background worker ──▶ write DB / call external service │
       └─────────────────────────────────────────────────────────────┘

       ┌─────────────────────────────────────────────────────────────┐
       │  Side path: user-uploaded images/attachments ──▶ [object storage] ──▶ served back via CDN │
       └─────────────────────────────────────────────────────────────┘
```

> The soul is that middle **monolithic app + one database.** The CDN, cache, read replicas, task queue, object storage — **these are all "add only when needed" side paths,** not opening defaults. **Getting those two core boxes solid matters far more than drawing a wall of boxes early.**

## 5. Component responsibilities

Each key part below: **what it does + why it's needed** (a part with no "why" is over-engineering).

- **Monolithic app (containing the classic 3 layers)**: an independently deployable program, internally divided clearly into a **presentation layer** (handle request/response, render), a **business logic layer** (rules, flows, transactions), and a **data access layer** (talk to the database). *Why it's needed*: it's where all the system's business lives. **Note: "monolith" refers to the deployment form (one thing released together), NOT "spaghetti code"** — internally it still layers and keeps clear module boundaries. A monolith lets you debug, transact, and change all within a single process, at the lowest dev and ops cost.
- **Database (primary)**: the system's **source of truth**, storing all core business data and providing transactions and strong consistency. *Why it's needed*: your data needs an authoritative home that won't lose it and can guarantee consistency. For the vast majority of projects, **one relational database is enough to carry you for a very, very long time.**
- **Load balancer**: distributes traffic across multiple app instances. *Why it's needed*: it lets the app layer "run several" to carry load; but its precondition is that **the app layer is stateless** (see Decision 5).
- **CDN**: caches static assets (JS, CSS, images) on nodes close to the user. *Why it's needed*: there's no point making the app server serve static files every time; hand them to the CDN — it's both fast and offloads a lot of traffic. This is extremely high value-for-money and can be added fairly early.
- **In-memory KV cache**: caches "read-a-lot, change-little, expensive-to-compute" results in memory, in front of the database. *Why it's needed*: the database is usually the first bottleneck, and a cache can block a lot of repeated reads. **But it has a real cost (cache consistency), so it's "added when you hit read pressure," not an opening default** (see Decision 3).
- **Read-only replica**: a read-only copy of the database, dedicated to absorbing read traffic. *Why it's needed*: when reads far outnumber writes, offload reads to the replica to relieve the primary. **Also added only on demand** (see Decision 4).
- **Task queue + background worker**: move "slow things that can be done later" (send email, generate reports, third-party calls) out of the main request path to execute asynchronously. *Why it's needed*: don't make the user wait there for an email to send; making time-consuming operations asynchronous is the only way the main request stays fast.
- **Object storage**: stores large files like user-uploaded images and attachments. *Why it's needed*: large files shouldn't be stuffed into the database (it bloats, slows down, makes backups hard); object storage is built for "large, rarely changing, fetched by key," and pairs with a CDN to serve users.

## 6. Key data flows

Two scenarios that best capture this kind of system: one read, one write. **You'll find it reassuringly plain.**

**Scenario 1: a read request (e.g., opening a detail page)**
```
1. User requests a page ──▶ load balancer ──▶ some app instance
2. App (business logic layer) asks the cache first: is there a ready copy of this data?
       ┌── hit ──▶ use it directly, jump to step 4 (DB untouched, fastest)
       └── miss ──▶ step 3
3. App (data access layer) queries the database ──▶ gets the data ──▶ writes it into the cache on the way (next time it'll hit)
4. Business logic layer organizes the data ──▶ presentation layer renders it into a page/JSON ──▶ returns to the user
```
> This is the whole essence of "**cache blocking reads**": let the most frequent reads be satisfied mostly at the cache layer; the database is only disturbed on a cache miss.

**Scenario 2: a write request (e.g., submitting a form / updating a record)**
```
1. User submits ──▶ load balancer ──▶ some app instance
2. Business logic layer: ① validate input (is it valid? authorized?)
3.                       ② write to the database inside a transaction (this is the source of truth, must be right)
4.                       ③ invalidate the relevant cache (delete/update the entry just changed)
       ⚠️ Order matters: write the DB first (authoritative), then invalidate the cache. Otherwise you may read stale data.
5. (Optional) ④ throw "side effects that can be done later" into the task queue: notification email, audit log, refresh reports
6. Return the result to the user (the user needn't wait for the async work in step 5 to finish)
```
> Two points: ① **a write is always anchored to the source of truth, the database; the cache is just its shadow** — and the shadow must be invalidated promptly after the data changes. ② **Side effects that can be async shouldn't block the main path**; the user waits only for "the part that must complete synchronously."

## 7. Data model and storage choices

The core entities are usually plain: `user` ─ `role/permission`; several **business entities** (orders / articles / tickets / customers) and their relationships; `uploaded file`; `audit/operation log`. Among them is a clear, strongly-relational structure.

| Data | Storage type | Why |
|---|---|---|
| Users / permissions / core business entities | Relational | Strong relationships between entities, transactions and strong consistency needed; this is exactly what relational excels at — **don't go looking far afield** |
| Hot read results (details, lists, config) | In-memory KV cache | Read-heavy, write-light, tolerant of a very brief inconsistency; sits in front of the DB to save effort |
| User-uploaded images / attachments | Object storage | Files are large, rarely change, fetched by key; stuffing them into the DB bloats it, slows it, and makes backups a nightmare |
| Static assets (JS/CSS/images) | CDN (edge cache) | Immutable, want fast global proximity; shouldn't consume the app server |
| Operation / audit logs | Relational (consider append log/columnar once volume grows) | A single table is enough early on — **don't bring in dedicated storage ahead of time for "it might get big someday"** |
| Session state | External shared storage (cache/DB), not in-process memory | In-process memory causes session stickiness and won't scale (see Decision 5) |

> Teaching point: **default to one relational database until you have concrete evidence it's not enough.** "Will this data get big someday?" "Might I need full-text search?" — these "might in the future" aren't reasons to bring in dedicated storage now. **One relational database + sensible indexes can accompany the vast majority of projects through their entire lives.** Adding it when you actually hit the wall is cheap; adding it ahead of time for things that haven't happened is expensive.

## 8. Key architectural decisions and trade-offs ⭐

**(The most valuable section of this template. Nearly every decision here points to the same answer: take the simple path first.)**

**Decision 0 (above all else): don't ask "how do I scale" first; ask "at this scale, right now, do I actually need it?"**
- This isn't one particular technical fork — it's a meta-decision running through the whole text. **The cost of over-engineering is real and immediate**: more components = more to understand, more ops, a bigger failure surface, slower development. And the "future scalability" it buys is often **a scale you'll never reach on this project in your life.**
- **Verdict**: **default to refusing complexity; let the requirements "force" you to add things, instead of you adding them in anticipation.** Before adding any component (cache, queue, a second service, a read replica), answer: "do I have a concrete problem right now that nothing else can solve?" If you can't answer, don't add it. This is the single most important rule of this template.

**Decision 1: server-side rendering (SSR) or client-side rendering (CSR)?**
- Server-side rendering: the server assembles the HTML and returns it directly. The upside is **a fast first screen and SEO-friendliness** (crawlers see the content directly), and a more direct implementation. The cost is somewhat weaker interactivity and the server bearing the rendering.
- Client-side rendering: return an empty shell + a pile of scripts that pull data and assemble the UI in the user's browser. The upside is **a rich interactive experience** (like a desktop app). The cost is a slow first screen, extra work for SEO, and higher complexity.
- **Verdict**: **it depends on the product form.** Content-type (websites, blogs, e-commerce detail — must be findable, must have a fast first screen) favors server-side rendering; interaction-heavy back-offices/tools (the user logs in and operates for a long time, SEO is irrelevant) suit client-side rendering. **Don't put a heavyweight frontend on a blog just because "client-side rendering is trendy"** — that's trading complexity for something you don't need.

**Decision 2: monolith or microservices?**
- Microservices: split by business into multiple independently deployed services. The upside is each service can scale independently, deploy independently, use heterogeneous tech, and teams can work in parallel. But the cost is **extremely expensive**: distributed transactions, inter-service communication and failures, data consistency, observability, deployment orchestration — **you've replaced "a function call within one process" with "a network call that can fail and has latency,"** and complexity spikes.
- Monolith: an independently deployable program with clear internal layering. The upside is that development, debugging, transactions, and deployment are all simple, and a small team is most efficient. The cost is "release as a whole," plus the collaboration friction of a single codebase at very large scale.
- **Verdict**: **monolith first! Almost always monolith first.** Microservices solve an "**organizational scale**" problem (many teams need to work in parallel without blocking each other), not a "technical advancement" problem. Adopting microservices when you have only one small team and your business boundaries haven't even stabilized is conjuring up the full suffering of a distributed system for yourself while reaping none of its benefits. **The correct path is: build a monolith with clear internal modules first, and only when the organization/scale genuinely outgrows it, split along the already-clear module boundaries** (see Section 12).

**Decision 3: when should you add a cache?**
- Add a cache from the start: seemingly "faster," but you've introduced the classic problem of **cache consistency** (data changed but cache not invalidated = users see stale data), conjuring up a whole class of bugs — and at this point you may have no read pressure at all.
- Add it when read pressure appears: when the database can't handle the repeated reads, cache the "read-heavy, write-light, tolerant of a very brief inconsistency" data.
- **Verdict**: **add it only when reads >> writes and the database genuinely starts to struggle, not as an opening default.** A cache is a trade of "a bit of consistency complexity for the performance of many repeated reads" — when there's no read pressure, you're net negative on this trade (you've paid the complexity but reaped no benefit).

**Decision 4: when should you do read/write splitting (add a read replica)?**
- Split too early: maintain primary-replica replication, deal with replication lag (read a replica right after a write and you may read stale data), and add ops for read volume that doesn't exist.
- Split when timely: when **read traffic** clearly becomes the primary's bottleneck and the business can tolerate the replica's slight lag, offload reads to a read-only replica.
- **Verdict**: **hold the line with indexes and cache first; only when those aren't enough and the bottleneck is genuinely on "reads," add a read replica.** Note it solves "too many reads," not "too many writes" — write-heavy is a different class of problem (encountered later, addressed differently). **Don't set up a primary-with-replicas while the database is still idle.**

**Decision 5: stateful or stateless app layer?**
- Stateful (store session/data in the memory of some app instance): convenient on a single machine, but the moment you go multi-instance, big trouble — the user's requests must return every time to "the machine that stored their state" (session stickiness), if that machine dies the user's state is lost, and **you simply can't scale horizontally freely.**
- Stateless (app instances store no session state; state lives in external shared storage): any instance can handle any request.
- **Verdict**: **the app layer must be stateless.** Put session and other state into external shared storage (cache or database). This is the precondition for "load balancing + horizontal scaling" to work, **and one of the few disciplines you "should follow from day one"** — it adds no complexity, yet clears the biggest obstacle to future scaling. The cost is merely one extra external read to fetch state each time, which is entirely worth it.

## 9. Scaling and bottlenecks

This section answers: from a few hundred users to many users, **where does the first thing buckle?** The order is highly regular; **remember this order and you won't expend effort in the wrong place ahead of time.**

- **The first bottleneck is almost always the database.** (Especially reads.)
  Fixes (try in order, from low cost to high):
  ① **Add indexes** — the cheapest, most-overlooked optimization; a lot of "slow" is actually a missing index or a full-table scan;
  ② **Add a cache** — block hot repeated reads in front of the DB (Decision 3);
  ③ **Add a read replica** — offload read traffic (Decision 4);
  ④ Only when those still aren't enough, consider heavier measures (see below).
  > **Key: before reaching for the nuclear weapon of "sharding," honestly max out the three moves — indexes, cache, read replicas.** Most projects never reach step four.
- **The second bottleneck is only then the app layer (not enough CPU/memory).**
  Fix: **because the app layer is stateless (Decision 5), just "run more instances + load balance" to scale horizontally** — this is the easiest scaling in the whole architecture, provided you held the stateless discipline from the start.
- **Static assets / bandwidth pressure**: hand it to the CDN (can be added fairly early, high value-for-money).
- **Slow operations dragging on response**: make them asynchronous (task queue); don't block the user's main request path.
- **Finally, only after it's truly very large** does it become time for heavyweight evolution like sharding and splitting services by business domain — and **by then you should have data, a team, and clear bottleneck evidence to back the decision,** never on a hunch ahead of time.

> The practical meaning of this bottleneck order: **it tells you "in what order to spend effort."** Splitting into microservices before the database has even been indexed is skipping the cheapest medicine and jumping straight to the most expensive, most painful surgery.

## 10. Security and compliance essentials

This kind of system isn't sexy, but not one security fundamental can be skipped — and **the overwhelming majority of incidents come from fundamentals done poorly, not from a lack of advanced defenses.**

- **Distinguish authentication from authorization clearly**: authentication (who you are) and authorization (what you can do) are two different things. **The most common vulnerability is "authentication done, authorization not done finely"** — e.g., changing a record ID lets you see/change someone else's data (privilege escalation). Every data access must check "does this user have permission to touch this record?"
- **Never trust user input**: validate and escape all input, blocking injection-class attacks (let data always be just data, never executed as a command/code). This is the oldest and most effective discipline.
- **Protect sensitive data**: never store passwords in plaintext (use a one-way, irreversible method); encrypt in transit and at rest; don't write keys/passwords into code or logs.
- **Session and credential security**: prevent sessions from being stolen/forged; require re-confirmation for sensitive operations.
- **Least privilege**: when the app connects to the database or third parties, grant only the permissions "necessary to do this one thing"; that way, if breached, the blast radius is small.
- **Don't pin security on "no one knows"**: set clear boundaries in the architecture (who can access what), rather than relying on "this endpoint is pretty obscure."
- **Compliance on demand**: when personal data is involved, support deletion, export, and clear notice of purpose — but likewise **do it to the actual scope involved; don't pile up complexity ahead of time for a compliance level you won't use.**

## 11. Common pitfalls / anti-patterns

**This section is the gem of this template — nearly every item is a concrete form of "over-engineering" or "skipping fundamentals."**

- ❌ **Going straight to microservices** → ✅ Monolith first, with clear internal layering; microservices are for "organizational scale," not for "looking advanced" (Decision 2).
- ❌ **Sharding too early** → ✅ Max out the three moves — indexes, cache, read replicas; sharding is the nuclear weapon you reach for last (Section 9).
- ❌ **A stateful app layer, causing session stickiness and inability to scale** → ✅ Stateless app layer, state in external shared storage (Decision 5).
- ❌ **N+1 queries (querying the DB one by one in a loop)** → ✅ Batch query / join in one go; this is this kind of system's most frequent performance killer, often masked by indexes until traffic grows and it blows up.
- ❌ **Adding a cache layer before there's any read pressure** → ✅ A cache trades consistency complexity for performance; with no pressure it's net negative (Decision 3).
- ❌ **Putting heavyweight client-side rendering on a content site** → ✅ Content-type favors server-side rendering (first screen + SEO); don't trade complexity for something you don't need (Decision 1).
- ❌ **Stuffing large files (images/attachments) into the database** → ✅ Put them in object storage; the database stores only their reference; otherwise it bloats the DB, slows queries, and makes backups a nightmare.
- ❌ **Blocking the main request path on slow operations (send email, generate reports)** → ✅ Make them asynchronous via the task queue; the user waits only for what must be synchronous.
- ❌ **Piling up middleware "for possible future scale"** → ✅ Let real requirements force you to add, instead of adding in anticipation (Decision 0).
- ❌ **Authentication done, authorization not done finely (privilege escalation)** → ✅ Check "can this user touch this record?" on every data access (Section 10).

## 12. Evolution path: MVP → growth → maturity

Architecture grows. **Don't fit an MVP to a maturity diagram — but more importantly, beware: most projects actually stay in stage one or two for life and never reach stage three.**

| Stage | User/scale | What the architecture looks like | What to worry about now |
|---|---|---|---|
| **MVP** | Launch ~ tens of thousands | **One monolith + one database**, just that simple. You may not even need a load balancer yet. | **Get features right and draw clear module boundaries**; resist every temptation of "advanced architecture" |
| **Growth** | Tens of thousands ~ fairly large | Still a monolith, but start adding side paths **on demand**: CDN (early), cache (when there's read pressure), read replica (when reads become the bottleneck), task queue (when there are slow operations); run more app instances + load balancing | **Follow the bottleneck order in Section 9 — treat what hurts**; hold statelessness; don't add it all at once |
| **Maturity** | Very large, and the organization grows too | **Only when it genuinely can't cope, and the team's scale also demands parallelism,** split the monolith into a few services along the already-clear module boundaries; do heavier sharding on the database | Cost, disaster recovery, organizational coordination; by now you should have ample data and bottleneck evidence to back each step |

> The golden rule of evolution: **let the monolith carry you until it genuinely can't go on — and "genuinely can't go on" is usually far later than you think.** Splitting is "a result forced out by scale," not "a blueprint planned from the start." And the module boundaries you drew clearly in stage 1 are exactly what makes a future smooth split possible — **so "monolith first" doesn't mean "don't design"; on the contrary, it requires you to think the boundaries through very clearly, only without splitting the deployment apart yet.**

## 13. Reusable takeaways

- 💡 **Simplicity is the most underrated architectural attribute, and one you must actively fight for.** Every component you didn't add is something you don't have to understand, operate, fail at, or be slowed down by. **"Can I get away without adding this?" should be your default question.**
- 💡 **The cost of over-engineering is real and immediate, while the "scalability" it buys is often never used.** First ask "at this scale, right now, do I actually need it?" — this one question can save you the bulk of your trouble.
- 💡 **Spend effort in the real order of the bottlenecks: the database (indexes → cache → read replica) first, horizontal scaling of the app layer second.** Don't skip the cheapest medicine and jump straight to the most expensive surgery.
- 💡 **"Monolith first" isn't because it's backward — it's because it defers "the full suffering of a distributed system" to the day you can genuinely afford it and genuinely need it.** Microservices solve an organizational problem, not a technical one.
- 💡 **Statelessness is a near-zero-cost discipline that clears the biggest obstacle for the future — worth following from day one.** It makes "add machines" the easiest scaling lever.
- 💡 **Let requirements drive architectural evolution, not your anticipation.** Adding when you actually hit the problem is cheap; adding ahead of time for things that haven't happened is expensive.

## 🎯 Quick quiz

<Quiz
  question="For 90% of ordinary projects, the correct architectural starting point is?"
  :options="['Go straight to microservices to look advanced', 'A modular monolith + good enough, evolving when pain forces it', 'You must shard the database first']"
  :answer="1"
  explanation="The overwhelming majority of systems die of over-engineering. A monolith defers 'the full suffering of distributed' to the day you genuinely need it and can afford it."
/>

---

## References & Further Reading

> This template is compiled from the following **classic methodologies** and **official architecture docs**.

**📖 Methodologies / official docs:**
- [The Twelve-Factor App](https://12factor.net/) — the classic methodology for building SaaS apps: 12 principles including config, stateless processes, and horizontal scalability.
- [Cache-Aside Pattern (Azure Architecture Center)](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside) — the classic cache-aside layered caching strategy.
- [Scaling reads with Amazon Aurora (AWS Docs)](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Replication.ReadScaling.html) — read/write splitting / read scaling via a reader endpoint + multiple read replicas.

---

> 📌 Remember the standard web app in one line: **what it teaches isn't "how to make a system complex," but "how to hold back from making it complex." The overwhelming majority of systems die of over-engineering, and restraint is exactly the hardest — and most valuable — architectural skill.**
