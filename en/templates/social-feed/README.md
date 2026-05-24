# Social Feed · Architecture Template

> **Representative products**: Twitter / X, Instagram, Weibo, the TikTok home feed
> **One-line positioning**: take the massive flood of "stuff other people posted," order it by the relationships/interests you care about, and stuff it into that one screen — your "home feed."

---

## 1. One-line positioning

A social-feed product = **a "content distribution engine"**: countless people keep posting things into it, and the moment you refresh, the system has to pick — out of an ocean of content — the few dozen items that are **"relevant to you, and most likely what you'd want to see,"** order them, and deliver them in front of your eyes.

The most counterintuitive thing about it, architecturally: the hard part of this kind of system is **not the write, not the storage, but "how to assemble a personalized list for each person at read time."** A single post is cheap, but "let a hundred million people each see their own version of the home page" is what pushes the whole architecture to its limit. Every design choice answers one question: **is this "personalized list" computed ahead of time when something is posted, or computed on the fly when the user refreshes?**

## 2. The essence of the business: what problem is it solving

What the user wants is **"open the app, don't have to think, and there's an endless stream of content that's relevant to me and pretty good-looking."** It replaces the chore of "going through each followee's profile one by one" — the system **kneads all the sources you follow / find interesting into a single stream you can never scroll to the bottom of.**

Where the value and the money come from:
- **Attention is currency**: the longer a user stays and the more they scroll, the more **ads** you can insert;
- **Feed ads**: an ad is itself "a piece of content disguised as content," blending seamlessly into the stream, billed per impression/click;
- **Ecosystem lock-in**: the deeper the relationship graph and accumulated content, the harder it is for the user to leave.

**Key fact: a post is one "write," but every follower's refresh is a "read" — and reads vastly outnumber writes.** A single post might be read millions of times. This means **almost the entire system load lands on the "read" side**, and this one fact dictates nearly every architectural trade-off downstream — above all that core puzzle: **whether to compute this home feed ahead of time.**

## 3. Core requirements and constraints

**Functional requirements (what the system must be able to do):**
- [ ] Publish content: post text/images/video, possibly @ people, with topics.
- [ ] Follow relationships (the social graph): who follows whom, forming a vast web of relationships.
- [ ] Home timeline: aggregate "people I follow / content I might like" and order it.
- [ ] User timeline: everything a given person has posted, in reverse chronological order.
- [ ] Interactions: like, repost, comment (which themselves become new content/signals).
- [ ] Ranking: decide "which item to show you first" — reverse-chronological, or algorithmic recommendation.
- [ ] Notifications: someone followed you, @-ed you, liked you.

**Non-functional requirements / quality attributes (this is where architecture really fights):**
| Quality attribute | Target | Why it matters for this kind of system |
|---|---|---|
| **Refresh latency (read latency)** | < a few hundred ms | A user pulls to refresh and stares at it; a spinner over 1 second feels laggy. The read path must be blazing fast. |
| **Read throughput (Feed QPS)** | Extremely high | Read volume is hundreds to thousands of times the write volume — the single biggest source of traffic in the system. |
| **Post-to-visible latency** | Seconds is acceptable | A follower refreshing into your post **a few seconds later** is OK; no hard real-time requirement. This is precious "slack." |
| **Availability** | 99.9%+ | If content won't load, the user leaves at once. The read path especially must be highly available. |
| **Eventual consistency is fine** | Yes | Missing one item, or seeing it a few seconds late, does no harm. **You almost never need strong consistency** — an enormous degree of design freedom. |

**Key constraints (boundaries you cannot cross):**
- 🔴 **Reads vastly outnumber writes, and reads are personalized.** You can't "compute once, share with everyone" the way a static web page does — everyone's home feed is different.
- 🔴 **The "celebrity problem"**: follower counts are extremely skewed. An ordinary person has a few hundred followers; a top account has tens of millions. **Any scheme that "does a bit of work per follower at post time" will explode on a celebrity.** This is the architecture's number-one roadblock.
- 🔴 **The social graph is huge with concentrated hotspots**: relationship queries (who follows whom) are frequent, and a celebrity's follower list is itself a super-hotspot.
- The timeline is "infinite scroll," so it needs efficient pagination and history backtracking.

## 4. The big-picture architecture

```
                          User (refresh home feed / post a piece of content)
                            │                    ▲
              post │ write   │                    │  read path │ refresh feed
                            ▼                    │
┌──────────────────────────────────────────────────────────────────────┐
│  Edge layer  API gateway / edge                                        │
│  • auth, throttle  • media upload straight to object storage  • split  │
│                                                       read from write  │
└──────────┬──────────────────────────────────────────────┬─────────────┘
           │ write                                          │ read
           ▼                                                ▼
┌────────────────────┐                          ┌────────────────────────┐
│  Post service       │                          │  Feed read service      │
│  • persist          │                          │  • pull my timeline     │
│    (authoritative)  │                          │    entries              │
│  • fan-out delivery │                          │  • backfill bodies      │
└────┬────────┬──────┘                          │    (in batch)           │
     │        │                                  │  • call ranking, assemble│
     │        │ query follower list              │    final list           │
     │        ▼                                   └──────┬──────────┬───────┘
     │  ┌──────────────┐    ┌───────────────────────┐    │          │
     │  │ Social graph │◀───│ Timeline store (one    │◀──┘          ▼
     │  │ service      │    │ per person): the pre-  │       ┌────────────┐
     │  │ (follows)    │    │ computed "inbox"       │       │ Ranking /  │
     │  └──────────────┘    │ in-memory KV / wide-col│       │ recommend  │
     │         ▲            └───────────▲───────────────┘    │ scoring svc│
     │ fan-out  │ ordinary: push        │ celebrity: pull &  └────────────┘
     └─────────┴───────────────────────┘ merge at read time
     │
     ▼
┌──────────────┐   ┌───────────────────┐   ┌──────────────────────────────┐
│ Content store│   │ Media store + CDN │   │ Notification service           │
│ (post bodies)│   │ (img/video→edge)  │   │ (followed/@-ed/liked → push)   │
└──────────────┘   └───────────────────┘   └──────────────────────────────┘
```

> The soul of this diagram is the middle block, the **"timeline store (one precomputed inbox per person),"** and the two paths flanking it: **fan-out on write / aggregate on read.** The system's core tension hides right here: **ordinary users go "push" (write into followers' inboxes at post time), celebrities go "pull" (merge in at read time), and the two converge in the Feed read service.** That's the "hybrid model."

## 5. Component responsibilities

- **Edge layer / API gateway**: auth, throttling, and **splitting read and write traffic apart** (the read path needs to be optimized to the extreme and heavily cached; the write path needs reliable persistence). Media goes through a separate channel straight to object storage, off the business path. *Why you need it*: read and write are "two different worlds" in this kind of system; split them at the outermost layer so each can be optimized separately downstream.
- **Post service**: receives posts, **writes content into the authoritative store**, then triggers **fan-out** — deciding whether and how this content advances into followers' timelines. *Why you need it*: it's the core of the write path and the executor of the "push vs pull" decision.
- **Social graph service (follow relationships)**: maintains the vast directed graph of "who follows whom," serving "a given person's follower list / followee list" queries. *Why you need it*: fan-out relies on it to fetch the follower list; pulling at read time relies on it to fetch "who I follow." It's the single source of truth for relationships.
- **Timeline store (one precomputed inbox per person)**: maintains, for each user, an **already-ordered list of content IDs** (note: it stores **references/IDs, not bodies**). On refresh, you just read this list. *Why you need it*: it takes the expensive job of "computing each person's home feed on the fly at read time" and does it ahead of time at write time — **trading space for read latency**, which is the key to surviving the massive read QPS.
- **Feed read service**: on refresh, pulls timeline entries, **backfills content bodies in batch**, calls the ranking service to score them, and assembles the final screen to return. *Why you need it*: the timeline holds only IDs, but what the user gets is a complete list with bodies and ranking — this layer does the "assembly."
- **Ranking / recommendation scoring service**: decides the order of items — pure reverse-chronological, or scored by "how likely you are to interact." *Why you need it*: content supply far exceeds what a user can consume, so **ranking quality directly determines dwell time** — which directly determines revenue.
- **Content store (post bodies)**: stores the post itself (text, referenced media addresses, author, time). Written once, read en masse, basically never changed.
- **Media store + CDN**: images/videos in object storage, distributed via CDN edge nodes. *Why you need it*: media is large and repeatedly read, so it must rely on a CDN to block traffic from the origin and pull latency right next to the user (see the video-streaming template).
- **Notification service**: when you're followed, @-ed, or liked, asynchronously generate a notification and push it. *Why you need it*: interaction feedback is the core of social stickiness, but it must not slow down the main path of posting / reading the feed — hence async.

## 6. Key data flows

**Scenario 1: An ordinary user posts something (fan-out on write / push model)**
```
1. User posts ──▶ gateway ──▶ post service
2. Post service: write the body into [content store] (this is the authoritative copy)   ← always done
3. Post service ──▶ social graph service: fetch "my follower list" (say, 800 people)
4. Async fan-out: write this "content ID" into each of the 800 followers' [timeline inboxes]
        Follower A inbox: [new ID, old, old, ...]
        Follower B inbox: [new ID, old, old, ...]
        ... (800 writes)
5. Done. Next time a follower refreshes, this post is already sitting in their inbox.
```
> This is called **fan-out on write**: do more work at post time (write 800 copies) so reads become trivial (just read your own single inbox). **For an ordinary person without many followers, this is a great deal** — because reads vastly outnumber writes, spending a little more on the write side saves the massive cost on the read side.

**Scenario 2: A user refreshes the home feed (the read path)**
```
1. User pulls to refresh ──▶ gateway ──▶ Feed read service
2. Read service: pull the latest N IDs from my [timeline inbox] (ordinary people's content is pre-stored here)
3. [The hybrid key] then query what the "celebrities" I follow recently posted ── pull in real time and merge
        (celebrities' content was never pushed to me, so it's pulled on the fly at this moment)
4. Batch backfill: take this batch of IDs to [content store] for bodies + to media/CDN for images
5. ──▶ ranking service: score and order this batch of candidates (chronological or algorithmic recommendation)
6. Assemble into a screen ──▶ return to the user
```
> Note steps 2 and 3: **ordinary people's content is "pushed ahead of time, read directly," while celebrities' content is "pulled on the fly at read time, merged in temporarily."** A single refresh uses both "push" and "pull" at once — that's exactly the essence of the hybrid model.

**Scenario 3: A celebrity posts something (why you can't fan out on write)**
```
A celebrity (50M followers) posts ──▶ post service
   If you fan out via the "push model" ──▶ you'd have to write once into each of 50M inboxes!!
        → one post = 50M writes, instantly blowing up storage and queues (this is "fan-out explosion")
   ✅ Hybrid solution: the celebrity does NOT fan out, only writes their own [user timeline]
        → followers actively pull and merge it in at "refresh read time" (step 3 of Scenario 2)
```
> This is the most important diagram in this architecture: **the "celebrity problem" is cracked with "divide push and pull" — ordinary people push, celebrities pull.** Carve out "the tiny minority that would explode" to use the pull model, and let the vast majority use the push model.

## 7. Data model and storage choices

Core entities: `User` ─(follows)─▶ `User` (forming the **social graph**); `User` ──posts──▶ `Post`; `Post` ──produces──▶ `Interactions (like/repost/comment)`; `User` ── owns ──▶ `Timeline (a string of post IDs)`.

| Data | Storage type | Why |
|---|---|---|
| User account / profile | Relational | Needs transactions and strong consistency (account, security-related) |
| Follow relationships (social graph) | Graph / wide-column (adjacency list) | Frequently queries "a person's follower/followee list," fundamentally graph adjacency; follower lists can be extremely long |
| Post bodies | Document / KV (fetched by ID) | Written once, read en masse, basically never changed; fastest fetched directly by post ID |
| Timeline inbox (one ID-list per person) | In-memory KV / wide-column | Core of the read path, needs ultra-low latency, read directly by user ID; stores only IDs not bodies, saving space |
| Media (images/video) | Object storage + CDN | Large files, immutable, repeatedly read; must be edge-distributed |
| Interaction counts (likes/reposts) | In-memory counter / KV | Written and read extremely frequently, needs fast increment/decrement, can tolerate eventual consistency |
| Features / signals for ranking | Columnar / feature store | Massive behavioral logs, fed to ranking models for offline/near-line computation |

> Teaching point: **the timeline inbox stores only "content IDs," not bodies.** One viral post is referenced by millions of inboxes; if each stored a full copy of the body, space would explode in an instant — store IDs, then batch-backfill the bodies at read time. This is the classic "reference vs copy" trade-off. **We describe the timeline store with "in-memory KV" because its access pattern is "read one ordered list at ultra-low latency by user ID," not because of any specific product.**

## 8. Key architectural decisions and trade-offs ⭐

**Decision 1: Feed via push model (fan-out on write), pull model (fan-out on read), or hybrid? (The Achilles' heel of this architecture.)**
- **Pull model (fan-out on read)**: at post time, store only your own; on refresh, **go query "everyone I follow" for what they recently posted, and aggregate and rank it on the spot.**
  - Pros: writes are trivially light (write one copy), no fan-out at all, simple to implement, follow-changes take effect immediately.
  - Cons: **reads are brutally heavy.** Follow 2,000 people and every refresh has to aggregate the latest content from 2,000 people — severe **read amplification**, high refresh latency — and reads are the highest-frequency operation, so it can't take high QPS.
- **Push model (fan-out on write)**: at post time, **push the content ID into all followers' inboxes**; on refresh, read your own single inbox — blazing fast.
  - Pros: **reads are trivially light** (read one pre-ranked list directly), a perfect match for "reads vastly outnumber writes."
  - Cons: **writes fan out**; hit a celebrity (tens of millions of followers) and you get **fan-out explosion** — one post means tens of millions of writes, blowing up the write path and storage.
- **Hybrid**: **the vast majority of ordinary users go push** (few followers, cheap fan-out, enjoy blazing-fast reads); **the few celebrities go pull** (too many followers, so don't push — let followers merge it in at read time).
- **Leaning**: **at scale you inevitably go hybrid.** Because reads vastly outnumber writes → you default to wanting push; but celebrities make pure push explode → carve celebrities out to use pull. **"Optimize for the 99% cheap case, and handle the 1% explosive case separately"** is the universal solution for these skewed-distribution systems. The cost is that **two paths exist in the system at once**, and the read service has to merge "what was pushed + what was pulled," raising complexity.

**Decision 2: How exactly do you crack "the celebrity fan-out explosion"?**
- The problem: follower counts are **extremely long-tailed** — the vast majority of people have very few followers, a tiny minority have tens of millions. One celebrity post, if pushed to all followers, produces tens of millions of writes in an instant, overwhelming queues and storage, and also slowing down the celebrity's own posting response.
- The fix: **set a "follower-count threshold" per account.** Below the threshold = ordinary person = normal fan-out (push) at post time; above the threshold = celebrity = **no fan-out** — the content stays only in their own user timeline, and followers actively **pull and merge** it into their home feed **at read time.**
- **Leaning**: this is how the hybrid model is implemented. The cost: ① the read service gets more complex (it has to identify "which celebrities I follow" and pull-and-merge in real time); ② the threshold needs tuning, and there's a "mid-tier" gray zone; ③ celebrity content's read path is heavier and needs **extra caching** (because it gets pulled by huge numbers of followers at once — a natural hotspot). **In essence, you move "explosion at write time" into "one extra merge at read time" — and at read time we have caching available.**

**Decision 3: Where does the timeline (inbox) live, and how deep does it go?**
- Store it all in a slow persistent store: saves memory, but read latency is high, violating "reads must be blazing fast."
- Put it all, permanently, in fast storage: reads are blazing fast, but massive users × long history blows up memory cost.
- **Leaning**: put the timeline inbox in **in-memory KV / wide-column** to guarantee read speed, but **keep only a limited recent count** (say the last few hundred items). When a user scrolls down to very old content, fall through to a "degraded path" that does a slow query against persistent storage. **Because "the vast majority of refreshes only look at the latest screen," keeping a full fast copy for cold deep-scrolls isn't worth it.** Again, "optimize for the hot path, degrade the cold path."

**Decision 4: Ranking — pure reverse-chronological, or algorithmic recommendation?**
- **Reverse-chronological**: newest on top. Simple, predictable, what-you-follow-is-what-you-see. But once content piles up it drowns the good stuff, and the "chatterbox" you follow floods your screen.
- **Algorithmic recommendation**: rank by a "how likely you are to dwell/interact" score, and can mix in "content you don't follow but might like."
  - Pros: dwell time ↑, revenue ↑, can break out of the relationship chain to surface good content.
  - Cons: needs a whole near-line/online pipeline of **behavioral-signal collection + features + scoring models**, with heavy compute cost; and it brings product and ethical issues like "filter bubbles / lack of explainability."
- **Leaning**: use chronological early on (simple, good enough); **at maturity, near-universally shift to algorithmic ranking**, because it directly drives the core business metrics. Architecturally this means hanging a **ranking subsystem** behind the read service, with the ranking features continuously computed and cached. **Key: ranking happens by "scoring a small batch of candidates (a few hundred) at read time," not by "recomputing the whole network for everyone in real time"** — the latter is a compute-impossible misconception.

**Decision 5: How real-time does post-to-visible need to be?**
- Chase "the moment you post, followers can see it": fan-out must complete synchronously, slowing the posting response and magnifying the explosion problem.
- Accept "second-level latency": **make fan-out async** (enqueue and write slowly), and the post service returns fast.
- **Leaning**: **almost always accept second-level eventual consistency.** A social feed isn't a financial transaction; refreshing into a post a few seconds late is perfectly fine. **This "slack" is the most precious design freedom in this kind of system** — it lets fan-out be async, peak-shaveable, and retryable on failure, without blocking the user.

## 9. Scaling and bottlenecks

Unlike an ordinary system: **here the bottlenecks concentrate on "fan-out/aggregation on the read side" and "celebrity hotspots," not plain database capacity.**

- **First bottleneck: enormous Feed read QPS.** Reads are hundreds to thousands of times the writes, and every refresh has to assemble a personalized list.
  Fixes: ① **precompute timelines** (push model — turn "compute on the fly at read time" into "fetch directly at read time"); ② **multi-tier caching** — hot content bodies, hot users' timelines, and full rendered screens can all be cached; ③ timelines store IDs only + batch backfill, shrinking the data volume.
- **Second bottleneck: celebrity fan-out explosion.** One post must reach tens of millions of inboxes.
  Fix: **push/pull hybrid** — celebrities don't fan out, switching to pull-and-merge at read time (Decisions 1, 2). This is the unavoidable core technique.
- **Third bottleneck: social-graph queries and hotspots.** A celebrity's follower list is itself a super-hotspot, frequently read for fan-out/pull.
  Fixes: ① shard the adjacency list; ② cache huge follower lists; ③ give super-hotspot accounts' content a dedicated **hotspot cache layer** (because it gets pulled by huge numbers of followers at once).
- **Fourth bottleneck: ranking compute.** Algorithmic recommendation has to score candidates for every refresh.
  Fixes: ① **precompute features offline/near-line**, doing only lightweight scoring online; ② **coarse-rank/recall** the candidate set down to a few hundred first, then **fine-rank**; ③ briefly cache ranking results.
- **Media bandwidth**: image/video traffic is enormous → CDN edge distribution + tiered caching (see the video-streaming template).

## 10. Security and compliance essentials

- **Content moderation (the number-one risk)**: this kind of system is a natural breeding ground for **illegal/harmful/false information.** The publishing path must connect to **moderation** (automated + human), moderating before fan-out or moderating alongside fan-out, and violating content must be **quickly recalled from all inboxes.**
- **Privacy and visibility**: private accounts, mute/block, who can see my content — these **visibility rules must be enforced on both the "fan-out" side and the "read" side**, not merely hidden in the front end. A common bug: the content has already been pushed into the inbox of someone who shouldn't see it.
- **Abuse and spam**: bot follower-padding, engagement-padding, bulk spam content. You need throttling, behavioral risk control, and fake-account detection.
- **The relationship chain is sensitive data**: "who follows whom, who liked whom" exposes social relationships, requiring access control and minimal exposure.
- **Ranking and manipulation**: a recommendation system can be manipulated by engagement-padding, or weaponized for improper content distribution — it needs observability and a human backstop.

## 11. Common pitfalls / anti-patterns

- ❌ **Pure push model, ignoring celebrities** → ✅ the moment you hit a celebrity (tens of millions of followers) you get **fan-out explosion.** Celebrities must use the pull model (push/pull hybrid).
- ❌ **Pure pull model, expecting to take high read QPS** → ✅ every refresh aggregates thousands of followees — severe **read amplification**, slow refresh. Ordinary users should go through a precomputed push model.
- ❌ **Recomputing the whole network's ranking for everyone in real time on every refresh** → ✅ compute-impossible. **Precompute timelines + score only a small batch of candidates.**
- ❌ **Storing bodies in the timeline inbox** → ✅ one viral post referenced by millions of inboxes blows up space. **Store only content IDs, and batch-backfill bodies at read time.**
- ❌ **Completing fan-out synchronously, blocking the posting response** → ✅ fan-out should be **async**, with posting returning fast; a social feed accepts second-level eventual consistency.
- ❌ **Enforcing visibility/blocking only in the front end** → ✅ it must be enforced on both the fan-out and read sides, or content leaks into the inbox of someone who shouldn't see it.
- ❌ **Serving media straight from the origin** → ✅ images/video must go through CDN edge distribution, or both bandwidth and latency collapse.

## 12. Evolution path: MVP → growth → maturity

| Stage | User/scale magnitude | What the architecture looks like | What to worry about now |
|---|---|---|---|
| **MVP** | Validate the idea / tens of thousands of users | **Pull model**: posts store only your own, and refresh queries followees on the fly to aggregate and rank; a single database; reverse-chronological; media served simply and directly or via a small CDN | **First validate whether anyone posts and anyone reads.** The pull model is the simplest to implement, and read amplification isn't yet fatal while nobody has many followers |
| **Growth** | Millions | Introduce a **push model + precomputed timelines** to take read QPS; add **multi-tier caching**; put media on a CDN; make fan-out async; the first celebrities start to appear | Find the read-side bottleneck and beat refresh latency down; keep an eye on the first batch of celebrities, prepare for push/pull hybrid |
| **Maturity** | Tens of millions ~ hundreds of millions | **Push/pull hybrid** (ordinary push, celebrity pull) + **algorithmic ranking** (recall, coarse-rank, fine-rank) + **multi-tier/hotspot caching** + global multi-region + full content moderation and risk control | Celebrity hotspots, ranking compute and quality, cost, disaster recovery, content compliance and information security |

## 13. Reusable takeaways

- 💡 **When reads and writes are asymmetric, move the cost to the "fewer-times" side.** Reads vastly outnumber writes → do more at write time (fan-out, precompute), making reads trivial. This is the general idea behind "fan-out on write," and it applies to any "write-light, read-heavy" system.
- 💡 **Skewed distributions (long tails) call for divide-and-conquer: optimize for the cheap majority, handle the explosive minority separately.** "Ordinary push, celebrity pull" is the embodiment of this wisdom; it equally applies to hot products, hot rooms, super-users, and every "tiny minority of extreme values" scenario.
- 💡 **Store a "reference," not a "copy."** The timeline stores only content IDs, backfilling at read time — avoiding a copy of one viral piece of content being duplicated a million times. This is the general move for deduplication and saving space.
- 💡 **Precomputation is a powerful lever for trading space for read latency, but precompute only for the hot path and degrade the cold path.** Keeping only the latest N items in the timeline and slow-querying for deep scrolls is "don't pay full price for the cold case."
- 💡 **Find your system's "slack dimension" and use it to the hilt.** Here it's the eventual consistency of "post-to-visible can be a few seconds slow" — and it's exactly what lets fan-out be async, peak-shaveable, and retryable. Identifying where "less real-time / less strongly consistent" is allowed often unlocks the elasticity of the whole architecture.

## 🎯 Quick quiz

<Quiz
  question="When a super-celebrity with tens of millions of followers posts, the most suitable way to distribute it is?"
  :options="['Push to all followers at write time (fan-out on write)', 'Pull at read time / push-pull hybrid, to avoid one push blowing up the system', 'Handle it exactly like an ordinary user']"
  :answer="1"
  explanation="Ordinary people use push, celebrities use pull (push-pull hybrid) — the classic divide-and-conquer for «long-tailed skewed distributions», avoiding one fan-out blowing up the system."
/>

---

## References & Further Reading

> This template is compiled from the following **engineering deep-dives** and **real open-source projects**.

**📖 Engineering blogs:**
- [The Architecture Twitter Uses to Deal with 150M Active Users (High Scalability)](https://highscalability.com/the-architecture-twitter-uses-to-deal-with-150m-active-users/) — fan-out on write pushes tweets into followers' timeline caches.
- [Scaling Instagram's recommendation system (Meta Engineering)](https://engineering.fb.com/2025/05/21/production-engineering/journey-to-1000-models-scaling-instagrams-recommendation-system/) — the feed recall / ranking funnel at massive scale.

**🔧 Open-source prototypes (read the code directly):**
- [mastodon/mastodon](https://github.com/mastodon/mastodon) — an open-source microblogging social network; readable source for the follow graph and home-timeline delivery implementation.

---

> 📌 Remember a social feed in one line: **it isn't "a database of posts" — it's "a distribution engine that assembles a personalized list for each person on the spot." The core tension is always "compute this home feed ahead of time (push), or compute it on the fly at refresh (pull)," and the answer is completely different for ordinary people and celebrities.**
