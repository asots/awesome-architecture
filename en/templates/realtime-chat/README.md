# Realtime Chat · Architecture Template

> **Representative products**: WhatsApp, WeChat, Slack, all manner of IM / group chat
> **One-line positioning**: get a message from one person to another (or to a crowd) — fast, lossless, and in order.

---

## 1. One-line positioning

A realtime-chat product = **an "always-online message delivery network"**: millions, tens of millions of people simultaneously holding open a long-lived connection that "can receive a push at any moment," and the system must guarantee that every message anyone sends **arrives at whoever should receive it as fast as possible, without loss, and in the correct order** — delivered instantly if they're online, stored first and topped up later if they're not.

The most counterintuitive thing about it, architecturally: the hard part is **not the "send," but "sustaining a massive number of long-lived connections" and "guaranteeing reliable, ordered delivery."** A single message is tiny, but "keeping tens of millions of persistent connections all open at once, and being able to pinpoint exactly which connection to push a message to," plus "achieving no-loss, no-duplicate, no-reordering in a reality where the network can drop at any moment," pushes the whole architecture toward core mechanisms like stateful connections, sequence numbers, and acknowledge-and-retry.

## 2. The essence of the business: what problem is it solving

What the user wants is **"the messages I send, the other party sees quickly; the ones they send, I receive instantly; not one can be lost, and the order can't be scrambled."** It replaces "phone calls / email" — communication that's either interrupting or very slow — with what IM offers: **instant, asynchronous, on-the-record continuous conversation.**

Where the value and the money come from:
- **Stickiness and network effects**: your whole relationship chain is here, the cost of migrating away is enormous — the strongest moat;
- **Enterprise / team collaboration** (like Slack): subscription per seat, selling team communication efficiency;
- Value-adds: stickers, memberships, enterprise services, platform ecosystem.

**Key fact: this kind of system's load profile is "massive concurrent persistent connections + a continuous stream of small messages," not "short, fast request-response."** An ordinary website's connections "come and go," while here connections "come and stay open." This one fact dictates why "long-connection gateways," "connections are stateful," and "how to find which connection on which machine to push a message to" become the core propositions of the architecture.

## 3. Core requirements and constraints

**Functional requirements (what the system must be able to do):**
- [ ] Send / receive messages: one-to-one, one-to-many (group).
- [ ] Real-time delivery: when the other party is online, the message arrives near-instantly.
- [ ] Offline messages: when the other party is offline, the message isn't lost, and is topped up once they're back online.
- [ ] Message ordering: within the same conversation, messages are presented in send order.
- [ ] Reliable delivery: no loss, no duplicates (everything sent eventually arrives, and is presented exactly once).
- [ ] Presence: show the other party as online/offline/typing.
- [ ] Groups: member management + fan-out of group messages to every member.
- [ ] Multimedia: send images, voice, files.
- [ ] Multi-device sync: phone and computer online at once, with messages and read state consistent.

**Non-functional requirements / quality attributes (this is where architecture really fights):**
| Quality attribute | Target | Why it matters for this kind of system |
|---|---|---|
| **Message latency** | < a few hundred ms when online | The soul of IM is "instant"; slow it down and it's no longer instant messaging. |
| **Reliable delivery** | No loss, no duplicates | Losing one important message = trust collapses. This is the bottom line. |
| **Message ordering** | Strictly ordered within a conversation | Scramble the order and the conversation is unreadable ("sure!" showing up before "want to grab dinner?"). |
| **Massive concurrent connections** | Millions ~ tens of millions of long connections | Connection count is a core scale metric unique to this kind of system. |
| **Availability** | 99.9%+ | Can't connect = can't use it, and the user is anxious at once. |
| **Presence real-timeness** | Eventual consistency is fine | State changes extremely frequently, brief inaccuracy is allowed — this is precious "slack." |

**Key constraints (boundaries you cannot cross):**
- 🔴 **A long connection is "stateful."** A connection physically hangs on one specific gateway machine, and the system must know "which machine a given user is connected to right now" to push a message to them. This is completely different from how stateless web services scale — the number-one constraint.
- 🔴 **The network is unreliable.** Connections can drop at any moment, messages can be lost or re-sent; reliability and ordering must be guaranteed by application-layer mechanisms (sequence numbers, acks, retries, dedup) on your own.
- 🔴 **Large-group fan-out**: one message sent into a ten-thousand-person group has to reach every member — another "fan-out" problem, sharing the same root as the social-feed celebrity problem.
- Message write volume is large and continuous: every message must be persisted to support offline / history / multi-device.

## 4. The big-picture architecture

```
   Sender (phone/computer, holding a long connection)        Receiver (online / offline)
        │  ① send message                                        ▲
        ▼                                                        │ ⑤ push (when online)
┌──────────────────────────────────────────────────────────────────────┐
│  Long-connection gateway layer (stateful: each machine holds a batch    │
│  of persistent connections)                                            │
│  Gateway A      Gateway B      Gateway C  ... (scale out, can hold      │
│                                                 millions of connections)│
└────┬───────────────────────────────────────────────┬──────────────────┘
     │ ② upstream message                              ▲ ④ "push to user U" → query routing
     ▼                                                │   → find U on gateway C → deliver
┌────────────────────┐         query/update           │
│  Message service     │ ◀──── user↔gateway routing ───┘
│  (core)              │       table (user U is on
│  • assign in-convo   │       gateway C right now)
│    sequence number   │              ▲
│  • persist message   │              │ updated on connect/disconnect
│  • decide receiver    │
│    online/offline     │
│  • online→deliver /    │
│    offline→store       │
└───┬──────────┬───────┘
    │          │ ③ write                       offline branch
    │          ▼                          ┌────────────────────┐
    │   ┌──────────────┐                  │ Offline message     │
    │   │ Message store│                  │ store               │
    │   │ (durable/    │                  │ + offline push svc  │
    │   │  ordered)    │                  │ (wake vendor push   │
    │   └──────────────┘                  │  channel)           │
    │                                     └────────────────────┘
    │                                        ▲ receiver pulls after coming online
    │ group message fan-out
    ▼
┌──────────────┐   ┌────────────────┐   ┌──────────────────────────────┐
│ Group/member │   │ Presence svc   │   │ Media store                    │
│ service      │   │ (presence,     │   │ (img/voice/file, send refs    │
│ (fan-out to  │   │  eventually    │   │  only)                        │
│  members)    │   │  consistent)   │   │                              │
└──────────────┘   └────────────────┘   └──────────────────────────────┘
```

> The soul of this diagram is two blocks: the **"long-connection gateway layer"** (statefully holding a massive number of persistent connections) and the **"user↔gateway routing table"** it depends on (to push a message to someone, first look up which gateway they're connected to right now). The whole system's core mechanisms revolve around these two, plus the **"sequence number (ordering) + persistence (no loss) + online/offline routing"** in the message service.

## 5. Component responsibilities

- **Long-connection gateway layer**: sustains a massive number of clients' **persistent connections**, does heartbeat keep-alive, receives upstream messages and pushes downstream. It is **stateful** — each connection is physically bound to one gateway. *Why you need it*: real-time push requires the server to be able to actively find a client and push to it, which can only rely on a connection held open the whole time; polling is both slow and wasteful. It's the fundamental thing distinguishing this kind of system from an ordinary web app.
- **User↔gateway routing table**: records "which gateway machine a given user is connected to right now." Updated on connect/disconnect. *Why you need it*: connections are stateful, so to deliver a message to user U you must first know where U is connected to send the delivery instruction to that gateway. **This is the key puzzle piece that lets stateful connections scale out.**
- **Message service (core)**: **assigns a monotonically increasing sequence number per message within a conversation** (ordering), **persists** the message (no loss), decides whether the receiver is online and **routes accordingly** (online → deliver directly, offline → store + trigger push). *Why you need it*: reliability and ordering can't come from the network itself; they must be guaranteed by this layer with sequence numbers, persistence, and acks — it's the hub of message flow.
- **Message store**: durably stores messages, supporting offline top-up, history backtracking, and multi-device sync. Write-heavy, read in conversation order. *Why you need it*: messages can't live only in memory/connections, or one disconnect loses them; persistence is the basis for "no loss" and "multi-device consistency."
- **Offline message store + offline push service**: when the receiver is offline, the message is stored in their "offline mailbox," and the system-level push channel **wakes** them; once online, they come pull and top up. *Why you need it*: users are offline much of the time, and you must guarantee "lose nothing even when they're away, and top up when they return."
- **Group / member service**: maintains group membership; group messages must **fan out** to every member. *Why you need it*: a group message is fundamentally "one message to be delivered to N receivers" — a fan-out problem (same root as the social-feed celebrity), and large groups especially need care.
- **Presence service**: maintains online/offline/typing state. *Why you need it*: it's the key to the social sense of presence. But state changes extremely frequently, so **eventual consistency is fine** — a second or two late isn't fatal.
- **Media store**: images/voice/files in object storage, with the message carrying only a **reference (address)**, not stuffing bytes into the message stream. *Why you need it*: large files can't crowd out the lightweight, low-latency message channel — transmit them separately.

## 6. Key data flows

**Scenario 1: One-to-one send, receiver online (the most core real-time path)**
```
1. Sender (connected to Gateway A) ── ① send message ──▶ Gateway A ── ② upstream ──▶ message service
2. Message service:
       a. Assign this message an [in-conversation sequence number seq=N] (guarantees order)
       b. Persist this message (write to message store)         ← persist first, guaranteeing no loss
       c. Query [routing table]: receiver U is connected to Gateway C right now, and online
3. Message service ── ④ "deliver to U" ──▶ Gateway C ── ⑤ push ──▶ receiver's screen
4. After receiving, the receiver replies with an [ack (got seq=N)]
5. No ack received → message service retries delivery per policy; the receiver dedups by seq (drops duplicates)
```
> Note: **persist first, then deliver** (no loss); **monotonic sequence number within a conversation** (ordering); **ack + retry + dedup by sequence number** (reliable and no duplicates). The whole reliable-delivery trio is on this one path.

**Scenario 2: Receiver offline (store + push + top up on coming online)**
```
1~2. Same as above; message service persists, queries routing table ── finds receiver U is offline (no active connection)
3. Message service: write the message into U's [offline mailbox]
4. ──▶ offline push service: via the system-level push channel, send U's device a wake-up push
5. (Some time later) U opens the app, establishes a long connection ──▶ gateway updates routing table (U is now on Gateway B)
6. Once online, U actively [pulls from the offline mailbox] all messages with seq greater than "the max seq I've received"
7. Present them ordered by seq → not one missing, order correct
```
> Note step 6: **resumable, breakpoint-style pulling based on "the max seq I've received"** — neither missing nor duplicating. The sequence number here solves both "top-up" and "dedup" at once.

**Scenario 3: Group message fan-out (the fan-out problem)**
```
Someone posts a message in a group ──▶ message service (assign the group conversation's seq, persist once)
   ──▶ group service: fetch the group member list
   ──▶ for each member: online → query routing table and deliver; offline → into their own offline mailbox + push

   Small group: just fan out one by one, no problem.
   Ten-thousand-person group: one message must reach tens of thousands instantly → "fan-out explosion" (same as the social-feed celebrity problem)
      ✅ Large-group optimization: store the message as "one shared group message stream," with each member maintaining a "read cursor,"
                  pulling by cursor when online/in the group, rather than pushing/storing a copy per person.
```
> Large-group fan-out and the social-feed "celebrity fan-out explosion" are **the same problem**: both are "one write must reach a massive number of receivers." The cracking idea is the same root too — **don't copy one per person (fan-out on write), switch to sharing one copy with each pulling by cursor (fan-out on read).**

## 7. Data model and storage choices

Core entities: `User`; `Conversation (one-to-one / group)`; `Message (with in-conversation seq)`; `User↔gateway routing (who's connected where)`; `Conversation membership`; `Presence`; `Read cursor`.

| Data | Storage type | Why |
|---|---|---|
| Messages (durable) | Ordered append store (sharded by conversation) | Write-heavy, read in conversation order, must support offline top-up and history; sequence numbers are naturally suited to append |
| User↔gateway routing table | In-memory KV | Extremely high-frequency read/write (every delivery queries it), needs ultra-low latency, connections change frequently |
| Presence | In-memory KV (with expiry) | Changes extremely frequently, read frequently, can be eventually consistent — suited to a fast cache with TTL |
| Offline mailbox | KV / ordered queue (per user) | Stores pending messages per user, pulled by sequence number after coming online |
| Conversation / group membership | Relational / wide-column | Member management needs consistency; large-group member lists can be very long |
| Media (img/voice/file) | Object storage | Large files, immutable; the message stores only a reference |
| User account | Relational | Needs transactions and strong consistency (account security) |

> Teaching point: **the message stream and the "connection routing table" are two kinds of data with completely different access patterns.** Messages must be durable, ordered, and backtrackable — suited to an ordered append store; the routing table wants "an ultra-fast lookup of where U is connected, on every delivery" — a textbook in-memory KV. **We describe the routing table and presence with "in-memory KV" because their access pattern is "ultra-high-frequency, low-latency, eventual-consistency-tolerant," independent of any specific product.** Put the routing table in a slow persistent store and every message's delivery gets dragged down.

## 8. Key architectural decisions and trade-offs ⭐

**Decision 1: How do you sustain and scale out long connections? (The Achilles' heel of this architecture.)**
- Root of the problem: a long connection is **stateful** — one connection physically hangs on one gateway machine. The stateless-web-service scaling model where "any machine can handle any request" **fails** here, because "to push a message to U" must reach "the machine U is actually connected to."
- Option A (single machine / vertical): one super-powerful machine holds all connections. Simple, but connection count has a physical ceiling, and a single point of failure = everyone drops at once.
- Option B (scale out + routing layer): **multiple gateways share connections**, plus a **user↔gateway routing table** recording "who's connected where"; on delivery, look up the route first, then send the message to the corresponding gateway.
- **Leaning**: **inevitably go with B.** The cost: ① an extra routing-table layer, which must be **updated in real time** as connections establish/drop (high-frequency writes); ② one extra lookup hop on the delivery path; ③ when a gateway machine fails, all its connections must be able to **reconnect and refresh the route.** **"Combine the stateful connection + a routing table for 'where the state is'" is the universal pattern for scaling out stateful services** — it explicitly separates "state binding" from "routing by state."

**Decision 2: How do you guarantee message ordering?**
- Order by arrival time: network jitter and re-sends make arrival order ≠ send order — **reordering.**
- **Order by a monotonically increasing sequence number within a conversation**: each message gets an incrementing seq within its conversation, and the receiver **presents in seq order**, not arrival order.
- **Leaning**: **use a monotonic sequence number for ordering**, and **scope the ordering to "within a single conversation"** (no need for global ordering — global ordering is extremely costly and unnecessary; you only care about "the order within this conversation"). The cost is needing a mechanism to "issue numbers for a given conversation" (and to keep it from becoming a bottleneck, usually issue numbers sharded by conversation). **"Order only within the smallest scope that genuinely needs order"** — a key piece of scope-narrowing wisdom.

**Decision 3: How do you achieve reliable delivery (no loss, no duplicates)?**
- Fire and forget: simplest, but one network jitter loses the message, violating the bottom line.
- **Ack + retry + idempotent dedup**: ① the receiver replies with an **ack** after receiving; ② the sender side **retries** if no ack comes back; ③ retries cause duplicates, so the receiver side dedups by **seq / message ID** (present each exactly once). Layer "**persist first, then deliver**" on top to guarantee that even if delivery fails, the message isn't lost and can be re-delivered.
- **Leaning**: **the trio (ack + retry + dedup) is indispensable**, because retry and dedup are a pair — you retry to "avoid loss," which inevitably introduces "duplicates," so you must dedup. The cost is a more complex protocol, every message carrying an ID/seq, and maintaining ack state. **"At-least-once delivery + idempotent dedup = the effect of exactly-once"** — the classic combo of distributed delivery.

**Decision 4: Group message fan-out — fan-out on write or shared read?**
- **Fan-out on write (deliver/store a copy per member)**: natural for small groups, simple at read time. But **large groups explode** — one message in a ten-thousand-person group instantly produces tens of thousands of deliveries/stores.
- **Shared read (store one copy of the group message stream, each member records "how far they've read")**: writes are trivially light (store one copy), and members pull by **read cursor.** Friendly to large groups, but reads must aggregate by cursor.
- **Leaning**: **fan-out on write for small groups, shared read for large groups** (isomorphic to the social-feed "ordinary push, celebrity pull"). The cost is two group-message-handling paths in the system at once. **This confirms again: any "one write reaches a massive number of receivers" fan-out problem has its solution in the trade-off between 'copy one each vs share one and pull on demand.'**

**Decision 5: How accurate does presence need to be?**
- Strongly consistent, real-time precise: state changes extremely frequently (every foreground/background switch, disconnect, heartbeat changes it), so strong consistency is extremely costly and unnecessary.
- **Eventual consistency + fast cache + expiry**: write state into an in-memory KV with TTL, allowing brief inaccuracy (you might still show someone online for a second or two right after they go offline).
- **Leaning**: **eventual consistency is plenty.** **Presence is a textbook case of "data that's allowed to be a bit inaccurate,"** and treating it as strongly consistent is an enormous waste. Identifying "which data is allowed brief inaccuracy" saves a lot of cost — this is precious design freedom.

## 9. Scaling and bottlenecks

Unlike an ordinary system: **here the number-one bottleneck is "long-connection count" and "connection routing," and only after that come message writes and large-group fan-out.**

- **First bottleneck: long-connection count.** A single machine has a ceiling on persistent connections it can hold, and you hit it as users grow.
  Fixes: ① **scale out gateways** (multiple machines share connections); ② a **connection routing layer** (user↔gateway routing table) so messages can find the right gateway; ③ nearby ingress, connection load balancing.
- **Second bottleneck: large-group fan-out.** One message must reach a massive number of members.
  Fix: **large-group shared read (fan-out on read) + read-cursor pulling**, rather than pushing/storing a copy per person (Decision 4).
- **Third bottleneck: message-store write volume.** Every message must be persisted — extremely large and continuous.
  Fixes: ① **shard by conversation** (scatter the write pressure); ② archive cold messages to a cheaper storage tier; ③ make the write path async and batched.
- **Fourth bottleneck: high-frequency read/write of the routing table / presence.** Every delivery queries the route, and state changes frequently.
  Fixes: put them in **in-memory KV**, shard by user, and use eventual consistency + TTL for presence to lower write pressure.
- **Media traffic**: images/voice/files go through object storage + (when necessary) CDN, not crowding the message channel.

## 10. Security and compliance essentials

- **End-to-end encryption (depends on product positioning)**: products with a strong-privacy positioning like WhatsApp encrypt/decrypt messages **on both ends, with the server only relaying ciphertext and unable to see the content.** This is the strongest privacy promise, but the cost is: the server can't do content moderation, can't do server-side full-text search, and multi-device sync key management is complex. Whether to adopt E2EE is a product-level trade-off.
- **Message privacy and minimal retention**: messages are highly sensitive data, requiring transport and storage encryption, access control, and clear retention and deletion policies.
- **Transport encryption**: even without end-to-end, the client-to-server link must be encrypted.
- **Abuse and spam**: prevent harassment, spam, and bulk group-pulling, requiring risk control and throttling.
- **Metadata privacy**: even with message content encrypted, metadata like "who talked to whom and when" is itself sensitive — keep as little as possible and control it strictly.
- **Groups and permissions**: who can join a group, who can see history — permissions must be enforced server-side, not merely hidden by the client.

## 11. Common pitfalls / anti-patterns

- ❌ **Using polling instead of long connections** → ✅ polling is either slow (long interval) or wasteful (short interval hammering the server) — neither achieves true real-time. **Real-time push relies on long connections.**
- ❌ **Not assigning sequence numbers, displaying by arrival order** → ✅ network jitter / re-sends inevitably reorder. **Use a monotonic in-conversation sequence number for ordering.**
- ❌ **No dedup, so retries cause duplicate messages** → ✅ retrying for "no loss" inevitably brings duplicates, **so you must dedup idempotently by seq/message ID.**
- ❌ **Fire and forget, no ack/retry** → ✅ one network jitter loses it. **Ack + retry + persist-before-deliver is what achieves no loss.**
- ❌ **Synchronous fan-out on write for large groups** → ✅ one message in a ten-thousand-person group is tens of thousands of deliveries instantly, jamming up. **Switch large groups to shared read + cursor pulling.**
- ❌ **Treating connections as stateless and routing freely** → ✅ connections are stateful, **you must have a routing table to know where a user is connected**, or the message can't be pushed to them.
- ❌ **Treating presence as strongly consistent** → ✅ state changes extremely frequently, **eventual consistency is plenty**, and strong consistency is an enormous waste.
- ❌ **Stuffing media bytes into the message stream** → ✅ large files crowd the low-latency message channel; **store them in object storage and carry only a reference in the message.**

## 12. Evolution path: MVP → growth → maturity

| Stage | User/scale magnitude | What the architecture looks like | What to worry about now |
|---|---|---|---|
| **MVP** | Validate the idea / tens of thousands of users | **Single-machine long connections** (or simple polling) + a single message store; the basic mechanisms of sequence numbers, acks, offline mailbox in place first; no large-group optimization | **First get the basics of "real-time, no-loss, ordered" right**; connections all on one machine still hold up |
| **Growth** | Millions of connections | **Distributed long-connection gateways + routing table**; offline message store + offline push; message store sharded by conversation; presence on a fast cache | Make long-connection scale-out, connection routing, and reliable delivery still hold under distribution; keep an eye on the first large groups |
| **Maturity** | Tens of millions of connections / massive messages | **Large-group optimization (shared read + cursor)** + **multi-active/multi-region** + nearby ingress + complete multi-device sync + (per positioning) **end-to-end encryption** + risk control and compliance | Large-group fan-out, connection scale, disaster-recovery multi-active, message-storage cost, privacy and compliance |

## 13. Reusable takeaways

- 💡 **Stateful services scale out via "state binding + a routing table."** A long connection hangs on one machine, so use a "where the state is" routing table to locate it. Scaling any stateful resource (sessions, connections, shard primary/replica) can borrow this pattern of "explicitly separating state binding from routing-by-state."
- 💡 **Order within "the smallest scope that genuinely needs order," don't chase global ordering.** A monotonic sequence number within a single conversation is enough; global ordering is costly and meaningless. Scope narrowing is the key to performance and complexity.
- 💡 **At-least-once delivery + idempotent dedup = the effect of exactly-once.** Retry (anti-loss) and dedup (anti-duplicate) are a twin pair of mechanisms — applicable to any "reliable delivery over an unreliable channel" scenario, from messages to payment callbacks to event delivery.
- 💡 **"One write reaching a massive number of receivers" is always a fan-out problem, with the solution between "copy one each vs share one and pull on demand."** Large-group fan-out, the social-feed celebrity, broadcast notifications — same root at heart, same trade-off applies.
- 💡 **Identify the data "allowed to be less accurate / less real-time" and degrade it.** Presence uses eventual consistency to save a lot of cost. Wherever you can find slack, you can trade it for elasticity and savings.
- 💡 **Persist first, then deliver.** Putting "persistence" before "sending" is the bedrock of "no loss" — this ordering principle holds in any write-and-deliver chain that demands reliability.

## 🎯 Quick quiz

<Quiz
  question="To achieve «no loss, no duplicates» over an unreliable network, the usual combo is?"
  :options="['Retry only', 'At-least-once delivery + idempotent dedup', 'Dedup only']"
  :answer="1"
  explanation="Retry (anti-loss) and dedup (anti-duplicate) are a twin pair of mechanisms; together they achieve the effect of «exactly-once» — applicable to reliable delivery over any unreliable channel."
/>

---

## References & Further Reading

> This template is compiled from the following **official engineering blogs**.

**📖 Engineering blogs:**
- [How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) — migrating massive message storage from Cassandra to ScyllaDB, with request coalescing to cut latency.
- [Slack: Flannel, an edge cache to make Slack scale](https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale/) — an edge cache carrying millions of WebSocket long connections.
- [WhatsApp: Scaling to Millions of Simultaneous Connections (Rick Reed)](http://www.erlang-factory.com/conference/SFBay2012/speakers/RickReed) — supporting millions of concurrent long connections on a single machine (one process per connection).

---

> 📌 Remember realtime chat in one line: **it isn't "a website that can send messages" — it's "an always-online message delivery network." The core difficulty is "sustaining a massive number of stateful long connections + achieving no-loss, no-duplicate, no-reordering over an unreliable network," and every architectural mechanism (gateway routing, sequence numbers, ack/retry/dedup, offline store-and-push) serves these two things.**
