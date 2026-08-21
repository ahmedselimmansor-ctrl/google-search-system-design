<div align="right">

**[🇸🇦 اقرأ بالعربية](../ar/06-indexing.md)**

</div>

# Chapter 06 · Indexing & the Inverted Index

**← [05 · Content Processing](05-content-processing.md)** · [🏠 Home](../../README.md) · **[07 · Serving](07-serving.md) →**

---

## 6.1 The one data structure that makes search possible

Everything in [Chapter 01](01-overview.md) reduced to a single idea: **do not scan documents, scan a term-keyed structure**. That structure is the inverted index, and it is the reason a query over 10¹¹ documents can finish in 40 milliseconds.

> **Diagram D-26 — Anatomy of the inverted index**

```mermaid
flowchart LR
    subgraph FWD["Forward view — what we crawled"]
        D1["doc 17<br/>'the quick brown fox'"]
        D2["doc 42<br/>'quick brown dogs run'"]
        D3["doc 99<br/>'the fox and the hound'"]
    end

    FWD -->|"INVERT"| INV

    subgraph INV["Inverted view — what we query"]
        direction TB
        L["📖 Lexicon / term dictionary<br/>term → df, pointer, stats"]

        T1["'brown' · df=2"] --> P1["→ 17:[3] · 42:[2]"]
        T2["'fox' · df=2"] --> P2["→ 17:[4] · 99:[2]"]
        T3["'quick' · df=2"] --> P3["→ 17:[2] · 42:[1]"]
        T4["'hound' · df=1"] --> P4["→ 99:[5]"]

        L --- T1
        L --- T2
        L --- T3
        L --- T4
    end

    INV --> Q["Query: 'quick brown'<br/>intersect P3 ∩ P1<br/>= {17, 42}<br/><br/>Phrase check: positions<br/>doc 17: quick@2, brown@3 ✅ adjacent<br/>doc 42: quick@1, brown@2 ✅ adjacent"]

    classDef fwd fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef inv fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef res fill:#bbf7d0,stroke:#15803d,color:#1c1917
    class D1,D2,D3 fwd
    class T1,T2,T3,T4,P1,P2,P3,P4,L inv
    class Q res
```

### The three components

| Component | Contents | Size (from [Ch 03](03-capacity-estimation.md)) |
|---|---|---:|
| **Lexicon** | Every distinct term → document frequency, pointer into postings, global statistics | ~40 GB |
| **Postings** | For each term, the sorted list of documents containing it, with frequencies and positions | ~225 TB |
| **Forward index / attachments** | Per-document data needed *after* retrieval: text for snippets, precomputed features, vectors | ~350 TB |

The lexicon is small enough to keep entirely in RAM on every leaf server. The postings are the bulk. The forward index is only consulted for the few documents that survive ranking — which is why it can live on slower storage.

### What a posting actually contains

```
posting = ⟨ docid_delta, term_frequency, field_mask, position_deltas[] ⟩

  docid_delta      gap from the previous docid in this list  (variable-byte)
  term_frequency   occurrences of this term in this document
  field_mask       bitmask: title | h1 | body | anchor | alt | url
  position_deltas  gaps between successive positions          (variable-byte)
```

The `field_mask` is disproportionately valuable: it lets the scorer know *where* a term appeared without touching the document, so a title match can be weighted far above a body match during the cheap L1 scoring pass.

---

## 6.2 Compression — where the money is

[Chapter 03](03-capacity-estimation.md) established that RAM is the dominant cost in the system. Index compression is therefore not an optimisation; it is the economics of the product. Every 10 % reduction in index size is roughly a 10 % reduction in the serving fleet.

> **Diagram D-27 — Posting list compression pipeline**

```mermaid
flowchart TB
    RAW["Raw docids for term 'search'<br/>1,247 · 1,251 · 1,299 · 1,300 · 1,847 …<br/>8 bytes each = 64 bits/posting"]

    RAW --> SORT["① Sort ascending<br/>required for gap coding<br/>and for fast intersection"]
    SORT --> DELTA["② Delta encode<br/>1,247 · 4 · 48 · 1 · 547 …<br/>gaps are small numbers"]

    DELTA --> CHOICE{"③ Choose codec"}

    CHOICE -->|"Simple, byte-aligned"| VB["Variable-byte<br/>7 data bits + 1 continuation bit<br/>small gaps → 1 byte<br/>fast to decode"]
    CHOICE -->|"Best ratio, bit-aligned"| GOL["Golomb / Rice<br/>optimal for geometric gaps<br/>slower decode"]
    CHOICE -->|"SIMD-friendly, modern"| PFOR["PForDelta / SIMD-BP128<br/>pack blocks of 128 at a<br/>common bit-width<br/>exceptions stored separately"]

    VB & GOL & PFOR --> SKIP["④ Add skip pointers<br/>every √n postings<br/>enables O(√n) seek<br/>during intersection"]

    SKIP --> BLOCK["⑤ Block structure<br/>128 postings per block<br/>with block max-score for<br/>WAND / BlockMax pruning"]

    BLOCK --> OUT["≈ 1.5 bytes per positional posting<br/><br/>64 bits → 12 bits<br/>≈ 5× reduction"]

    subgraph EXTRA["Additional levers"]
        E1["Docid reordering:<br/>assign nearby ids to similar docs<br/>→ smaller gaps → 10-20% further gain"]
        E2["Frequency capping:<br/>tf above ~255 adds no ranking value"]
        E3["Position pruning:<br/>drop positions in tier 2,<br/>lose phrase queries, save 60%"]
        E4["Stopword handling:<br/>keep but store cheaply —<br/>deleting breaks phrase search"]
    end

    OUT --> EXTRA

    classDef step fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef codec fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef win fill:#bbf7d0,stroke:#15803d,color:#1c1917
    class SORT,DELTA,SKIP,BLOCK step
    class VB,GOL,PFOR codec
    class OUT win
```

**On stopwords.** The classic textbook advice is to delete `the`, `a`, `of` from the index because they appear in nearly every document. Modern web search does *not* do this, because it breaks phrase and quoted queries — `"to be or not to be"` becomes unanswerable. Instead the high-frequency terms are kept but stored in cheaper structures, and query processing simply avoids leading with them.

**Docid reordering** deserves emphasis because it is nearly free. If document IDs are assigned so that similar documents (same host, same topic, near-duplicate cluster) receive numerically adjacent IDs, then the gaps within each posting list become smaller, and smaller gaps compress better. Sorting documents by URL before assigning IDs — so all pages of one site are contiguous — typically yields a further 10–20 % size reduction for essentially zero engineering cost.

---

## 6.3 Sharding: the decision that defines the serving architecture

An index of 600 TB cannot live on one machine. How you split it determines everything about the query path.

> **Diagram D-28 — Document partitioning vs term partitioning**

```mermaid
flowchart TB
    subgraph DOCP["✅ Document partitioning — the standard choice"]
        direction TB
        DP["Each shard holds ALL terms<br/>for a SUBSET of documents"]
        DPS1[("Shard 1<br/>docs 0–10⁸<br/>full lexicon")]
        DPS2[("Shard 2<br/>docs 10⁸–2×10⁸<br/>full lexicon")]
        DPS3[("Shard N<br/>…<br/>full lexicon")]
        DP --> DPS1 & DPS2 & DPS3
        DPQ["Query → ALL shards in parallel<br/>each returns its local top-K<br/>root merges N × K candidates"]
        DPS1 & DPS2 & DPS3 --> DPQ
    end

    subgraph TERMP["⚠️ Term partitioning — rarely used at web scale"]
        direction TB
        TP["Each shard holds a SUBSET of terms<br/>for ALL documents"]
        TPS1[("Shard A<br/>terms a–f<br/>all 10¹¹ docs")]
        TPS2[("Shard B<br/>terms g–p<br/>all 10¹¹ docs")]
        TPS3[("Shard C<br/>terms q–z<br/>all 10¹¹ docs")]
        TP --> TPS1 & TPS2 & TPS3
        TPQ["Query → only shards owning<br/>the query's terms<br/>but huge posting lists must<br/>cross the network to intersect"]
        TPS1 & TPS2 & TPS3 --> TPQ
    end

    classDef good fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef bad fill:#fed7aa,stroke:#c2410c,color:#1c1917
    class DP,DPS1,DPS2,DPS3,DPQ good
    class TP,TPS1,TPS2,TPS3,TPQ bad
```

| Property | Document partitioning | Term partitioning |
|---|---|---|
| Servers contacted per query | **All** (high fan-out) | Only those owning query terms (low fan-out) |
| Network volume per query | Small — top-K results only | **Huge** — full posting lists must be shipped to intersect |
| Load balance | Even — every shard does similar work | **Terrible** — "the" shard is melted, "xylophone" shard idles |
| Adding documents | Trivial — write to any shard, or add a shard | Requires touching many shards |
| Failure of one shard | Lose a slice of the corpus, results degrade gracefully | Lose entire terms — some queries become unanswerable |
| Ranking quality | Global stats (IDF) need a small side-channel | Global stats are local and exact |
| Scoring locality | **Excellent** — all data for a doc is co-located | Poor — a document's terms are spread across shards |

**Document partitioning wins decisively**, and the deciding factor is the third row. Term frequencies on the web follow a Zipf distribution, so a term-partitioned index has a permanent, unfixable hot-spotting problem. Document partitioning gives every shard statistically identical work.

The cost you accept is total fan-out: every query touches every shard, which creates the tail-latency problem that [Chapter 07](07-serving.md) and [Chapter 12](12-reliability.md) spend their entire budgets mitigating. That is a good trade — tail latency is *manageable* with hedging and timeouts, whereas Zipf hot-spotting is not manageable at all.

### One global statistic must be shared

Document partitioning has one genuine flaw: **IDF (inverse document frequency) is a global quantity**, but each shard only sees its own documents. If shards computed IDF locally, the same term would be scored differently on different shards and the merged ranking would be incoherent.

The fix is cheap: periodically compute global `df` for every term in a batch job and broadcast a compact table (~40 GB — the lexicon) to every leaf. IDF changes slowly, so an hourly or daily refresh is more than adequate.

---

## 6.4 Building the index

> **Diagram D-29 — Index construction pipeline**

```mermaid
flowchart TB
    SRC[("Processed documents<br/>from Chapter 05")]
    LINKS[("Link graph")]

    subgraph OFFLINE["Offline signal computation"]
        PR["PageRank<br/>~30 iterations over 10¹² edges"]
        ANC["Anchor text aggregation<br/>group inlinks by target"]
        QS["Quality & spam scores"]
        EMB["Embedding generation<br/>doc → dense vector"]
    end

    LINKS --> PR
    LINKS --> ANC
    SRC --> QS
    SRC --> EMB

    subgraph MAP["MAP phase"]
        M1["For each document:<br/>emit ⟨term, docid, tf, fields, positions⟩<br/>for every token"]
        M2["Attach precomputed<br/>per-doc signals"]
    end

    SRC --> M1
    PR & ANC & QS --> M2
    M1 --> M2

    M2 --> SHUF["SHUFFLE<br/>partition by hash(term)<br/>sort by ⟨term, docid⟩<br/>— the expensive step:<br/>10¹⁴ records across the network"]

    subgraph RED["REDUCE phase"]
        R1["Group all postings for one term"]
        R2["Delta-encode + compress"]
        R3["Build skip pointers<br/>and block max-scores"]
        R4["Emit immutable index segment"]
    end

    SHUF --> R1 --> R2 --> R3 --> R4

    R4 --> SEG[("Immutable index segments<br/>on distributed file system")]
    EMB --> VEC[("Vector index<br/>ANN graph / IVF-PQ")]

    SEG --> ASSIGN["Shard assembler<br/>group segments into shards<br/>assign shards to tiers"]
    VEC --> ASSIGN

    ASSIGN --> PUB{"Publish"}
    PUB --> VALID["Validation gate<br/>golden-query regression suite<br/>size and count sanity checks"]
    VALID -->|"Pass"| SHIP["Ship to serving replicas<br/>staged, region by region"]
    VALID -->|"Fail"| HALT["🛑 Halt publish<br/>keep serving previous version"]

    SHIP --> LIVE(("Live serving tier"))

    classDef off fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef mr fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef store fill:#e9d5ff,stroke:#7e22ce,color:#1c1917
    classDef gate fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class PR,ANC,QS,EMB off
    class M1,M2,SHUF,R1,R2,R3,R4 mr
    class SEG,VEC store
    class VALID,HALT gate
```

**The shuffle is the whole cost of index building.** Map and reduce are embarrassingly parallel and cheap. Moving 10¹⁴ ⟨term, docid⟩ records across the network so that all postings for one term land on one reducer is what takes the hours. This is precisely the workload MapReduce was designed for, and it is not a coincidence that the paper came out of a search company.

**The validation gate is not optional.** Publishing a broken index to the serving tier is one of the few genuinely catastrophic failure modes in the system — it degrades results globally, and the corpus is too large to inspect by hand. A fixed suite of golden queries with known-good results runs against every candidate index, and any regression halts the publish while the previous version keeps serving.

---

## 6.5 Segments, merging and immutability

Index segments are **immutable once written**. Updates never modify a segment; they create new ones. This is the log-structured merge (LSM) discipline, and it buys several properties at once:

- Readers need no locks — a segment being read can never change underneath them.
- Writes are sequential appends, which is the only access pattern that is fast on every storage medium.
- Replication is trivial: an immutable file can be copied anywhere and cached forever.
- Rollback is trivial: keep the previous segment set and switch a pointer.

The cost is **read amplification** — a query must consult every live segment — and this is what merging repays.

> **Diagram D-30 — Segment lifecycle and tiered merging**

```mermaid
stateDiagram-v2
    [*] --> InMemory: new/updated docs buffered

    InMemory --> Flushed: buffer full or timer fires
    note right of InMemory
        Small in-RAM segment.
        Searchable immediately —
        this is what makes
        near-real-time possible.
    end note

    Flushed --> L0: written as immutable segment
    L0 --> L0: more flushes accumulate

    L0 --> Merging: L0 segment count > threshold
    Merging --> L1: merged, sorted, recompressed
    L1 --> Merging2: L1 size > threshold
    Merging2 --> L2: larger merged segment
    L2 --> Merging3: periodic major merge
    Merging3 --> L3: full compaction

    L3 --> Published: passes validation gate
    Published --> Serving: shipped to replicas
    Serving --> Superseded: newer version published
    Superseded --> Deleted: after rollback window expires
    Deleted --> [*]

    Serving --> Rollback: regression detected
    Rollback --> Serving: previous version restored

    note right of Merging3
        Major merge also applies
        deletions: docs marked
        dead in a tombstone list
        are physically dropped here.
    end note
```

**Deletions are tombstones, not edits.** Removing a document writes a marker into a delete list; the document keeps occupying index space until the next major merge physically drops it. Queries filter tombstoned docids at read time. This is the only way to handle deletion in an immutable structure, and it means "delete" has a latency of *seconds* (filtered from results) but "reclaim the space" has a latency of *hours* (next merge). For legal removals, the first latency is what matters — and it is satisfied.

---

## 6.6 Index tiering

[Chapter 03](03-capacity-estimation.md) showed tiering is worth roughly two orders of magnitude of fleet cost. Here is the mechanism.

> **Diagram D-31 — Tiered index and fallthrough**

```mermaid
flowchart TB
    Q["Incoming query"] --> T0

    subgraph T0["🔥 Tier 0 — hot"]
        T0D["10⁹ docs · 1% of corpus<br/>~6 TB · RAM-resident<br/>600 replicas<br/>full positions, all signals"]
    end

    T0 --> C0{"Enough high-quality<br/>candidates?<br/>count ≥ K and<br/>min score ≥ θ"}
    C0 -->|"Yes ~85%"| DONE0["✅ Serve from Tier 0<br/>latency ~40 ms"]
    C0 -->|"No"| T1

    subgraph T1["⚡ Tier 1 — warm"]
        T1D["10¹⁰ docs · 10% of corpus<br/>~60 TB · RAM + NVMe<br/>90 replicas<br/>full positions"]
    end

    T1 --> C1{"Enough now?"}
    C1 -->|"Yes ~12%"| DONE1["✅ Serve from T0 + T1<br/>latency ~80 ms"]
    C1 -->|"No"| T2

    subgraph T2["🧊 Tier 2 — cold"]
        T2D["8.9 × 10¹⁰ docs · 89%<br/>~534 TB · NVMe<br/>36 replicas<br/>positions pruned to save space"]
    end

    T2 --> DONE2["✅ Serve from all tiers<br/>latency ~200 ms<br/>rare, long-tail queries"]

    subgraph ASSIGN["What decides a document's tier"]
        A1["Query coverage — does it ever get retrieved?"]
        A2["PageRank / authority"]
        A3["Click-through history"]
        A4["Freshness and change rate"]
        A5["Quality and spam scores"]
        A6["Language and locale demand"]
    end

    ASSIGN -.recomputed daily.-> T0D
    ASSIGN -.-> T1D
    ASSIGN -.-> T2D

    classDef hot fill:#fecaca,stroke:#b91c1c,color:#1c1917
    classDef warm fill:#fed7aa,stroke:#c2410c,color:#1c1917
    classDef cold fill:#dbeafe,stroke:#1e40af,color:#1c1917
    classDef ok fill:#bbf7d0,stroke:#15803d,color:#1c1917
    class T0D hot
    class T1D warm
    class T2D cold
    class DONE0,DONE1,DONE2 ok
```

**Tiering is a bet, and the bet has a cost.** The fallthrough condition (`count ≥ K and min score ≥ θ`) is a heuristic. When it fires incorrectly, the system serves a Tier-0-only result set for a query whose best answer was in Tier 2 — a silent relevance failure that no latency metric will reveal. Tuning θ is a direct trade of cost against quality on rare queries, and it must be monitored with the human-rated quality metrics from [Chapter 02](02-requirements.md), never with latency or CTR alone.

---

## 6.7 The complete index data model

> **Diagram D-32 — Index structures**

```mermaid
classDiagram
    class Lexicon {
        +HashMap~term, TermMeta~ terms
        +int64 total_docs
        +lookup(term) TermMeta
        +globalIDF(term) float
    }

    class TermMeta {
        +int64 doc_frequency
        +int64 total_term_frequency
        +int64 postings_offset
        +int32 postings_length
        +float max_impact_score
        +int32 num_blocks
    }

    class PostingList {
        +int64 term_id
        +PostingBlock[] blocks
        +SkipList skips
        +iterator() PostingIterator
        +seek(docid) Posting
        +blockMaxScore(i) float
    }

    class PostingBlock {
        +int32 num_postings
        +int8 bit_width
        +bytes docid_deltas
        +bytes frequencies
        +bytes position_deltas
        +float max_score
        +decode() Posting[]
    }

    class Posting {
        +int64 docid
        +int16 term_frequency
        +int16 field_mask
        +int32[] positions
    }

    class ForwardIndex {
        +getDocument(docid) DocAttachments
        +getSnippetText(docid) string
    }

    class DocAttachments {
        +int64 docid
        +string url
        +string title
        +bytes compressed_text
        +float pagerank
        +float quality_score
        +float spam_score
        +string language
        +int64 last_modified
        +int8[] embedding
        +bytes precomputed_features
    }

    class IndexSegment {
        +string segment_id
        +int32 tier
        +int64 doc_count
        +int64 created_at
        +bool immutable
        +Lexicon lexicon
        +PostingList[] postings
        +ForwardIndex forward
        +BitSet tombstones
    }

    class Shard {
        +int32 shard_id
        +int64 docid_range_start
        +int64 docid_range_end
        +IndexSegment[] segments
        +search(query, k) Candidate[]
    }

    class VectorIndex {
        +int32 dimensions
        +ANNGraph graph
        +search(vector, k) Candidate[]
    }

    Lexicon "1" --> "*" TermMeta
    TermMeta "1" --> "1" PostingList
    PostingList "1" --> "*" PostingBlock
    PostingBlock "1" --> "*" Posting
    ForwardIndex "1" --> "*" DocAttachments
    IndexSegment "1" --> "1" Lexicon
    IndexSegment "1" --> "*" PostingList
    IndexSegment "1" --> "1" ForwardIndex
    Shard "1" --> "*" IndexSegment
    Shard "1" --> "0..1" VectorIndex
```

---

## 6.8 Query-time index traversal

Knowing the structure, here is how a leaf server actually uses it — and the optimisation that makes it fast.

**Naïve intersection** of two posting lists walks both fully: O(len(A) + len(B)). For a term like "the" that is 10¹⁰ postings. Unacceptable.

**Skip-pointer intersection** walks the shorter list and *seeks* in the longer one, using skip pointers to jump: O(len(shorter) × log). Much better.

**WAND / BlockMax-WAND** goes further and is what production systems use. Each block stores its maximum possible contribution to the score. During traversal the engine tracks the current *k*-th best score; any block whose maximum possible score cannot beat it is **skipped entirely without decompression**.

```
threshold θ ← score of current k-th best candidate

for each candidate block:
    upper_bound ← Σ blockMaxScore(term) over query terms
    if upper_bound ≤ θ:
        skip the whole block          ← no decompression, no scoring
    else:
        decode block, score documents, update θ
```

In practice this skips **80–95 %** of postings for a typical multi-term query, and it is exact — the top-k returned is identical to the result of full evaluation. It is a rare case of a large speedup with no quality cost, which is why block max-scores are stored at build time (see Diagram D-27, step ⑤).

---

## 6.9 Trade-offs recorded

| Decision | Chosen | Rejected | Why |
|---|---|---|---|
| Partitioning | By document | By term | Zipf hot-spotting is unfixable |
| Mutability | Immutable segments + merge | In-place updates | Lock-free reads, trivial replication and rollback |
| Compression | PForDelta/SIMD blocks | Golomb-Rice | Decode speed beats a few % of ratio when RAM-resident |
| Positions | Kept in tiers 0–1, pruned in tier 2 | Kept everywhere | Phrase queries are rare on tail documents |
| Stopwords | Kept, stored cheaply | Removed | Removal breaks quoted-phrase queries |
| Deletion | Tombstones + lazy merge | Immediate rewrite | Immutability is worth more than prompt space reclamation |
| Tiering | 3 tiers by query coverage | Flat index | ~100× fleet cost reduction |
| Docid assignment | Sorted by URL | Random | 10–20 % free compression gain |

---

<div align="center">

**← [05 · Content Processing](05-content-processing.md)** · [🏠 Home](../../README.md) · **[07 · Serving](07-serving.md) →** · [🇸🇦 العربية](../ar/06-indexing.md)

</div>
