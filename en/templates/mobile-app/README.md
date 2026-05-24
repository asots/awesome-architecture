# Mobile App · Architecture Template

> **Representative products**: the overwhelming majority of iOS / Android apps (social, notes, travel, banking, collaboration tools…)
> **One-line definition**: on a phone where "the network drops without warning and battery and memory are both tight," build an app that always reacts instantly and works even offline.

---

## 1. One-line definition

A mobile app = **"half a system" running in your pocket** + **a sync mechanism that quietly keeps it aligned with the cloud.**

The most counterintuitive thing architecturally: the biggest difference from the "web pages" you know isn't the small screen — it's that **you cannot assume the network exists.** A web page can "ask the server on every click," because it's usually on stable broadband; a phone, however, drops signal at any moment — in an elevator, a subway, an underground garage. So the central question of the whole architecture shifts from "how do I send the request to the server" to "**when the network is gone, can this app still be used, and does the user even notice?**" The answer is the soul of this template: **offline-first + data sync.**

## 2. The business essence: what problem it solves

What users want is a tool they can **pull out on the spot, tap once and get an instant reaction, and keep working with whether or not there's a signal.** It replaces the miserable experience of "open the web page → spinner waiting to load → connection hiccups, start over."

The phone as a medium brings two iron realities:

- **The network is unstable and expensive**: signal drops, latency spikes, data costs money. Any design that "must wait for the server to reply before moving on" turns into spinners and freezes under a weak network.
- **Device resources are limited**: battery, memory, storage, and CPU are all finite, and you're competing with dozens of other apps. You can't "just compute mindlessly" the way a server can.

**The key fact: on a phone, "respond instantly" isn't an optimization item — it's a survival line.** A web page half a second slow, the user may tolerate; a phone app that doesn't react on a tap, and the user's finger is already swiping to close it. This one fact drives nearly every architectural trade-off below — especially "**why write data locally first and let the UI move first.**"

## 3. Core requirements and constraints

Split requirements into two kinds. **Distinguishing "function" from "quality" is the architect's first fundamental.**

**Functional requirements (what the system must be able to do):**
- [ ] Instant-response interaction: tap and it moves, without waiting on the network.
- [ ] Offline usable: browse, edit, and create even without signal, then auto-catch-up once connected.
- [ ] Data sync: push local changes to the cloud, pull cloud changes back to local, keep multiple devices consistent.
- [ ] Conflict handling: when the same record is changed simultaneously on two devices (or local and cloud), it must converge sensibly.
- [ ] Push notifications: when the server has a new message/change, it can proactively wake the app.
- [ ] Media handling: take photos, upload images and video, without blocking the main flow under a weak network.
- [ ] Background sync: when the app goes to the background or just launches, quietly align the data.

**Non-functional requirements / quality attributes (this is where architecture actually fights):**
| Quality attribute | Target | Why it matters for this kind of system |
|---|---|---|
| **Interaction response latency** | < 100ms (local operations) | The user's finger is far faster than the network; a bit slow and "this app feels laggy." |
| **Weak-network/offline availability** | Complete core operations even offline | Subways, elevators, and garages are the norm, not edge cases. |
| **Sync correctness** | No data loss, no scrambling | The user edits the same note on two devices and half of it is lost — a fatal collapse of trust. |
| **Resource usage** | Thrifty on battery, memory, data | A battery- and data-hungry app gets uninstalled by users and killed in the background by the system. |
| **Startup speed** | Cold start as fast as possible | The first screen must show content immediately, even if it's the cached old data first. |

**Key constraints (boundaries you cannot cross):**
- 🔴 **The network is unreliable and beyond your control.** This is the number-one constraint; the entire architecture serves it.
- 🔴 **You can't force all users to upgrade.** Old client versions will **survive on users' phones for a very, very long time** (some people just won't update). This means **the server API must stay backward compatible for the long haul** — the most easily overlooked, and most fatal, constraint on mobile.
- 🔴 **Device resources are limited**; local storage and computation must both be used sparingly.
- Once client code ships, you **can't pull it back**; a buggy version will keep producing data, and the server has to tolerate it.

## 4. The big-picture architecture

```
┌──────────────────────────── User's phone (client = half a system) ──────────────────────┐
│                                                                                          │
│   ┌──────────────┐      ┌──────────────────┐      ┌──────────────────────────┐          │
│   │   UI layer    │◀────▶│  State management │◀────▶│  Local DB / cache          │          │
│   │ (UI, interact)│      │ (in-memory app    │      │ (persisted "copy of truth") │          │
│   │              │      │  state)           │      └────────────┬─────────────┘          │
│   └──────────────┘      └──────────────────┘                   │                        │
│        ▲   writes land locally first, UI reflects instantly (optimistic update)          │
│        │                                                       ▼                          │
│        │                                          ┌──────────────────────────┐          │
│        └──────── backfill after sync succeeds/fails ──│ Sync engine + outbox queue │          │
│                                                   │ (queue, retry, pull, merge) │          │
│                                                   └────────────┬─────────────┘          │
└────────────────────────────────────────────────────────────────┼────────────────────────┘
                                                                   │ async under weak network, interruptible, retryable
                          push "new changes, come pull"            │
   ┌───────────────────────┐         wake             ┌──────────▼──────────────────┐
   │     Push service       │ ─────────────────────────▶│   Mobile-specific backend (BFF)│
   │ (system-level channel) │                           │ • Trim/aggregate data for mobile│
   └───────────▲───────────┘                           │ • Take upstream changes,        │
               │ trigger push                           │   push down remote changes      │
               │                                        │ • Auth, rate limit, API versioning│
               │                                        └───┬───────────┬──────────┘
               │                                            │           │
   ┌───────────┴───────────────────────────────────────────▼──┐   ┌────▼──────────────┐
   │   Sync service / business backend                          │   │   Object storage   │
   │ • Holds the cloud "authoritative data"  • Versions/timestamps │ │ (images/video etc.)│
   │ • Conflict adjudication                                     │   └───────────────────┘
   └───────────────────────────────────────────────────────────┘
```

> The soul is that whole block of **client** on the left and the **sync engine** in the middle — together they degrade "the network" from "a roadblock you must wait on for every operation" into "an optional thing that quietly aligns in the background." This is the meaning of "half a system in your pocket."

## 5. Component responsibilities

Each key part above: **what it does + why it's needed** (a part with no "why" is over-engineering).

- **UI layer**: renders the interface, responds to gestures. It **reads only local state**, never waits directly on the network. *Why it's needed*: decoupling the UI from the network is the only way to achieve "tap and it moves" — the precondition for instant response.
- **State management**: the in-memory application state of "what the current interface should show," the intermediary between the UI and the local database. *Why it's needed*: the interface needs to reflect data changes quickly and consistently, which requires a centralized, predictable source of state.
- **Local DB / cache**: the persisted copy of data on the phone, the **"local truth" when offline.** All reads read it first, all writes land in it first. *Why it's needed*: without local persistence there's no offline usability and no "show the old data first" cold-start speed. It's the physical foundation of offline-first.
- **Sync engine + outbox queue**: queues local write operations and sends them asynchronously to the server when there's a network; meanwhile pulls remote changes and merges them into local. It handles retries, deduplication, resumable transfer, and conflict handling. *Why it's needed*: it's the buffer zone between "local" and "cloud," absorbing all the trouble of the unreliable network (drops, slowness, retries, out-of-order) in this one layer so it doesn't pollute the UI.
- **Mobile-specific backend (BFF, Backend for Frontend)**: the access layer dedicated to mobile. It **trims and aggregates data from multiple backend services into exactly the shape mobile needs**, returns it in one call to cut round-trips and data usage; it handles auth, rate limiting, and **holds the line on API version compatibility.** *Why it's needed*: generic backend interfaces are often too "fat" (many fields, multiple requests required), wasting data and round-trips under a weak network; the BFF "kneads the data into shape and then feeds it" on behalf of mobile.
- **Sync service / business backend**: holds the cloud's **authoritative data**, records each record's version/timestamp, adjudicates conflicts when it receives upstream changes, and broadcasts changes to other devices. *Why it's needed*: for multiple devices to align, you need an authoritative source of "who's in charge" and a set of adjudication rules.
- **Push service**: through a system-level channel, **proactively wakes** a sleeping app to pull when the server has a new change. *Why it's needed*: an app can't poll forever (battery- and data-hungry); waking it via push is the key to keeping it fresh while sparing resources.
- **Object storage**: stores bulky media like images and video. *Why it's needed*: media is large and immutable; it shouldn't be stuffed into the database or travel with sync messages; the client uploads/fetches it directly to/from object storage, separating the media channel from the data-sync channel.

## 6. Key data flows

Three scenarios that best capture this system's character.

**Scenario 1: user creates a record (the core path of offline-first + optimistic update)**
```
1. User taps "save" on this note ──▶ sync engine first writes it into the [local DB] (marked "pending sync")
2. ──▶ state management updates immediately ──▶ UI shows this note immediately
       ★ At this instant the user has already seen the result, with zero wait on the network. This is the "optimistic update."
3. ──▶ sync engine throws this write into the [outbox queue], to be processed slowly in the background
4.    When there's a network ──▶ send asynchronously to the BFF ──▶ sync service persists it, assigns a version number
5.    Success ──▶ change this local record's mark from "pending sync" to "synced" (the UI barely notices)
      Failure ──▶ leave it in the queue, retry later; meanwhile the UI stays usable as normal
```
> Note step 2: **the network hasn't joined in yet, and the user is already using it.** This is the magic by which offline-first turns "spin and wait for the server" into "instant reaction." The network steps down from "lead role" to "supporting role wrapping up in the background."

**Scenario 2: pull and merge remote changes (multi-device alignment)**
```
Device B changed a record ──▶ sync service records the new version ──▶ via the [push service] sends device A a "there's a change"

  Device A receives the push ──▶ wakes the app ──▶ sync engine goes to the BFF with "my local version/cursor" to pull a delta
  BFF ──▶ returns only "the part newer than yours" (a delta, not the full set, saving data)
  Sync engine ──▶ [merges] the remote changes into the local DB:
        · local untouched this record   → overwrite directly
        · local also changed this record → trigger [conflict resolution] (see Decision 3)
  Merge done ──▶ state management refreshes ──▶ UI updates
```
> Architectural point: **pull a delta, not the full set** — relying on both sides carrying a "version/cursor"; **wake by push, not polling** — saving battery and data.

**Scenario 3: upload an image (media and data on separate paths)**
```
1. User picks an image ──▶ client first generates a local thumbnail, UI shows it immediately (placeholder)
2. ──▶ throw the "upload image" task into the outbox queue (resumable transfer)
3.    When there's a network ──▶ client uploads the image directly to [object storage], gets a reference (URL/key)
4. ──▶ send only this "reference" to the backend along with the data sync (no large files in the sync message)
5.    Other devices pull this record ──▶ fetch the image from object storage on demand by reference
```
> Architectural point: **bulky media goes through object storage, small metadata goes through the sync channel** — two separate paths; under a weak network, a half-uploaded image can resume, and never blocks the main flow.

## 7. Data model and storage choices

Core entities: `user` ─ `device` ─ `domain object (e.g., note/order/message)`; every domain object carries **sync metadata**: `version/timestamp`, `sync status (pending/synced/conflict)`, `mapping of local ID to server ID`. Media is an independent `media object`; the data stores only its reference.

| Data | Storage type | Why |
|---|---|---|
| Client domain data (offline read/write) | Embedded database on the device | Needs offline availability, fast local queries, and persistence of the "local truth" |
| Outbox operations / sync queue | Local persisted queue | After the app is killed and restarts, unsent changes must not be lost |
| Images / video and other media | Object storage | Large, immutable, fetched by reference; shouldn't be stuffed into the database or sync message |
| Cloud authoritative data + version info | Server database (supporting delta queries by version/cursor) | To adjudicate conflicts and "send only deltas," it must be retrievable by version |
| Access tokens / keys | The device's secure keystore (system-level encrypted area) | Sensitive credentials must never land in plaintext in ordinary storage (see Section 10) |

> Teaching point: **the mobile "local database" is not a cache — it's "a copy of the truth."** Treat it as a temporary cache of "go fetch from the server if not found" and you've fallen back to the online-first old path; treat it as "locally authoritative, aligned with the cloud in the background" and that's offline-first. **The client holding real state is exactly what makes it "half a system" rather than a "thin client."**

## 8. Key architectural decisions and trade-offs ⭐

**(The most valuable section of this template.)** Every fork on mobile revolves, almost without exception, around "the network is unreliable" and "how much state the client holds."

**Decision 1: offline-first, or online-first?**
- Online-first (thin client): every operation hits the server; the client stores almost nothing. Simple to implement, always the latest data, no sync or conflicts to handle. **But under a weak/no network it freezes outright and is unusable,** and every interaction waits a network round-trip — slow.
- Offline-first (thick client): write locally first, move the UI first, then sync in the background. **Usable offline, fast wherever you tap.** The cost is building your own local storage, sync engine, and conflict resolution — an order of magnitude more complexity.
- **Verdict**: **any app that's "interaction-frequent, wants to feel smooth, and may be used under a weak network" should be offline-first.** This is the fundamental choice that distinguishes mobile from web. The cost is the full complexity of sync and conflicts — but that's precisely where the core value of mobile architecture lies; there's no dodging it. Pure-query scenarios, or those with extremely high strong-consistency requirements (like a real-time balance), can stay online-first or hybrid.

**Decision 2: sync strategy — full, incremental, or bidirectional?**
- Full sync: pull all the data every time. Simplest to implement, but **once the data grows it's data-, battery-, and time-hungry,** and under a weak network it simply can't finish.
- Incremental sync: both sides record a "version/cursor," transmitting only "the part that changed since last time." Saves data, supports resumable transfer. The cost is maintaining version state on both ends and handling out-of-order and packet loss.
- Bidirectional sync: push local changes up, pull cloud changes down — both directions needed. Most complete in function (supports multi-device collaboration), but **conflicts are almost inevitable** and must be paired with conflict resolution.
- **Verdict**: **incremental + bidirectional** is the standard for a mature mobile app. First use "pull a delta" to drive data usage down, then use "bidirectional + conflict resolution" to support multiple devices. The cost is that the sync engine is the hardest part of the whole app to get right and the most prone to hidden bugs — worth the most design and testing.

**Decision 3: how to resolve conflicts? (the must-answer question of bidirectional sync)**
- **Last-Write-Wins**: whoever's timestamp/version is newer wins. Simplest to implement, but **silently drops the other side's change** — dangerous for important data. Suits "not-so-important, fine-to-overwrite" fields (like "last read position").
- **Version vectors / causal tracking**: record "how many times each record was changed on which devices," and from that judge whether it's a "true conflict" or "one side contains the other." Precisely identifies conflicts, at the cost of heavier metadata and more complex logic.
- **The CRDT idea (conflict-free merge data structures)**: design data into structures whose "merge result is identical regardless of order" (like counters, sets, collaborative documents). Achieves **automatic merging and never loses a change,** but applies only to data that fits into such structures, and has a high implementation bar.
- **Verdict**: **tier by the data's importance** — use Last-Write-Wins for the trivial, consider the CRDT idea or semantic merging for important lists/sets, and for true conflicts that can't be auto-resolved, **hand them to the user to choose manually** ("keep this one or that one"). **There's no one-size-fits-all strategy; the key is to think through "if this record gets overwritten and lost, can the user accept it?"**

**Decision 4: how much state does the client hold? (thick vs. thin)**
- Thin: cache only a little data, keep business logic on the server as much as possible. The client is light, easy to maintain, and logic changes don't require shipping a version — but offline ability is weak and interaction is slow.
- Thick: store lots of data locally and run lots of business logic (validation, computation, sorting); strong offline ability, fast response. The cost is logic scattered across both client and server, **prone to inconsistency,** and client logic is hard to change once shipped.
- **Verdict**: **"compute locally what can be computed locally, but keep authoritative adjudication on the server."** The client can optimistically compute a result to move the UI first, but right-or-wrong is ultimately decided by the server. Note: once client logic ships it's hard to change, so **don't hardcode "rules that may change frequently" (like risk control, pricing) into the client** — those go on the server, or become pushable config.

**Decision 5: how to feed data to mobile — a generic API or BFF trimming?**
- Generic API: the backend provides a set of generic interfaces, and mobile composes the calls itself. Easy for the backend, but mobile often has to **make several requests and get back a pile of unused fields,** wasting round-trips and data under a weak network.
- BFF (trim and aggregate for mobile): a dedicated layer that **aggregates and trims, in one shot, the data some mobile page needs** before returning it.
- **Verdict**: mobile is worth a BFF. **"One round-trip to get exactly the data one screen needs" is a huge optimization for the weak-network experience.** The cost is maintaining one more layer, and the BFF having to evolve along with the client's page needs.

**Decision 6: API version compatibility — can you assume everyone upgraded?**
- Assume everyone upgraded: the server serves only the latest version, and changes interfaces at will. **But in reality, old versions live on users' phones for many years,** and the moment the server changes an interface they depend on, those old clients **crash outright or malfunction.**
- Always backward compatible: add-only new fields, never change old field semantics, use version negotiation to keep old clients running.
- **Verdict**: **mobile APIs must be backward compatible — this is iron law, non-negotiable.** Because you **can't force an upgrade,** any "breaking change" is actively crashing a batch of users' apps. The cost is that interfaces can only "grow forward," carry historical baggage for the long haul, and need a clear versioning and deprecation strategy. **This is the most critical discipline distinguishing mobile architecture from "internal service-to-service calls."**

## 9. Scaling and bottlenecks

Unlike an ordinary backend: mobile's bottleneck often isn't "can the server take it," but **"the last-mile network" and "sync itself."**

- **First bottleneck: sync conflicts and data volume.** The more user data and the more devices, the harder a full sync becomes and the more frequent the conflicts.
  Fixes: ① change full to incremental (transmit only changes); ② paginate/chunk the sync, don't pull it all in one breath; ③ sync by "recent/most-used" priority, pull cold data on demand; ④ tier the conflict strategy, auto-resolve everything that can be auto-merged to reduce bothering the user.
- **Second bottleneck: the weak-network experience.** With poor signal, sync is slow, media won't transfer, and operations feel "unresponsive."
  Fixes: ① optimistic update so the UI always moves first, hiding the network in the background; ② queueing + resumable transfer, so a drop can resume and a restart loses nothing; ③ media goes through object storage direct upload, compressible, deferrable; ④ request coalescing and debouncing, don't fire a flurry of tiny requests under a weak network.
- **Third bottleneck: push reachability.** Pushes may be delayed, throttled by the system, or turned off by the user, leaving the client "unaware there's new data."
  Fixes: ① treat push only as "a hint to go pull," not as "the data itself" (a push may be lost, the data must not be); ② back it up with "app returns to foreground / scheduled" fallback pulls; ③ for critical changes, guarantee eventual consistency via "the next sync will definitely pull it," not by relying on a single push delivery.
- **Fourth bottleneck (unique to mobile): client version fragmentation.** Users linger on a motley assortment of old versions, and the server has to serve them all at once.
  Fixes: ① strict backward compatibility of the API; ② make changeable rules into "server-pushable config" to bypass shipping a version; ③ monitor each version's share, and for versions too old to be compatible, do a "graceful forced-upgrade prompt"; ④ the server must tolerate dirty data produced by old/buggy clients and do defensive validation.

## 10. Security and compliance essentials

- 🔴 **Devices get lost, stolen, jailbroken/rooted.** You must assume "the attacker can physically hold this phone and read the app's local files."
- **Encrypt local sensitive data**: personal information and business data stored on the phone should be encrypted where warranted; highly sensitive data (passwords, keys) goes only into the system-provided **secure keystore,** never in plaintext into an ordinary database or file.
- **Store tokens securely**: login tokens go in the secure keystore, not ordinary storage; design **short-lived tokens + refreshable/revocable,** so that if the device is lost you can invalidate it on the server.
- **Transport security + certificate validation**: encrypt all communication; do **certificate pinning** on critical interfaces to stop anyone from using a forged certificate in the middle to eavesdrop/tamper with traffic.
- **Jailbreak / root risk**: on such devices, the app's local protections (including locally stored keys) can all be bypassed. The architectural countermeasure is **"don't put trust in the client"** — truly important validation, risk control, and billing decisions are all done on the server, and the client's conclusions are always treated as "pending server confirmation."
- **The server doesn't trust the client**: any data from the client may be tampered with (including from a modified old-version/cracked client). The server must re-validate permissions, parameters, and quotas.

## 11. Common pitfalls / anti-patterns

- ❌ **Treating the app as a thin client, syncing a server request on every operation** → ✅ Default to offline-first: write locally first, move the UI first, so it won't freeze under a weak network.
- ❌ **Not doing optimistic update — spinning and waiting for the server's reply after a tap** → ✅ Reflect the user's operation into local and the UI immediately; the network wraps up in the background. **Perceived response matters more than the true round-trip.**
- ❌ **Changing the API at will, assuming everyone upgraded** → ✅ The API is always backward compatible; old clients persist for the long haul, and a breaking change is crashing them.
- ❌ **Storing sensitive data in plaintext locally** → ✅ Encrypt storage, credentials go in the secure keystore; assume the device will be breached.
- ❌ **Pulling the full data set on every sync** → ✅ Do incremental sync with a version/cursor — saves data, battery, and is the only way to finish under a weak network.
- ❌ **Treating the local database as just a "read cache," with the truth always on the server** → ✅ Local is "a copy of the truth" — that's what makes offline usability possible.
- ❌ **Treating push as "the data itself," so a lost push loses the data** → ✅ Push is only "a hint to go pull"; data relies on sync for eventual consistency.
- ❌ **Hardcoding changeable business rules (risk control/pricing) into the client** → ✅ Put them on the server or make them pushable config; the client can't be changed once shipped.
- ❌ **Stuffing bulky media into the sync message to travel together** → ✅ Media goes through object storage direct upload; sync only its reference.

## 12. Evolution path: MVP → growth → maturity

Architecture grows. **Don't fit an MVP to a maturity diagram.**

| Stage | User/scale | What the architecture looks like | What to worry about now |
|---|---|---|---|
| **MVP** | Validate the idea | **An online-first thin client**: the UI calls backend interfaces directly, stores almost no local data, works when the network is good | First validate "does anyone want this app" — don't build a sync engine up front |
| **Growth** | 10K–1M users | Add **local cache + optimistic update**: cache core data locally, land operations locally first to move the UI first, one-way sync in the background; introduce push and a BFF; the API starts caring about backward compatibility | Get the weak-network experience and response speed up; watch the "tapped, no reaction" stall points |
| **Maturity** | 10M+ / multi-device | **Full offline-first + bidirectional incremental sync**: a locally authoritative database, a robust sync engine, tiered conflict resolution, resumable transfer, media on a separate path, strict API version governance and config push | Sync correctness, conflict convergence, version-fragmentation governance, resource usage and battery thrift |

## 13. Reusable takeaways

- 💡 **First ask "can this system assume the network exists" — the answer decides everything.** If it can't, you must put state where it's closest to the user (local) and make remote communication an optional background behavior. This idea also applies to any "edge computing" or "weakly-connected IoT" scenario.
- 💡 **Perceived response often matters more than true latency.** Optimistic update doesn't get the data to the server sooner, but it makes the user "feel fast." Any design that "lets the user see feedback sooner and hides the wait in the background" is worth money.
- 💡 **Converge all the trouble of the unreliable thing (the network) into one dedicated buffer layer (the sync engine).** Don't let dirty work like retries, out-of-order, and conflicts leak to the UI. This is the same as the general idea of "isolate a volatile/unreliable dependency within one layer."
- 💡 **Anything that "can't be pulled back once shipped" must have a backward-compatible interface.** Mobile clients are like this, and so are publicly-exposed APIs and data formats others depend on. When you can't force the other side to upgrade, compatibility is discipline.
- 💡 **Don't put trust where you can't control it.** The client runs on the user's (possibly the attacker's) device, and any of its conclusions must be confirmed by a server you do control.

## 🎯 Quick quiz

<Quiz
  question="What is the most fundamental architectural difference between a mobile app and an ordinary web page?"
  :options="['The screen is smaller', 'It must assume the network can drop at any moment, hence local-first and offline-usable', 'It uses a different programming language']"
  :answer="1"
  explanation="It's 'half a system tucked in your pocket that may drop offline at any moment' — every trade-off answers 'is it still smooth when the network is gone, and does the data still align?'"
/>

---

## References & Further Reading

> This template is compiled from the following **official architecture guides** and **real open-source projects**.

**📖 Official guides / philosophy:**
- [Android: Build an offline-first app](https://developer.android.com/topic/architecture/data-layer/offline-first) — Google official: the local data source as the single source of truth, the UI reads local, a background sync engine.
- [Local-first software (Ink & Switch)](https://www.inkandswitch.com/essay/local-first/) — the original long essay proposing the "local-first" concept: local-first, offline-usable, and conflict merging.

**🔧 Open-source prototypes (read the code directly):**
- [automerge/automerge](https://github.com/automerge/automerge) — the classic CRDT library, automatically merging concurrent changes across devices without a central server.

---

> 📌 Remember the mobile app in one line: **it isn't "a web page with a smaller screen" — it's "half a system tucked in your pocket that may drop offline at any moment." Every architectural trade-off ultimately answers: 'when the network is gone, is it still smooth, and does the data still align?'**
