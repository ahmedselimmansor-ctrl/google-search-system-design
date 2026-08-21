<div align="right">

**[🇸🇦 اقرأ بالعربية](../ar/07-serving.md)**

</div>

# Chapter 07 · The Query Serving Path

**← [06 · Indexing](06-indexing.md)** · [🏠 Home](../../README.md) · **[08 · Ranking](08-ranking.md) →**

---

## 7.1 The serving topology

Document partitioning ([Chapter 06](06-indexing.md)) means every query must reach every shard. With thousands of shards, a single root talking directly to all of them would need thousands of outbound RPCs per query — the root becomes a fan-out bottleneck and its own tail-latency amplifier.

The answer is a **tree**.

> **Diagram D-33 — Root, intermediate and leaf serving tree**

```mermaid
flowchart TB
    FE["Frontend<br/>1 RPC out"]
    FE --> MIX["Mixer<br/>orchestrates verticals"]

    MIX --> ROOT["Index Root<br/>~40 RPCs out"]

    ROOT --> I1["Intermediate 1<br/>~40 RPCs out"]
    ROOT --> I2["Intermediate 2"]
    ROOT --> I3["Intermediate …"]
    ROOT --> IN["Intermediate 40"]

    I1 --> L1["Leaf 1<br/>shard 1"]
    I1 --> L2["Leaf 2<br/>shard 2"]
    I1 --> L3["Leaf …"]
    I1 --> L40["Leaf 40<br/>shard 40"]

    I2 --> M1["Leaf 41 …"]
    I3 --> M2["Leaf … "]
    IN --> M3["Leaf 1600"]

    subgraph REPL["Each leaf is itself replicated"]
        R1["Replica A"]
        R2["Replica B"]
        R3["Replica C"]
    end

    L1 -.-> REPL

    subgraph FLOW["What flows back up"]
        F1["Leaf → Intermediate: top-K docids + scores<br/>K ≈ 100, a few KB"]
        F2["Intermediate → Root: merged top-K of its 40 leaves"]
        F3["Root → Mixer: global top-K, ~1000 candidates"]
    end

    classDef top fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef mid fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef leaf fill:#fde68a,stroke:#b45309,color:#1c1917
    class FE,MIX,ROOT top
    class I1,I2,I3,IN mid
    class L1,L2,L3,L40,M1,M2,M3 leaf
```

**Why a tree and not a flat fan-out?**

| Property | Flat (root → 1,600 leaves) | Tree (root → 40 → 1,600) |
|---|---|---|
| RPCs from the root | 1,600 | 40 |
| Merge work at the root | 1,600 × K results | 40 × K results |
| Root CPU | Bottleneck | Modest |
| Tail exposure at root | max of 1,600 latencies | max of 40 latencies (each already hedged below) |
| Extra hop latency | 0 | ~0.5 ms |

The tree costs one extra network hop (~0.5 ms out of a 300 ms budget) and buys a 40× reduction in fan-out at every level. Each intermediate absorbs the tail of its own 40 leaves locally, so a single slow leaf perturbs one subtree rather than the whole query.

---

## 7.2 Query understanding

Before any index is touched, the raw string must become a structured query. This stage is cheap in milliseconds and enormously expensive in relevance if done badly.

> **Diagram D-34 — Query understanding pipeline**

```mermaid
flowchart TB
    RAW["Raw query string<br/>e.g. 'wether in cairo tommorow'"]

    RAW --> SAN["Sanitize<br/>trim · Unicode NFC · strip control chars"]
    SAN --> LANGD["Language & script detection<br/>+ user locale signals"]
    LANGD --> NORMZ["Language-specific normalization<br/>see Chapter 05"]

    NORMZ --> TOKQ["Tokenize<br/>same analyzer as index — critical!"]

    TOKQ --> PARSE["Operator parsing<br/>quotes · site: · filetype: · -term · OR"]

    PARSE --> SPELL{"Spelling"}
    SPELL -->|"confident"| AUTOC["Auto-correct<br/>'wether' → 'weather'<br/>show 'showing results for…'"]
    SPELL -->|"uncertain"| SUGG["Offer 'did you mean'<br/>serve original"]
    SPELL -->|"clean"| PASS["Pass through"]

    AUTOC & SUGG & PASS --> INTENT["Intent classification"]

    INTENT --> I1["Navigational<br/>'facebook login'"]
    INTENT --> I2["Informational<br/>'how does pagerank work'"]
    INTENT --> I3["Transactional<br/>'buy usb-c cable'"]
    INTENT --> I4["Local<br/>'coffee near me'"]
    INTENT --> I5["Fresh / news<br/>'election results'"]

    I1 & I2 & I3 & I4 & I5 --> EXPAND["Query expansion"]
    EXPAND --> E1["Synonyms: 'car' ≈ 'automobile'"]
    EXPAND --> E2["Stemming variants"]
    EXPAND --> E3["Entity linking: 'cairo' → Cairo, Egypt"]
    EXPAND --> E4["Implicit terms from context"]

    E1 & E2 & E3 & E4 --> WEIGHT["Term weighting<br/>which terms are essential<br/>vs droppable"]

    WEIGHT --> EMB["Dense query embedding<br/>for vector retrieval"]

    WEIGHT --> PLAN["Structured query plan"]
    EMB --> PLAN

    PLAN --> OUT(("→ Cache lookup,<br/>then index fan-out"))

    classDef stage fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef intent fill:#fde68a,stroke:#b45309,color:#1c1917
    class SAN,LANGD,NORMZ,TOKQ,PARSE,EXPAND,WEIGHT,EMB stage
    class I1,I2,I3,I4,I5 intent
```

### The single most important rule in this chapter

> **The query analyzer and the index analyzer must be the same code.**

If the index lowercased and stemmed but the query path did not, `Running` will never match the indexed token `run`. This class of bug is silent — no error is thrown, results are merely and permanently worse. Every mature search system shares one analyzer implementation between build and serve, and treats any divergence as a build-breaking error.

### Intent shapes everything downstream

| Intent | Retrieval emphasis | Freshness weight | Result presentation |
|---|---|---|---|
| Navigational | One exact target; precision@1 is all that matters | Low | Single dominant result, sitelinks |
| Informational | Broad recall, diverse sources | Medium | Ten blue links, possibly a featured snippet |
| Transactional | Product corpus, availability, price | Medium | Shopping unit, reviews |
| Local | Geo-filtered corpus | High | Map pack, hours, distance |
| Fresh / news | Recency dominates authority | **Very high** | Top stories carousel, timestamps |

A ranking function tuned for informational queries will do badly on navigational ones (it will spread probability across many "relevant" pages when the user wanted exactly one). This is why intent classification precedes ranking rather than being folded into it.

---

## 7.3 The full serving sequence, with tail-latency defences

> **Diagram D-35 — Serving path including hedging and degradation**

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant FE as Frontend
    participant QU as Query Understanding
    participant CA as Result Cache
    participant MX as Mixer
    participant RT as Root
    participant IM as Intermediate
    participant LF as Leaf
    participant RK as Ranking L2/L3
    participant DS as Doc Store
    participant SN as Snippets

    C->>FE: query
    FE->>FE: assign request id, start deadline clock
    FE->>QU: parse & understand
    QU-->>FE: structured plan
    FE->>CA: lookup(normalized_query, locale, safesearch)

    alt Cache hit and fresh
        CA-->>FE: cached SERP
        FE-->>C: results ✅ ~15 ms
    else Cache miss
        FE->>MX: execute plan (deadline: 250 ms)
        MX->>RT: retrieve(top 1000)

        par Fan-out with per-level deadlines
            RT->>IM: sub-query, deadline 90 ms
            IM->>LF: sub-query, deadline 70 ms
        end

        LF->>LF: WAND traversal + L1 scoring
        LF-->>IM: top-100 docids + scores

        Note over IM,LF: ⏱️ Hedged request:<br/>if a leaf has not replied by p95,<br/>send the same query to another<br/>replica and take the first answer.

        alt Leaf times out entirely
            IM->>IM: proceed without that shard
            IM-->>RT: partial results + coverage flag
            Note over IM,RT: Degrade, never fail.<br/>99.5% coverage still<br/>produces a good SERP.
        end

        IM-->>RT: merged top-K
        RT-->>MX: global candidates (~1000)

        MX->>RK: L2 rescore 1000 → 100
        RK->>RK: L3 neural rerank 100 → 10
        RK-->>MX: final order

        par Snippets in parallel
            MX->>DS: fetch attachments for top 10
            DS-->>MX: text, title, metadata
            MX->>SN: generate query-biased snippets
            SN-->>MX: snippets
        end

        MX->>MX: blend verticals, host dedup, spam filter
        MX->>CA: store with TTL by intent
        MX-->>FE: SERP
        FE-->>C: results ✅ ~210 ms
    end

    C->>FE: click (logged async)
    FE-)RK: training signal
```

### The three tail-latency defences

From [Chapter 02](02-requirements.md): with 1,600 leaves and a 1 % chance each is slow, ~99.99 % of queries touch at least one slow server. Three techniques, all from Dean & Barroso's *The Tail at Scale*, keep that from becoming user-visible.

| Technique | Mechanism | Cost |
|---|---|---|
| **Hedged requests** | After waiting for the p95 latency, send a duplicate to another replica; use whichever returns first | ~5 % extra load |
| **Tied requests** | Send to two replicas immediately; each cancels the other's queued copy when it starts executing | Slightly more messaging, near-zero wasted work |
| **Deadline propagation** | Every hop passes down its remaining budget; a leaf that cannot finish in time returns partial results rather than blowing the budget | Free — just plumbing |

Plus the architectural defence: **coverage degradation**. If a shard does not answer, the query proceeds without it and the response is annotated with the coverage fraction. Serving 99.5 % of the corpus in 200 ms is a far better product than serving 100 % in 3 seconds — and the alternative, returning an error, is the worst outcome of all.

---

## 7.4 Where the milliseconds go

> **Diagram D-36 — Latency budget timeline (cache miss, p99 path)**

```mermaid
gantt
    title Query latency budget — cache miss (units: milliseconds)
    dateFormat X
    axisFormat %s

    section Network
    Client to edge (TLS resumed)      :net1, 0, 20
    Edge to serving cell              :net2, 20, 25

    section Frontend
    Request setup and routing         :fe1, 25, 33
    Query understanding               :qu1, 33, 53
    Cache lookup (miss)               :ca1, 53, 55

    section Retrieval
    Root to intermediate to leaf      :rt1, 55, 58
    Leaf WAND traversal and L1 score  :lf1, 58, 98
    Merge up the tree                 :mg1, 98, 105

    section Ranking
    L2 rescore 1000 to 100            :l2, 105, 135
    L3 neural rerank 100 to 10        :l3, 135, 185

    section Assembly
    Doc store fetch (parallel)        :ds1, 185, 205
    Snippet generation                :sn1, 205, 215
    Blend, dedupe, serialize          :bl1, 215, 230

    section Return
    Response to client                :ret, 230, 250
    Tail headroom to p99 SLO          :crit, head, 250, 300
```

Read this as a budget you are *spending*. Two structural facts fall out of it:

1. **Retrieval and ranking together are ~50 % of the budget** (55 ms → 185 ms). Everything else is overhead you minimise so that this half can be as good as possible.
2. **The 50 ms of headroom at the end is not slack — it is the tail.** p50 finishes near 210 ms; p99 uses the headroom. Spending it on features means missing the SLO for 1 % of queries.

---

## 7.5 Universal search — blending the verticals

Modern SERPs are not one ranked list. They interleave web results with images, news, video, maps, shopping and direct answers. Each vertical is a separate retrieval system with its own index and ranking, and the mixer must decide *which* to show, *where*, and *how much space* each gets.

> **Diagram D-37 — Vertical triggering and blending**

```mermaid
flowchart TB
    Q["Structured query + intent"] --> TRIG["Trigger evaluation<br/>one classifier per vertical"]

    TRIG --> V1{"Web<br/>always on"}
    TRIG --> V2{"Images?"}
    TRIG --> V3{"News?"}
    TRIG --> V4{"Video?"}
    TRIG --> V5{"Local?"}
    TRIG --> V6{"Shopping?"}
    TRIG --> V7{"Direct answer?"}

    V1 -->|"always"| RW["Web retrieval"]
    V2 -->|"visual intent<br/>p > θ_img"| RI["Image retrieval"]
    V3 -->|"fresh intent<br/>+ recent news exists"| RN["News retrieval"]
    V4 -->|"video intent"| RV["Video retrieval"]
    V5 -->|"geo intent<br/>+ location available"| RL["Local retrieval"]
    V6 -->|"commercial intent"| RS["Product retrieval"]
    V7 -->|"factual question<br/>+ high-confidence answer"| RA["Answer extraction"]

    RW & RI & RN & RV & RL & RS & RA --> NORM["Score normalization<br/>⚠️ different verticals produce<br/>incomparable score scales"]

    NORM --> ALLOC["Slot allocation<br/>predicted utility per vertical<br/>× position value"]

    ALLOC --> LAYOUT["Layout assembly"]
    LAYOUT --> POST["Post-processing"]

    POST --> P1["Host diversity: max 2 results per domain"]
    POST --> P2["Near-duplicate removal across verticals"]
    POST --> P3["Safety filters: SafeSearch, policy"]
    POST --> P4["Freshness boost if news triggered"]

    P1 & P2 & P3 & P4 --> SERP(("Final SERP"))

    classDef always fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef cond fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef risk fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class V1,RW always
    class V2,V3,V4,V5,V6,V7 cond
    class NORM risk
```

**The hard part is score normalisation.** A web result's score and an image result's score come from different models trained on different objectives; they are not on a common scale, so you cannot simply sort them together. The standard solution is to convert each vertical's score into a **predicted utility** — an estimate of the probability the user is satisfied by that item — calibrated on held-out human judgements. Only calibrated probabilities are comparable across verticals.

**Triggering too eagerly is the classic failure.** Showing a news carousel for a query with no news intent pushes the actually-relevant web result below the fold. Vertical triggers are therefore tuned conservatively: the bar for *adding* a vertical is higher than the bar for a web result to keep its slot.

---

## 7.6 Snippet generation

The snippet is what the user actually reads. It is generated per query, not stored, because the useful passage depends on what was asked.

> **Diagram D-38 — Query-biased snippet generation**

```mermaid
flowchart TB
    IN["Top-10 docids + query terms"]

    IN --> FETCH["Fetch document text<br/>from forward index<br/>⚠️ 10 parallel reads —<br/>this is the doc-store hot path"]

    FETCH --> SPLIT["Split into candidate passages<br/>sentences or sliding windows"]

    SPLIT --> SCORE["Score each passage"]
    SCORE --> S1["Query term coverage<br/>how many distinct terms appear"]
    SCORE --> S2["Term proximity<br/>are they close together"]
    SCORE --> S3["Position in document<br/>earlier is usually better"]
    SCORE --> S4["Sentence completeness<br/>avoid mid-sentence cuts"]
    SCORE --> S5["Readability and information density"]

    S1 & S2 & S3 & S4 & S5 --> PICK["Select best 1–2 passages"]

    PICK --> TRIM["Trim to display width<br/>~160 chars, respect word and<br/>grapheme-cluster boundaries"]
    TRIM --> BOLD["Bold matched terms<br/>including morphological variants"]
    BOLD --> SAFE["Escape HTML,<br/>strip bidi control characters"]

    SAFE --> TITLE["Title selection<br/>title tag · h1 · anchor text ·<br/>whichever best matches intent"]

    TITLE --> META{"Structured data<br/>available?"}
    META -->|"Yes"| RICH["Rich result:<br/>ratings · price · dates · FAQ"]
    META -->|"No"| PLAIN["Plain snippet"]

    RICH & PLAIN --> OUT(("SERP item"))

    classDef hot fill:#fecaca,stroke:#b91c1c,color:#1c1917
    classDef proc fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    class FETCH hot
    class SPLIT,SCORE,PICK,TRIM,BOLD,SAFE proc
```

Two notes that matter more than they look:

- **Snippet generation is a document-store read amplifier.** Every uncached query performs ~10 random reads against a petabyte-scale store, in the latency-critical path. This is why the forward index is co-located with the leaf serving tier where possible, and why snippet caching is a separate cache layer in [Chapter 10](10-caching.md).
- **Bidi control characters must be stripped.** In a bilingual context this is not cosmetic: Unicode direction-override characters can make displayed text read differently from its actual content — a genuine SERP-spoofing vector. Sanitising them is a security control, not a formatting nicety.

---

## 7.7 What the leaf server actually does

For completeness, the innermost loop:

```
function leafSearch(query, k, deadline):
    candidates ← empty top-k heap
    θ ← −∞                                   # k-th best score so far

    postings ← [openPostingList(t) for t in query.terms]
    sort postings by document frequency ascending    # rarest term drives

    while not exhausted and now() < deadline:
        # BlockMax-WAND pruning (Chapter 06 §6.8)
        if Σ blockMaxScore(p) for p in postings ≤ θ:
            skipBlock(postings)
            continue

        d ← nextCommonDocid(postings)
        if d is null: break

        if isTombstoned(d) or failsFilters(d, query):
            continue

        score ← L1Score(d, query)            # cheap: BM25-ish + static signals
        if score > θ:
            candidates.push(d, score)
            if candidates.size > k:
                candidates.pop()
                θ ← candidates.min()

    return candidates, exhausted ? COMPLETE : PARTIAL
```

Three properties of this loop define the system's character:

1. **It is deadline-aware.** The loop checks the clock and returns what it has. A leaf never blows the budget; it returns a partial answer flagged as such.
2. **L1 scoring is deliberately cheap.** BM25-style term scoring plus precomputed static signals — no model inference, no document fetch. Expensive scoring happens later, on far fewer documents.
3. **Pruning is exact.** BlockMax-WAND skips work without changing the top-k. The cheap path and the correct path are the same path.

---

<div align="center">

**← [06 · Indexing](06-indexing.md)** · [🏠 Home](../../README.md) · **[08 · Ranking](08-ranking.md) →** · [🇸🇦 العربية](../ar/07-serving.md)

</div>
