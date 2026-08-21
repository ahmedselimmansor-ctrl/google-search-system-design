<div align="right">

**[🇸🇦 اقرأ بالعربية](../ar/11-freshness.md)**

</div>

# Chapter 11 · Freshness & Real-Time Indexing

**← [10 · Caching](10-caching.md)** · [🏠 Home](../../README.md) · **[12 · Reliability](12-reliability.md) →**

---

## 11.1 The problem batch indexing cannot solve

[Chapter 06](06-indexing.md) described index construction as a MapReduce over the whole corpus — 1 PB of text, ~14 hours. That is perfectly adequate for the ordinary web. It is completely useless for an earthquake.

The requirement from [Chapter 02](02-requirements.md) is *median time-to-index under 5 minutes for the hot set*. A full rebuild cannot deliver that at any cost, because the bottleneck is not compute — it is the fact that a full rebuild recomputes **10¹¹ documents to reflect changes in 10⁶ of them**. The write amplification is 100,000×.

The answer is not to make the batch job faster. It is to **stop rebuilding what did not change** — which is precisely the insight behind Google's Percolator paper (2010) and the Caffeine indexing system it enabled.

> **Diagram D-55 — Two-track indexing**

```mermaid
flowchart TB
    WEB["🌐 Web changes"]

    WEB --> DETECT{"How was the<br/>change learned?"}

    DETECT -->|"Push: feed, sitemap ping,<br/>API submission"| FAST
    DETECT -->|"Pull: scheduled recrawl<br/>found new content"| CLASSIFY
    DETECT -->|"Signal: trending query<br/>with no good results"| FAST

    CLASSIFY{"Is this page<br/>freshness-critical?"} -->|"Yes — news, forum,<br/>high change rate,<br/>high query demand"| FAST
    CLASSIFY -->|"No — the other 99.9%"| SLOW

    subgraph FAST["⚡ FAST TRACK — streaming · seconds to minutes"]
        F1["Priority fetch<br/>bypasses normal queue"]
        F2["Streaming processing<br/>parse · dedupe · classify"]
        F3["Incremental signal computation<br/>approximate PageRank,<br/>partial anchor text"]
        F4["Write to in-memory<br/>real-time index segment"]
        F5["Immediately searchable"]
        F1 --> F2 --> F3 --> F4 --> F5
    end

    subgraph SLOW["🐢 BATCH TRACK — hours to days"]
        S1["Normal crawl queue"]
        S2["Batch processing"]
        S3["Full-precision signals<br/>exact PageRank over<br/>the complete graph"]
        S4["MapReduce index build"]
        S5["Segment publish"]
        S1 --> S2 --> S3 --> S4 --> S5
    end

    F5 --> SERVE
    S5 --> SERVE

    subgraph SERVE["🔍 Serving tier — queries both, merges results"]
        M1[("Real-time segments<br/>small · in RAM · minutes old")]
        M2[("Batch segments<br/>large · immutable · hours-days old")]
        M3["Query fans out to both,<br/>results merged and deduped"]
        M1 --> M3
        M2 --> M3
    end

    F5 -.eventually superseded by.-> S5

    classDef fast fill:#fecaca,stroke:#b91c1c,color:#1c1917
    classDef slow fill:#dbeafe,stroke:#1e40af,color:#1c1917
    classDef serve fill:#bbf7d0,stroke:#15803d,color:#1c1917
    class F1,F2,F3,F4,F5 fast
    class S1,S2,S3,S4,S5 slow
    class M1,M2,M3 serve
```

**The key structural idea: the fast track trades accuracy for latency, and the batch track corrects it later.** A document indexed via the fast path has approximate ranking signals — PageRank computed from a partial graph, anchor text from whatever links happen to be known. It is *findable* within minutes but not perfectly *ranked*. When the batch pipeline catches up hours later, the accurate version supersedes it. Users get a fast, slightly-wrong answer immediately rather than a perfect answer tomorrow, which for breaking news is unambiguously the right trade.

---

## 11.2 Incremental processing — the Percolator model

Batch processing recomputes everything. Incremental processing computes only what changed, and propagates consequences. The mechanism is **observers**: triggers that fire when a specific column of a specific row changes.

> **Diagram D-56 — Observer-driven incremental indexing**

```mermaid
flowchart TB
    W["Write: crawler stores<br/>new content for URL X"]

    W --> T1[("Crawl table<br/>row: com.example/X<br/>column: contents:html ← NEW")]

    T1 -->|"triggers"| O1["👁️ Observer: DocumentChanged"]

    O1 --> A1["Recompute content hash + SimHash"]
    A1 --> D{"Actually different<br/>from previous version?"}
    D -->|"No — 304 or identical"| STOP["🛑 Stop.<br/>No downstream work.<br/>This early exit is what<br/>makes incremental viable."]
    D -->|"Yes"| A2["Reparse: extract text, links, metadata"]

    A2 --> T2[("Write: contents:text,<br/>metadata:*, outlinks:*")]

    T2 -->|"triggers"| O2["👁️ Observer: LinksChanged"]
    O2 --> A3["Diff old vs new outlink set"]
    A3 --> A4["For each ADDED link → write<br/>an inlink record on the target row"]
    A3 --> A5["For each REMOVED link → delete<br/>the inlink record on the target row"]

    A4 & A5 --> T3[("Link graph rows updated<br/>on OTHER documents")]

    T3 -->|"triggers"| O3["👁️ Observer: InlinksChanged"]
    O3 --> A6["Update aggregated anchor text"]
    A6 --> A7["Mark target's PageRank as stale<br/>⚠️ do NOT recompute globally —<br/>queue for the next batch pass"]

    T2 -->|"triggers"| O4["👁️ Observer: ReadyToIndex"]
    O4 --> A8["Emit index update:<br/>docid, tokens, positions, signals"]
    A8 --> T4[("Real-time index segment")]

    T4 --> SERVE(("Searchable<br/>within minutes"))

    subgraph TX["Why this needs transactions"]
        X1["An observer reads several rows<br/>and writes several rows."]
        X2["Two observers firing concurrently<br/>on the same target row would<br/>corrupt the inlink set."]
        X3["→ Percolator adds multi-row<br/>snapshot-isolation transactions<br/>ON TOP of the wide-column store,<br/>using a timestamp oracle."]
        X1 --> X2 --> X3
    end

    O2 -.requires.-> TX

    classDef obs fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef store fill:#e9d5ff,stroke:#7e22ce,color:#1c1917
    classDef stop fill:#bbf7d0,stroke:#15803d,color:#1c1917
    class O1,O2,O3,O4 obs
    class T1,T2,T3,T4 store
    class STOP stop
```

### Why this is genuinely hard

The wide-column store from [Chapter 09](09-storage.md) deliberately offers **only single-row atomicity**. But an observer that adds an inlink must read the target row, modify it, and write it back — and two observers doing this concurrently on a popular target will lose updates.

Percolator's contribution was adding **multi-row snapshot-isolation transactions on top of** a Bigtable-class store, using a centralised timestamp oracle and two-phase commit encoded in extra columns. The cost is significant: a transactional write is several times more expensive than a plain one. The benefit is that incremental indexing becomes *correct*, which is what makes it usable in production rather than a source of subtly corrupted link data.

**The `STOP` node deserves attention too.** The early exit when content is unchanged is not an optimisation detail — it is the property that makes the whole approach viable. Most recrawls find nothing new ([Chapter 04](04-crawling.md)), so most observer chains terminate at the first step. Without that early exit, incremental processing would do batch-scale work at streaming-scale frequency.

---

## 11.3 Discovery: push beats pull for freshness

Waiting to *discover* a change by recrawling is the slowest possible path. Publishers who want to be indexed quickly can tell you directly.

> **Diagram D-57 — Change discovery channels, ranked by latency**

```mermaid
flowchart LR
    subgraph PUSH["📤 PUSH — publisher-initiated · seconds"]
        P1["Indexing API submission<br/>latency: seconds<br/>⚠️ requires site verification,<br/>strict quotas, abuse-prone"]
        P2["Sitemap ping on update<br/>latency: seconds–minutes"]
        P3["PubSubHubbub / WebSub<br/>real-time feed push"]
        P4["RSS / Atom polling<br/>latency: minutes"]
    end

    subgraph SIGNAL["📡 SIGNAL-DRIVEN · minutes"]
        S1["Query demand spike:<br/>many users searching a term<br/>with no good results<br/>→ targeted crawl"]
        S2["Social / external mention<br/>volume spike"]
        S3["A high-authority page<br/>links to something new"]
    end

    subgraph PULL["📥 PULL — crawler-initiated · minutes–weeks"]
        L1["High-frequency recrawl<br/>of known hot pages<br/>latency: minutes"]
        L2["Sitemap periodic re-read<br/>latency: hours"]
        L3["Normal scheduled recrawl<br/>latency: days–weeks"]
        L4["Link discovery from<br/>other crawled pages<br/>latency: unbounded"]
    end

    P1 & P2 & P3 & P4 --> VERIFY["Verification layer<br/>⚠️ push channels are attacker-controlled"]
    S1 & S2 & S3 --> VERIFY
    L1 & L2 & L3 & L4 --> VERIFY

    VERIFY --> V1["Confirm site ownership"]
    VERIFY --> V2["Enforce per-site quotas"]
    VERIFY --> V3["Fetch and verify the content<br/>actually changed — never trust<br/>the claim alone"]
    VERIFY --> V4["Apply spam and quality scoring<br/>BEFORE fast-track admission"]

    V1 & V2 & V3 & V4 --> FT["Fast-track queue"]

    classDef push fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef sig fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef pull fill:#dbeafe,stroke:#1e40af,color:#1c1917
    classDef sec fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class P1,P2,P3,P4 push
    class S1,S2,S3 sig
    class L1,L2,L3,L4 pull
    class VERIFY,V1,V2,V3,V4 sec
```

**Every push channel is an attack surface.** A "notify me when you update" API is, from a spammer's perspective, a way to jump the crawl queue. The verification layer is therefore not optional plumbing: ownership must be proven, quotas must be enforced per verified site, the claimed change must be confirmed by actually fetching the page, and quality scoring must run *before* the document enters the fast path — not after. A fast track without a verification gate is simply a fast track for spam.

**The signal-driven channel is the clever one.** If thousands of users suddenly search for a term and the index has no good results, that is direct evidence the corpus is missing something important right now. Turning unmet query demand into crawl priority closes the loop from [Diagram D-01](../../README.md) — user behaviour steering the crawler — and it catches events no publisher thought to announce.

---

## 11.4 Serving a corpus split across two indexes

The serving tier must query batch segments and real-time segments together and produce one coherent ranking.

> **Diagram D-58 — Merging real-time and batch results**

```mermaid
sequenceDiagram
    autonumber
    participant R as Root
    participant L as Leaf server
    participant B as Batch segments
    participant RT as Real-time segments
    participant M as Merge logic

    R->>L: query, top-k
    par Search both segment sets
        L->>B: traverse batch postings<br/>(large, compressed, immutable)
        L->>RT: traverse real-time postings<br/>(small, in RAM, recent)
    end
    B-->>L: candidates with full-precision signals
    RT-->>L: candidates with approximate signals

    L->>M: merge

    M->>M: 1️⃣ Deduplicate by docid<br/>a doc may exist in BOTH —<br/>the real-time version is newer
    Note over M: Rule — newest version wins.<br/>Batch segments carry a watermark<br/>timestamp, and RT entries older<br/>than it are discarded.

    M->>M: 2️⃣ Reconcile score scales<br/>⚠️ approximate PageRank is not<br/>comparable to exact PageRank
    Note over M: Fix: calibrate RT signals against<br/>batch distribution, or apply an<br/>explicit freshness prior instead of<br/>pretending the scores are equivalent.

    M->>M: 3️⃣ Apply tombstones<br/>deletions from either source

    M->>M: 4️⃣ Apply freshness boost<br/>only if the query has fresh intent

    M-->>R: unified top-k + coverage flags

    Note over L,RT: Compaction, continuously:<br/>real-time segments are folded<br/>into batch segments and dropped,<br/>keeping the RT set small enough<br/>to stay in RAM.
```

**Step 2 is where naïve implementations break.** A document in the real-time index has an approximate PageRank computed from a partial link graph — typically an *underestimate*, since most of its inbound links have not been discovered yet. If you merge those scores directly with exact batch scores, fresh documents are systematically penalised, and your expensive freshness pipeline produces content nobody sees.

The correct handling is to treat the two score distributions as different and calibrate between them, or — more robustly — to keep freshness as an *explicit* ranking prior applied by the ranker for freshness-intent queries, rather than hoping it emerges implicitly from mixed-precision signals.

---

## 11.5 The freshness/cost trade-off

Freshness is not free, and its cost is not evenly distributed.

| Corpus slice | Share | Recrawl interval | Path | Relative cost per doc |
|---|---:|---|---|---:|
| Breaking news, live feeds | ~0.001 % | seconds–minutes | Push + fast track | **~1000×** |
| Active news sites, forums | ~0.1 % | minutes–hours | Fast track | ~100× |
| Frequently updated sites | ~1 % | hours–daily | Mixed | ~10× |
| Ordinary web pages | ~20 % | weekly–monthly | Batch | 1× |
| Static archives, PDFs | ~79 % | quarterly or never | Batch, low priority | ~0.1× |

**Concentrating almost all freshness spending on ~0.1 % of the corpus is the entire strategy.** Applying fast-track treatment uniformly would multiply indexing cost by roughly two orders of magnitude while improving results for almost nobody, because the other 99.9 % of documents were not going to change anyway.

This is the same shape as the tiering decision in [Chapter 06](06-indexing.md) and the caching decision in [Chapter 10](10-caching.md), and it is worth naming explicitly as a recurring pattern:

> **Web-scale systems are made affordable by exploiting extreme skew. Identify the small subset that matters, spend lavishly on it, and treat the rest cheaply and in bulk.**

Every major cost decision in this repository is an instance of that one idea.

---

<div align="center">

**← [10 · Caching](10-caching.md)** · [🏠 Home](../../README.md) · **[12 · Reliability](12-reliability.md) →** · [🇸🇦 العربية](../ar/11-freshness.md)

</div>
