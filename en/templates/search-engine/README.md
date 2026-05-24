# Search Engine · Architecture Template

> **Representative products**: Google, Bing, Elasticsearch, e-commerce / on-site search
> **One-line definition**: pre-"flip" a massive corpus of documents into an inverted index, so any keyword can find the most relevant results — already ranked — within milliseconds.

---

## 1. One-line definition

A search engine = **an offline world that slowly "builds the index"** + **an online world that queries the index in milliseconds.**

Its core insight is a single sentence: **shift the work from "query time" to "index time."** The reason a user's query can pull an answer out of a billion documents in tens of milliseconds is that the system **organized the data inside out, offline, long ago.** To understand a search engine is to understand this trade: "**spend preprocessing to buy real-time speed.**"

## 2. The business essence: what problem is it solving

The volume of information a human faces far exceeds what they can process. A search engine solves "**from a sea of content, quickly scoop the few most relevant items in front of you**" — it turns "finding a needle in a haystack" into "a precise, sub-second hit."

Where the money comes from: ads next to search results (search intent = the most valuable ad context), licensing of enterprise / on-site search, and traffic-steering plus ranking in e-commerce search.

> A counterintuitive fact: **no matter how strong the search-engine tech, if it can't find the right thing, it's worthless.** Half of its product value rests on "relevance" — a soft metric with no standard answer, requiring continuous tuning.

## 3. Core requirements and constraints

**Functional requirements:**
- [ ] Full-text retrieval: enter a keyword, find the documents containing it
- [ ] Relevance ranking: what you most need to see ranks first
- [ ] Tokenization / autocomplete / spell correction
- [ ] Filtering and faceting (filter / aggregate by dimensions like price, category)
- [ ] Index updates: new content becomes searchable

**Non-functional requirements / quality attributes:**
| Quality attribute | Target | Why it matters for this kind of system |
|---|---|---|
| **Query latency** | < a few hundred ms | Search is an instant interaction; too slow and people leave |
| **Relevance** | The more accurate the better | The soul of the product; directly decides whether it's any good |
| **Index freshness** | Seconds to minutes | How soon new content becomes searchable |
| **Scale** | Billions of documents | Both index and queries are massive |

**Key constraints (boundaries you cannot cross):**
- 🔴 **Documents are massive, queries even more so**: the two workloads have completely different shapes and must be designed separately.
- 🔴 **Relevance has no standard answer**: it's subjective and must be tuned continuously with data, unlike "1+1=2."
- 🔴 **You can't scan the raw text in real time**: running `LIKE '%keyword%'` one by one across a billion documents wouldn't finish by nightfall — you must build an index in advance.

## 4. The big picture

```
═══════════ Offline: build the index (slow and meticulous) ═══════════
   ┌─────────┐   ┌───────────────┐   ┌───────────────┐
   │ Crawler/│──▶│ Doc           │──▶│ Index build:  │
   │ source  │   │ processing:   │   │ produce the   │
   │         │   │ tokenize/norm │   │ inverted index│
   └─────────┘   └───────────────┘   └───────┬───────┘
                                             ▼
                          ┌───────────────────────────────────┐
                          │ Inverted index (shards + replicas)│
                          │ term → "docs containing it"       │
                          └─────────────────┬─────────────────┘
═══════════ Online: query the index (speed first) ═══════════     │
   ┌─────────┐   ┌───────────────┐   ┌───────────────▼──────┐
   │ User    │──▶│ Query parse:  │──▶│ ① Recall: quickly    │
   │ query   │   │ tokenize/fix/ │   │   scoop candidates   │
   │         │   │ autocomplete  │   │   from each shard    │
   └─────────┘   └───────────────┘   │   (broad and coarse) │
        ▲                            │ ② Rank: finely score │
        │                            │   the candidates     │
        │                            │   (precise and fine) │
        └──────── ranked results ◀───┴──────────────────────┘
```

> The soul is the **inverted index** in the middle: it's the offline world's "pre-organized output," and also the online world's "nerve to respond in milliseconds." The whole architecture is just two things: "build it offline, query it online."

## 5. Component responsibilities

- **Crawler / data ingestion**: crawl or receive the documents to be searched. *Why it's needed*: the raw material for the index.
- **Document processing / tokenization**: split documents into terms, normalize case / word forms, drop stop words. *Why it's needed*: the basic unit of search is the "term," so text must first be cut into terms.
- **Index building**: flip "document → terms" into the "term → list of documents" inverted index. *Why it's needed*: this is the core act of "shifting the work to index time."
- **Inverted-index storage (shards + replicas)**: store the massive index in slices, with replicas to absorb query volume. *Why it's needed*: a billion documents can't fit on one machine, nor can one machine survive the query load.
- **Query service**: parse the query, correct, autocomplete, fan out to each shard to recall, then merge. *Why it's needed*: the entry point and coordinator for online queries.
- **Ranking / scoring**: decide the order of results. First a coarse pass by keyword matching (recall), then a more complex model for fine ranking. *Why it's needed*: relevance is where the product's value lives.

## 6. Key data flows

**Scenario 1: build the index (offline, do the work ahead of time)**
```
1. The crawler fetches the document "architectural thinking matters"
2. Tokenize ──▶ [architectural, thinking, matters]
3. Write into the inverted index:
      "architectural" → [doc7, doc12, doc99, ...]
      "thinking"      → [doc3, doc7, ...]
   (note: it stores "which docs a term points to," not "which terms a doc contains")
```

**Scenario 2: a query (online, the two stages of recall + ranking)**
```
1. User searches "architectural thinking" ──▶ query parse, tokenize, correct
2. ① Recall: look up the inverted lists in each shard, take the intersection of the
        "architectural" and "thinking" doc lists ──▶ a few thousand candidates (fast, broad, coarse)
3. ② Rank: compute a fine relevance score only for these few thousand candidates
        (keyword weight, freshness, personalization...) ──▶ pick the top 10 (slow, precise, but on a small set)
4. Merge the results from each shard, return the ranked top 10
```
> The key is the **funnel** of steps 2 and 3: you can never finely rank a billion documents one by one (the math doesn't fit) — instead you "**coarsely filter candidates first, then finely rank a small set.**"

## 7. Data model and storage choices

The core structures: `inverted index (term → doc list + positions + term frequency)`; `forward index (document → field values, for display and filtering)`; `raw documents`.

| Data | Storage type | Why |
|---|---|---|
| Inverted index | A dedicated search-engine store (sharded) | Tailored for the "look up documents by term" access shape |
| Forward index / doc fields | Columnar / KV | Fetch display fields, do faceted aggregation |
| Raw documents | Object storage | Large, immutable, fetched by ID |
| Hot query results | Cache | Head queries are highly concentrated; caching pays off big |

> Teaching point: **an inverted index is "data pre-organized into the handiest possible shape for the problem of 'look up documents by term.'"** Doing full-text search with a relational `LIKE` is grabbing the wrong tool — it never organized the data for this problem.

## 8. Key architectural decisions and trade-offs ⭐

**Decision 1: an inverted index, or just scan directly? (the very foundation of search) ⭐**
- Sequential scan (`LIKE '%term%'`): no preprocessing, but every query scans all documents — infeasible at the billion scale, and you can't do relevance ranking.
- Inverted index: build "term → document" offline, hit it directly at query time.
- **The lean**: inverted index, inevitably. The cost is that **the index takes extra storage, and writes (building the index) become heavier and slower** — which is precisely "spend at write time to buy speed at read time."

**Decision 2: how do you shard the index? By document, or by term?**
- By document (each shard holds the complete index for a subset of documents): queries must fan out to **all** shards and merge (scatter-gather), but writes are simple and scaling is natural.
- By term (each shard holds the full document list for a subset of terms): a query may only need to ask a few shards, but writes and hot-term handling get complex.
- **The lean**: the vast majority go with **document-based sharding** — simple, easy to scale — at the cost of every query having to ask all shards (tail latency dragged by the slowest shard).

**Decision 3: the two-stage architecture of recall + ranking ⭐**
- Rank in one shot: compute a complex score for every matching document — at volume the math blows up.
- Two stages: **recall** a massive set of candidates with a lightweight method, then **finely rank** a small set with a heavy model.
- **The lean**: almost every large search is a two-stage funnel. **This "coarse first, fine second" idea is equally general in recommendation and fraud control.**

**Decision 4: index freshness — how real-time?**
- Full rebuild: simple, but new content has to wait for the next round to be searchable.
- Near-real-time incremental: new documents enter the index quickly, but it's complex to implement and has resource overhead.
- **The lean**: for most scenarios **near-real-time (seconds to minutes) is enough**; don't pay a disproportionate cost for "absolute real-time."

## 9. Scaling and bottlenecks

- **First bottleneck: documents grow, the index won't fit on one machine.** → Fix: shard the index (cut by document), add replicas to absorb reads.
- **Second bottleneck: query volume grows.** → Fix: scale query nodes horizontally + replicas + cache the head hot queries.
- **Third bottleneck: scatter-gather tail latency** (a query waits for the slowest shard). → Fix: control the shard count, pick the best replica, degrade on timeout (return results even if one shard is missing).
- **Fourth bottleneck: index updates and queries contend for resources.** → Fix: split read and write — query on replicas, build elsewhere, then switch over once built.

## 10. Security and compliance highlights

- **Crawl compliance**: respect robots, copyright, and crawl rate; don't hammer someone else's site down.
- **Query privacy**: search terms expose intent and privacy intensely — protect them strictly, anonymize, and limit retention.
- **Permission filtering (critical for enterprise search)**: a user can only find documents **they're authorized to see** — permission filtering must take effect at the recall stage, not "search it out first then hide it" (otherwise existence can be inferred from the result count).
- **Query injection**: escape query syntax to prevent crafted malicious queries from dragging down the system or escalating privileges.

## 11. Common pitfalls / anti-patterns

- ❌ **Doing full-text search with a database `LIKE '%x%'`** → ✅ An inverted index, the proper way to do full-text retrieval.
- ❌ **Not separating index time from query time, so they drag each other down** → ✅ Build offline, query online, reads and writes apart.
- ❌ **Finely ranking all documents in one shot** → ✅ The two-stage recall + ranking funnel.
- ❌ **Chasing 100% real-time indexing** → ✅ Near-real-time is usually enough; don't overpay.
- ❌ **Piling on tech but never tuning relevance** → ✅ Continuously tune ranking with click / feedback data; it's only useful if it finds the right thing.
- ❌ **Enterprise search that searches first then hides** → ✅ Permission filtering must step in at the recall stage.

## 12. Evolution path: MVP → Growth → Maturity

| Stage | Scale | What the architecture looks like | What to worry about now |
|---|---|---|---|
| **MVP** | Millions of documents | **Use an off-the-shelf search engine**, single machine or a few nodes, basic tokenization + keyword ranking | Just get "findable, reasonably accurate" running |
| **Growth** | Tens of millions to billions | Index sharding + replicas, near-real-time updates, autocomplete / correction, initial relevance tuning | Query latency, relevance, index freshness |
| **Maturity** | Billions + personalization | Recall + machine-learning ranking (LTR), query understanding, personalization, multi-language, large-scale distributed indexing | Continuous relevance optimization, scale, cost, experience |

## 13. Reusable takeaways

- 💡 **"Shift the work from query time to write / index time" is the essence of all read optimization.** Caching, materialized views, precomputation, the inverted index — all variants of the same idea.
- 💡 **The two-stage recall + ranking funnel is the general paradigm for "picking a few from a sea of candidates."** Recommendation systems, fraud control, and ad serving all use it.
- 💡 **A special query shape needs a specially organized data structure.** Full-text retrieval needs an inverted index, just as geo queries need a spatial index — don't force a generic tool onto a special problem.
- 💡 **Scatter-gather (fan-out / merge)** is a common skeleton for distributed queries, and its cost is always "waiting for the slowest shard."

## 🎯 Quick quiz

<Quiz
  question="What is the fundamental reason a search engine can return results in milliseconds from a massive corpus?"
  :options="['Scanning all documents in real time at query time', 'Pre-building the data into an inverted index offline', 'The machines are fast enough']"
  :answer="1"
  explanation="Shifting the work from 'query time' to 'index time' — this is exactly the essence of all read optimization: caching, materialized views, precomputation, and the rest."
/>

---

## References & Further Reading

> This template is compiled from the following **real open-source projects** and **official engineering blogs**.

**🔧 Open-source prototypes (read the code directly):**
- [apache/lucene](https://github.com/apache/lucene) — the full-text retrieval core (the foundation under Solr / Elasticsearch / OpenSearch), the standard implementation of inverted index + immutable segments.
- [quickwit-oss/tantivy](https://github.com/quickwit-oss/tantivy) — a Lucene-inspired Rust full-text retrieval library, good for understanding the inverted index and the internals of a retrieval engine.

**📖 Engineering blogs:**
- [Practical BM25 – The BM25 Algorithm (Elastic)](https://www.elastic.co/blog/practical-bm25-part-2-the-bm25-algorithm-and-its-variables) — a term-by-term breakdown of BM25 relevance scoring (TF / IDF, k1, b).
- [Practical BM25 – How Shards Affect Relevance (Elastic)](https://www.elastic.co/blog/practical-bm25-part-1-how-shards-affect-relevance-scoring-in-elasticsearch) — how shards affect relevance scoring (the trade-offs of distributed retrieval).

---

> 📌 Remember a search engine in one line: **it's not "flipping through every document online," it's "organizing the data inside out offline, so online you only query an index that was prepared long ago" — and every design choice answers "how do I trade preprocessing for millisecond-level precision at query time."**
