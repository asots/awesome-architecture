# Video Streaming · Architecture Template

> **Representative products**: Netflix, YouTube, all manner of on-demand / live platforms
> **One-line positioning**: deliver big, heavy video files to any screen on the planet with the least stalling and the least bandwidth.

---

## 1. One-line positioning

A video-streaming product = **a "content logistics network"**: take a multi-GB source video, process it into versions that fit thousands of networks and screens, stock it ahead of time at the "forward warehouse" nearest the user, and when the user hits play, serve it from nearby, on demand, segment by segment.

The most counterintuitive thing about it, architecturally: its biggest difference from an ordinary website isn't logic complexity — it's that **the most expensive, scarcest resource is "bandwidth,"** and the heaviest one-time investment is **"transcoding compute."** Almost the entire architecture answers two things: **how to turn one video into multiple versions "everyone can watch smoothly" (transcoding)**, and **how to "stock those versions nearest the user, while hitting the origin as little as possible" (CDN distribution).**

## 2. The essence of the business: what problem is it solving

What the user wants is **"hits play and it goes, no spinner, crisp, and scrubbable."** It replaces the old era of "download the whole file before you can watch" — the system lets you **watch as it downloads (streaming)**, and whether your network is good or bad, on a phone or a TV, it tries to give you "the best quality your current network can handle."

Where the money comes from:
- **Subscriptions** (monthly viewing, which demands a stable, smooth experience);
- **Ads** (free viewing + pre-roll/mid-roll ads billed per impression);
- The shared precondition for both is "**smooth playback**" — stalling is the number-one killer of experience and retention.

**Key fact: every extra minute someone watches consumes a real, tangible chunk of bandwidth.** On an ordinary website "one more request costs almost nothing," but here "streaming one more HD movie is a real bandwidth bill." And **bandwidth is this business's single biggest cost line** — this one fact dictates why CDN, cache hit rate, and bitrate tiers become the core topics of the architecture.

## 3. Core requirements and constraints

**Functional requirements (what the system must be able to do):**
- [ ] Upload / ingest: creators or studios upload the source video.
- [ ] Transcoding: turn one source file into **multiple bitrate / resolution** versions.
- [ ] Streaming playback: watch as it downloads, rather than downloading first.
- [ ] Adaptive bitrate (ABR): switch quality in real time with network speed — go 4K when the network's good, drop to SD when it's bad, **prioritizing no-stall.**
- [ ] Seek / fast-forward: jump to any timestamp and start playing.
- [ ] Recommendations: help users find what they want in a massive library.
- [ ] Metadata: title, cover, subtitles, duration, the list of available bitrates, etc.
- [ ] (Optional) Live: record-transcode-stream as it happens, latency-sensitive.

**Non-functional requirements / quality attributes (this is where architecture really fights):**
| Quality attribute | Target | Why it matters for this kind of system |
|---|---|---|
| **Startup time / first-frame latency** | < 1–2 s | If the spinner spins too long after hitting play, the user bails. The faster the first frame, the better. |
| **Rebuffering rate** | As low as possible, near 0 | A spinner mid-playback is the number-one killer of experience, directly affecting retention. |
| **Bandwidth cost / per GB delivered** | As low as possible | Bandwidth is the biggest cost line; margin lives or dies by it. This is the biggest difference from ordinary systems. |
| **CDN hit rate** | As high as possible | Hitting the edge cache = cheap and fast; falling back to origin = expensive and slow. Hit rate directly determines cost. |
| **Availability** | 99.9%+ | If it won't play, the user leaves at once. |
| **Picture quality** | As high as possible without stalling | Key to experience, but it must be balanced against bandwidth cost. |

**Key constraints (boundaries you cannot cross):**
- 🔴 **Bandwidth is both expensive and the bulk of the cost.** This is the number-one constraint; every effort on CDN and cache hit rate serves it.
- 🔴 **Video files are huge.** A source film is several GB, and transcoded versions multiply that — storage and distribution both strain.
- 🔴 **Transcoding is heavy compute, a slow operation.** Transcoding one long video into dozens of versions consumes massive CPU/GPU time, and must never sit in the user's request path.
- 🔴 **User networks vary wildly.** The same video has to play as smoothly as possible on gigabit fiber and on a weak signal in the subway — which forces multi-bitrate + adaptive.
- Live adds a constraint: **extremely latency-sensitive**, you can't leisurely transcode and stock the way on-demand can.

## 4. The big-picture architecture

```
   Creator/studio                                                Viewers (all kinds of screens/networks)
      │ upload source (several GB)                                          ▲
      ▼                                                                     │ request segments by current bandwidth
┌──────────────┐                                                            │
│ Upload/ingest│                                                            │
│ service      │                                                            │
│ • receive raw│                                                            │
└──────┬───────┘                                                            │
       │ put into source storage + dispatch transcode task                  │
       ▼                                                                     │
┌─────────────────────────────────────────────┐                            │
│  Transcode pipeline (async, heavy compute,    │                            │
│  elastically scalable)                        │                            │
│  ┌──────────┐   one source                    │                            │
│  │ Transcode│   split into small segments      │                            │
│  │ task     │   each segment transcoded in      │                           │
│  │ queue    │   parallel into multiple tiers:   │                           │
│  └────┬─────┘     1080p / 720p / 480p / 240p    │                           │
│       ▼           + generate a "manifest"        │                          │
│  ┌──────────────┐                                │                          │
│  │ Elastic      │ (auto-scale by queue length)   │                          │
│  │ transcode    │                                │                          │
│  │ worker pool  │                                │                          │
│  └──────┬───────┘                                │                          │
└─────────┼──────────────────────────────────────┘                          │
          │ write                                                            │
          ▼                                                                   │
┌────────────────────────┐      warm-up / on-demand origin pull               │
│  Object storage         │ ─────────────────────────┐                       │
│  (authoritative master) │                          ▼                       │
│  • segments of all tiers│              ┌──────────────────────────────┐    │
│  • manifest files       │              │   CDN (multi-tier edge nodes, │────┘
└────────────────────────┘              │   global footprint)           │
                                          │   • cache hot segments, serve │
┌────────────────────────┐               │     from nearby               │
│ Metadata svc / recommend│ ──title/────▶│   • fall back to origin only  │
│ (title/cover/bitrates)  │   manifest    │     on miss                   │
└────────────────────────┘               │   ◀═ most traffic absorbed here═
                                          └──────────────────────────────┘
                         Player (ABR adaptive bitrate):
                         get manifest → estimate current bandwidth → request the next
                         segment at the "right tier"; network better → switch higher;
                         network worse → switch lower; the goal is "never stall"
```

> The soul of this diagram is two blocks: the **"transcode pipeline"** (turn one source into multiple versions everyone can watch — heavy compute, must be async) and the **"CDN + object storage"** (stock content next to the user and beat bandwidth cost down). Everything else (metadata, recommendations) serves the goal of "letting this logistics network deliver video fast and cheap."

## 5. Component responsibilities

- **Upload / ingest service**: receives the source video, stores it in the "master" area of object storage, and **dispatches a transcode task**. *Why you need it*: the source is the starting point of everything; it decouples "upload" from "the heavy reprocessing afterward" — upload returns immediately, transcoding takes its time.
- **Transcode pipeline**: **splits the source into small segments**, **transcodes each segment in parallel into multiple bitrate/resolution tiers**, and generates a **manifest** describing "which tiers exist, and where each tier's segments are." *Why you need it*: user networks and screens vary wildly — one version can't serve all; and segmenting + multi-tier is the physical basis for "adaptive bitrate" and "watch as it downloads." It must be **async and elastically scalable**, because transcoding is extremely heavy.
- **Object storage (authoritative master / all outputs)**: stores the source and **all bitrate segments + manifests.** Written once, read en masse, immutable. *Why you need it*: this is the single authoritative copy of the content, and the origin the CDN pulls from on a miss.
- **CDN (edge distribution network)**: caches hot segments at the **edge node nearest the user**, responding from nearby; falls back to origin object storage only on a miss. *Why you need it*: **this is the key to beating down bandwidth cost and latency at the same time** — letting the vast majority of traffic be absorbed at the edge, fast and cheap, while protecting the origin.
- **Player (client-side ABR logic)**: gets the manifest first, **estimates the current bandwidth in real time**, and decides which bitrate tier to request for the next segment; switches quality up when the network's good, down when it worsens, with the **primary goal of not stalling.** *Why you need it*: the network fluctuates in real time, and only the client can sense the true bandwidth right now and adjust instantly. **The "brain" of adaptation lives on the player side.**
- **Metadata service**: stores "information about the video" — title, cover, duration, subtitles, the list of available bitrates, etc. (note: not the video itself). *Why you need it*: browsing, search, and the pre-play info display all rely on it; it's lightweight, high-frequency reads, kept separate from the heavyweight video byte stream.
- **Recommendation service**: picks content for the user from a massive library. *Why you need it*: the bigger the library, the harder it is to "find what you want," and recommendations directly drive watch time and retention.

## 6. Key data flows

**Scenario 1: A video from upload to watchable (the write / processing path)**
```
1. Creator uploads source (several GB) ──▶ upload service ──▶ store in [object storage · master area]
2. Upload service dispatches a [transcode task] to the queue, then returns immediately ("upload OK, processing")
3. Transcode pipeline asynchronously picks up the task:
       a. Split the source into many small segments (say 4–10 seconds each)
       b. Transcode each segment in parallel into multiple tiers: 1080p / 720p / 480p / 240p ...
       c. Generate the [manifest]: lists "which tiers exist, and the address of each segment of each tier"
4. Write all outputs (segments + manifest) back to [object storage]
5. (Optional) [Warm up] hot content's segments by pushing them to CDN edge nodes
6. Metadata service marks the video "playable," the available-bitrate list ready
```
> Note step 2: **upload returns immediately, and transcoding runs slowly in the background.** Transcoding is heavy compute and must never make the user wait there for tens of minutes — this is the textbook case of "make the heavy job async."

**Scenario 2: A viewer hits play (the read / distribution path — the most core)**
```
1. User hits play ──▶ metadata service: get the [manifest] (which bitrates, where the segments are)
2. Player estimates current bandwidth (say 5 Mbps right now) ──▶ decides to request 720p first
3. Player requests "segment 1 of 720p" ──▶ the nearest [CDN edge node]
       hit → edge returns directly (fast, cheap)        ← the vast majority of cases
       miss → CDN pulls once from origin [object storage], caches it along the way, then returns (slow, expensive)
4. As soon as segment 1 arrives, playback starts (watch as it downloads), while prefetching subsequent segments in the background
5. Network improves (rises to 20 Mbps) → request 1080p for the next segment; worsens → drop to 480p
       ⟲ re-decide every few seconds, every segment — goal: quality as high as possible AND never stall
```
> Note step 3: **an edge hit is fast and cheap, an origin pull is slow and expensive.** The entire distribution economics rests on "CDN hit rate." Note step 5: **quality is switched dynamically "per segment,"** not fixed for the whole film — that's adaptive bitrate (ABR).

**Scenario 3: Live (why it's different from on-demand)**
```
Streamer pushes ──▶ real-time ingest ──▶ segment-and-transcode as it arrives (must be fast, no leisurely transcoding)
        ──▶ push to CDN edge immediately ──▶ viewers pull and watch with only a few seconds' latency
   Key difference: on-demand can "transcode at leisure first, then distribute"; live is "transcode and distribute as it's
                  produced." The transcode window is tiny and extremely latency-sensitive — the architecture must be
                  redesigned for "real-time."
```
> Live compresses on-demand's "process first, distribute later" into "process and distribute simultaneously"; the latency constraint is completely different, usually requiring a **dedicated low-latency path.**

## 7. Data model and storage choices

Core entities: `Video (source)` ──transcode──▶ `Multi-tier segments` + `Manifest`; `Video` ── related to ──▶ `Metadata (title/cover/subtitles)`; `User` ──produces──▶ `Watch behavior (for recommendations)`.

| Data | Storage type | Why |
|---|---|---|
| Source (master) | Object storage | Huge files, immutable, written once, read occasionally (origin pull/re-transcode) |
| Transcode outputs (per-tier segments + manifest) | Object storage + CDN | Huge, immutable; **read en masse → must rely on CDN edge distribution** |
| Video metadata (title/cover/available bitrates) | Relational / document | Lightweight, high-frequency reads, must support browse and search, kept separate from the video byte stream |
| Transcode task state | Queue + state store | Async tasks needing queuing, retry, progress tracking |
| Watch behavior / playback-quality logs | Columnar / time-series | Massive, aggregated by time, fed to recommendations and quality monitoring (rebuffering rate, etc.) |
| Users / subscriptions | Relational | Needs transactions and strong consistency (billing-related) |

> Teaching point: **"the video itself" and "information about the video" are two completely different kinds of data and must be stored separately.** Video bytes are object-storage material — "huge, immutable, CDN-distributed"; metadata is database material — "lightweight, high-frequency reads, searchable." Mix them and the browse page gets dragged down by enormous video bytes. **We describe video storage with "object storage + CDN" because its access pattern is "immutable large files + massive reads from nearby," independent of any specific product.**

## 8. Key architectural decisions and trade-offs ⭐

**Decision 1: Transcoding — synchronous or asynchronous? (Hardly a contest, but understand why.)**
- **Synchronous**: transcode on the spot after upload, telling the user "success" only when done.
  - Pros: straightforward logic.
  - Cons: transcoding one long video takes minutes to tens of minutes, **trapping the user inside the upload request**; the compute it consumes also contends with online requests for resources, so it crashes the moment traffic arrives.
- **Asynchronous**: upload returns "processing" immediately, the transcode task goes into a queue, and a background elastic worker pool digests it slowly.
  - Pros: smooth user experience; transcode compute can **scale elastically** independently; failures can be retried; peak-shaveable.
  - Cons: there's a "upload-to-playable" latency; you must maintain a task queue, state tracking, retry, and dedup.
- **Leaning**: **must be async.** This is the iron law of "move heavy compute off the request path" — any operation far exceeding the user's patience should be made async.

**Decision 2: How many bitrate tiers do you store? (Cost vs experience.)**
- Too few tiers (say only HD + SD): saves transcode compute and storage, but **can't fit all kinds of networks** — weak-network users either stall or can only watch something very fuzzy.
- Too many tiers (a dozen-plus, from 4K to ultra-low): excellent fit, smooth adaptation, but **transcode compute doubles, storage doubles** (every extra tier means one more copy of all segments stored).
- **Leaning**: **pick a limited set of tiers covering mainstream networks and screens** (typically a handful to a dozen-plus), and you can prepare more tiers for hot content and fewer for cold. The cost is a continuous trade-off between "fit/quality" and "transcode + storage cost." **This is a pure cost-vs-experience balancing act with no standard answer — it depends on your users' network distribution.**

**Decision 3: CDN strategy — warm up the hot content, or pull from origin on demand across the board?**
- **On demand across the board (passive)**: the first user to request a given segment triggers an origin pull, after which it's cached at the edge.
  - Pros: simple, only caches what's actually watched.
  - Cons: **the "first viewer" at every edge node has to wait one origin pull** (slow); the moment a big hit launches, huge numbers of misses pull from origin simultaneously — a **thundering herd hammering the origin.**
- **Warm up the hot content (active)**: for content predicted/known to blow up (a new show's premiere, a top creator), **push the segments to all edge nodes in advance.**
  - Pros: a hit from the moment of play, fast startup; eliminates the thundering herd.
  - Cons: you must predict popularity, and warming up something nobody watches wastes edge storage and push bandwidth.
- **Leaning**: **pull the cold long tail from origin on demand, actively warm up the top hot content** — combine the two. **"Stock the predictable hotspots ahead, fetch the unpredictable long tail on demand"** — exactly the logistics warehousing mindset.

**Decision 4: On-demand vs live — do you need two architectures?**
- On-demand (VOD): the content already exists, so you can **transcode it at leisure first, then slowly stock the CDN**; the optimization focus is hit rate and cost.
- Live: the content is **being produced right now**, so you must segment-transcode-distribute as it arrives; the optimization focus is **end-to-end latency**, with a tiny transcode window.
- **Leaning**: the two **usually need different paths.** Live sacrifices some transcode refinement and caching strategy for low latency; on-demand processes at leisure for cost and quality. Forcing the on-demand architecture onto live fails because latency won't meet the bar. **"Latency-sensitive" and "cost-sensitive" are two different design orientations — don't force one architecture onto two constraints.**

**Decision 5: The adaptive-bitrate (ABR) decision authority — server side or client side?**
- The server decides which tier to push: the server would have to sense each client's real-time network, which is nearly unrealistic.
- **The client (player) decides**: the player knows best "how fast this network is right now, how many seconds of buffer remain," picking the bitrate segment by segment.
- **Leaning**: **the decision authority is on the client.** The server is only responsible for "having all tiers ready and writing the manifest clearly," while which tier each segment picks is decided by the player in real time. **Put the decision at "the end with the most complete information"** — only the client can sense the local network in real time.

## 9. Scaling and bottlenecks

Unlike an ordinary system: **here the bottlenecks are "transcode compute" and "CDN bandwidth/hit rate," not the business database.**

- **First bottleneck: transcode compute.** During an upload peak or bulk ingest, transcode tasks pile up and the "playable" latency stretches out.
  Fixes: ① **async queue peak-shaving**; ② **elastically scale** the transcode worker pool (scale out when the queue is long, scale in when idle); ③ **transcode segments in parallel** (split one video into segments processed in parallel); ④ transcode fewer tiers for cold content.
- **Second bottleneck: CDN bandwidth and hit rate (the cost core).** Bandwidth is the biggest cost line, and every point the hit rate drops sends origin-pull cost and latency soaring.
  Fixes: ① **tiered caching** (edge → regional mid-tier → origin, intercepting origin pulls layer by layer); ② **warm up hot content**; ③ improve segment cacheability (immutable segments are naturally cache-friendly).
- **Third bottleneck: the thundering herd on hot content.** The moment a hit launches, huge numbers of users request the same batch of segments at once; if the edge misses, they'll all pull from origin and **crush it.**
  Fixes: ① warm-up; ② **coalesce/dedup origin-pull requests** (concurrent origin pulls for the same segment fire only once); ③ tiered caching to absorb the shock.
- **Storage growth**: tiers × content volume → storage bloat. Fix: downgrade tiers / archive cold content to a cheaper storage tier.

## 10. Security and compliance essentials

- **Content rights (DRM) and hotlink protection**: paid content needs rights protection and playback authorization, to prevent segments being scraped directly and to prevent hotlinking from consuming your bandwidth. This is the bottom line for a paid-video business.
- **Pirated uploads and content moderation**: user-upload platforms must detect pirated/violating content (moderate on upload) and be able to take it down quickly.
- **Bandwidth theft**: someone hotlinking your CDN resources = spending your money. You need source validation, signed URLs, and expiry control.
- **Geographic compliance (rights geo-restriction)**: much content is only licensed in specific regions, so distribution must apply geo-based access control.
- **Privacy**: watch history is sensitive behavioral data, requiring protection and minimal exposure.

## 11. Common pitfalls / anti-patterns

- ❌ **No transcoding, serving the source directly** → ✅ the source is too big to play for weak-network/small-screen users, and can't adapt. **It must be transcoded into multiple tiers.**
- ❌ **No CDN, serving straight from the origin** → ✅ bandwidth cost explodes, latency is high, the origin gets crushed. **Video distribution must go through the CDN edge.**
- ❌ **Synchronous transcoding, blocking the upload request** → ✅ transcoding is heavy compute and **must be async** (queue + elastic worker pool), with upload returning immediately.
- ❌ **No adaptive bitrate (ABR), one tier locked the whole way** → ✅ bad network stalls, good network wastes. **Switch dynamically per segment with network speed.**
- ❌ **Serving the whole film as one big file** → ✅ can't watch as it downloads, can't adapt, can't cache only hot segments. **It must be segmented + manifest.**
- ❌ **Pulling hot content from origin on demand too, across the board** → ✅ the thundering herd at launch crushes the origin. **Predictable hotspots must be warmed up.**
- ❌ **Storing video bytes and metadata together** → ✅ the browse page gets dragged down by huge bytes. **Bytes go to object storage + CDN, metadata to the database.**

## 12. Evolution path: MVP → growth → maturity

| Stage | User/scale magnitude | What the architecture looks like | What to worry about now |
|---|---|---|---|
| **MVP** | Validate the idea | **Single bitrate + direct distribution**: transcode one tier after upload (or just the source), put it in object storage, maybe hang a simple CDN; no ABR or just the most basic segmenting | **First validate whether anyone watches**; don't self-build a global transcode cluster and multi-CDN from day one |
| **Growth** | Millions of plays | **Multi-bitrate transcode pipeline** (async queue + elastic scaling) + **CDN distribution** + player **ABR**; metadata separated from video bytes; start watching hit rate | Beat down the rebuffering rate, beat down bandwidth cost (hit rate); keep transcoding from piling up |
| **Maturity** | Tens of millions ~ hundreds of millions | **Global multi-CDN + tiered caching + intelligent warm-up** + fine-grained ABR + large-scale elastic transcoding + recommendation system + (as needed) a **live low-latency path** + DRM/geo compliance | Bandwidth cost, hit rate, thundering herd, transcode compute, disaster recovery, rights compliance |

## 13. Reusable takeaways

- 💡 **Move "heavy compute" off the user's request path.** The idea of async transcoding applies to any operation "far exceeding the user's patience" — image processing, report generation, bulk imports should all go into a queue, be elastically scalable, and be retryable.
- 💡 **Putting data nearest the consumer (cache/CDN) is the strongest lever for cutting latency and cost at the same time.** Here it's video segments; elsewhere it might be static assets, API responses, computed results. Hit rate is the cost sheet.
- 💡 **Put the decision at "the end with the most complete information."** ABR hands the bitrate decision to the player that knows the local network best; likewise, many decisions should sink to the layer that holds the most context.
- 💡 **Stock the "predictable hotspots" ahead, fetch the "unpredictable long tail" on demand.** The warm-up + on-demand-origin-pull combo is the universal recipe for any hotspot/long-tail mixed workload.
- 💡 **Make one piece of content into multiple tiers, and pick dynamically by real-time conditions.** The essence of multi-bitrate + adaptive is "prepare a set of elastic tiers, then pick the most suitable at runtime by the real environment" — this idea applies equally to throttling/degradation and tiered service quality.
- 💡 **Identify your system's true "cost bulk" and architect around it.** Here it's bandwidth, which is why CDN, hit rate, and bitrate tiers are the core topics — not code elegance.

## 🎯 Quick quiz

<Quiz
  question="The biggest cost and bottleneck of video streaming usually comes from?"
  :options="['CPU compute', 'Bandwidth — delivering massive video worldwide', 'Memory']"
  :answer="1"
  explanation="That's why CDN, cache hit rate, and bitrate tiers are the core topics — stock content nearest the user, cutting latency and saving bandwidth at the same time."
/>

---

## References & Further Reading

> This template is compiled from the following **official engineering blogs / docs**.

**📖 Engineering blogs / docs:**
- [Netflix: Per-Title Encode Optimization](https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2) — a per-title encoding ladder (bitrate-ladder optimization for transcoding + adaptive bitrate).
- [Netflix Open Connect Overview (PDF)](https://openconnect.netflix.com/Open-Connect-Overview.pdf) — the content-distribution architecture of a self-built CDN (Open Connect) deploying cache servers at ISPs.
- [HLS vs. DASH (Wowza)](https://www.wowza.com/blog/hls-vs-dash) — a comparison of the two major adaptive-bitrate streaming protocols (segmenting and bitrate switching).

---

> 📌 Remember video streaming in one line: **it isn't "a website that can play video" — it's "a global content logistics network." First process one source into multiple versions everyone can watch (transcoding), then stock it at the forward warehouse nearest each person (CDN), and every architectural trade-off ultimately answers "how to deliver video fast, stall-free, and bandwidth-cheap."**
