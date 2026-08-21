<div align="right">

**[🇸🇦 اقرأ بالعربية](../ar/08-ranking.md)**

</div>

# Chapter 08 · Ranking & Relevance

**← [07 · Serving](07-serving.md)** · [🏠 Home](../../README.md) · **[09 · Storage](09-storage.md) →**

---

## 8.1 Ranking is the product

Everything in Chapters 04–07 exists to deliver a few hundred plausible candidates to this chapter. Retrieval is an engineering problem with correct answers. **Ranking is a judgement problem with no ground truth** — which documents are "best" for `jaguar` depends on who is asking, when, and why.

The architectural consequence is the **cascade**: apply cheap scoring to many documents and expensive scoring to few, so that total cost stays inside the latency budget while the final ordering gets the benefit of the most powerful model you can afford.

> **Diagram D-39 — The retrieval and ranking funnel**

```mermaid
flowchart TB
    C0["🌐 Corpus<br/>10¹¹ documents"]

    C0 -->|"inverted index lookup<br/>cost: ~0"| C1["📑 Term-matched set<br/>10⁶ – 10⁹ documents<br/>never materialized"]

    C1 -->|"BlockMax-WAND pruning<br/>exact top-k, no quality loss"| C2["⚡ L0 · Retrieved<br/>~10⁵ documents<br/>touched per shard"]

    C2 -->|"L1: BM25 + static signals<br/>~10 ns per doc<br/>runs ON the leaf server"| C3["🔢 L1 · Cheap-scored<br/>~10³ per shard<br/>→ 10³ globally after merge"]

    C3 -->|"L2: gradient-boosted trees<br/>hundreds of features<br/>~10 µs per doc"| C4["🌲 L2 · Feature-scored<br/>~10² documents"]

    C4 -->|"L3: cross-encoder transformer<br/>full query-document attention<br/>~1 ms per doc, accelerated"| C5["🧠 L3 · Neural re-ranked<br/>~10 documents"]

    C5 -->|"diversity · freshness ·<br/>host dedup · policy filters"| C6["✅ Final SERP<br/>10 results"]

    subgraph BUDGET["Cost discipline — the whole point of the cascade"]
        B1["L1: 10⁵ docs × 10 ns  =   1 ms"]
        B2["L2: 10³ docs × 10 µs  =  10 ms"]
        B3["L3: 10² docs × 1 ms   = 100 ms"]
        B4["Invert any stage's ratio<br/>and the budget explodes"]
    end

    classDef wide fill:#dbeafe,stroke:#1e40af,color:#1c1917
    classDef mid fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef narrow fill:#fecaca,stroke:#b91c1c,color:#1c1917
    classDef final fill:#bbf7d0,stroke:#15803d,color:#1c1917
    class C0,C1,C2 wide
    class C3,C4 mid
    class C5 narrow
    class C6 final
```

**Read the budget box carefully — it is the core insight.** L3 is 100,000× more expensive per document than L1. Running L3 on 10⁵ documents would take 100 seconds. The funnel is not an optimisation layered on top of ranking; the funnel *is* how expensive ranking becomes possible at all.

The corresponding risk: **a document wrongly dropped at L1 can never be recovered by L3.** Recall errors early in the cascade are permanent. This is why L1 is deliberately generous — it optimises recall, not precision, and leaves precision to the stages that can afford it.

---

## 8.2 The signal taxonomy

> **Diagram D-40 — Where ranking signals come from**

```mermaid
mindmap
  root((Ranking<br/>Signals))
    Query dependent
      Computed at query time
      Term frequency in doc
      Inverse document frequency
      BM25 and field weighted variants
      Term proximity
      Phrase and order match
      Field matches
        Title match
        URL match
        Anchor text match
        Heading match
      Semantic similarity
        Dense vector cosine
        Cross encoder relevance
      Query coverage
        Fraction of terms matched
        Essential term present
    Query independent
      Precomputed at index time
      Link based
        PageRank
        Host authority
        Trust distance from seeds
      Content quality
        Originality
        Depth and completeness
        Spam classifier score
      User signals aggregated
        Historical click through rate
        Long click rate
        Bounce rate
      Page experience
        Load performance
        Mobile friendliness
        Layout stability
        HTTPS
      Freshness
        Publication date
        Last substantive update
        Change rate
    Context dependent
      Resolved per request
      Location
        Country and region
        City for local intent
      Language and locale
      Device and screen
      Time
        Time of day
        Seasonality
        Breaking events
      Consented personalization
        Search history
        Explicit preferences
```

The split matters operationally:

| Class | Computed | Storage cost | Latency cost |
|---|---|---|---|
| Query-independent | Offline, at index time | High — stored per document | ~0 at query time |
| Query-dependent | At query time, in the leaf | ~0 | The dominant serving cost |
| Context-dependent | At query time, from request | ~0 | Low, but multiplies cache keys |

**Push everything you can into the query-independent column.** That is the recurring principle from [Chapter 01](01-overview.md) — do the expensive work offline. The query path should *combine* precomputed numbers, not compute them.

---

## 8.3 PageRank — the query-independent authority signal

PageRank models a random surfer who follows links at random and occasionally jumps to a random page. The stationary probability of being at page *p* is its PageRank.

$$PR(p) = \frac{1-d}{N} + d \sum_{q \in In(p)} \frac{PR(q)}{L(q)}$$

where *d* ≈ 0.85 is the damping factor, *N* is the number of pages, *In(p)* is the set of pages linking to *p*, and *L(q)* is the outdegree of *q*.

> **Diagram D-41 — PageRank computation at scale**

```mermaid
flowchart TB
    G[("Link graph<br/>10¹¹ nodes · 10¹² edges<br/>far too large for one machine")]

    G --> INIT["Initialize<br/>PR(p) = 1/N for all p"]

    INIT --> ITER["Iteration k"]

    subgraph MR["One iteration = one MapReduce"]
        MAP["MAP<br/>for each page q with rank PR(q):<br/>emit ⟨p, PR(q)/L(q)⟩ for each outlink p<br/>also emit ⟨q, adjacency list⟩"]
        SHUF["SHUFFLE<br/>group contributions by target page"]
        RED["REDUCE<br/>PR(p) = (1−d)/N + d × Σ contributions<br/>re-emit adjacency list for next round"]
        MAP --> SHUF --> RED
    end

    ITER --> MAP
    RED --> CONV{"Converged?<br/>Σ|PR_k − PR_k−1| < ε<br/>typically 20–50 iterations"}

    CONV -->|"No"| ITER
    CONV -->|"Yes"| POST["Post-processing"]

    POST --> P1["Handle dangling nodes<br/>pages with no outlinks<br/>redistribute their mass uniformly"]
    POST --> P2["Log-scale and bucket<br/>into a small integer<br/>— only relative order matters"]
    POST --> P3["Compute host-level<br/>and domain-level aggregates"]

    P1 & P2 & P3 --> OUT[("Per-document authority score<br/>→ stored in doc attachments<br/>→ shipped with the index")]

    subgraph VARIANTS["Practical variants"]
        V1["Topic-sensitive PageRank:<br/>separate vectors per topic,<br/>combined by query topic"]
        V2["TrustRank:<br/>seed from hand-vetted good sites,<br/>propagate trust to fight spam"]
        V3["Damped by link quality:<br/>nofollow, paid links and<br/>link-farm edges discounted"]
    end

    OUT --> VARIANTS

    classDef compute fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef store fill:#e9d5ff,stroke:#7e22ce,color:#1c1917
    class MAP,SHUF,RED,INIT compute
    class G,OUT store
```

**Two honest caveats.**

First, **PageRank is one signal among hundreds**, and its relative weight has fallen enormously since 1998. Treating it as "the Google algorithm" is a common and significant error. It provides a query-independent prior on authority; it says nothing about whether a page answers the question.

Second, **dangling nodes are not a footnote.** A large fraction of the web has no outlinks (PDFs, images, leaf pages). Without redistributing their rank mass, probability leaks out of the system every iteration and the computation does not converge to a proper distribution. Handling them correctly is a real implementation requirement, not a theoretical nicety.

---

## 8.4 Learning to rank

Hand-tuning hundreds of feature weights is impossible. Ranking is learned from labelled data.

> **Diagram D-42 — Learning-to-rank training and deployment loop**

```mermaid
flowchart TB
    subgraph LABELS["① Label acquisition"]
        HR["Human raters<br/>guideline-driven judgements<br/>graded 0–4 relevance<br/>expensive, unbiased"]
        CL["Click logs<br/>cheap, abundant<br/>heavily position-biased"]
        DEBIAS["Bias correction<br/>inverse propensity weighting<br/>position-bias model<br/>result-randomization experiments"]
        CL --> DEBIAS
    end

    subgraph FEATS["② Feature extraction"]
        FX["Log query-document feature vectors<br/>exactly as computed in production<br/>⚠️ training/serving skew is<br/>the #1 source of silent failure"]
    end

    HR & DEBIAS --> FX

    subgraph TRAIN["③ Model training"]
        OBJ{"Objective formulation"}
        OBJ -->|"Pointwise"| PW["Predict relevance per doc<br/>simple, ignores ranking structure"]
        OBJ -->|"Pairwise"| PR2["Predict which of two docs<br/>ranks higher — RankNet"]
        OBJ -->|"Listwise"| LW["Optimize the whole list<br/>directly against NDCG —<br/>LambdaMART, LambdaRank"]
        LW --> MODEL["Trained model<br/>GBDT for L2 · transformer for L3"]
        PW --> MODEL
        PR2 --> MODEL
    end

    FX --> OBJ

    subgraph EVAL["④ Offline evaluation"]
        E1["NDCG@10 on held-out<br/>human-rated queries"]
        E2["Slice analysis:<br/>by language · locale ·<br/>query length · intent · head vs tail"]
        E3["Counterfactual estimation<br/>from logged data"]
    end

    MODEL --> E1 & E2 & E3

    E1 & E2 & E3 --> GATE{"Beats production<br/>on every slice?"}
    GATE -->|"No"| BACK["Iterate — investigate<br/>the regressed slice"]
    BACK --> OBJ

    GATE -->|"Yes"| AB["⑤ Online A/B test<br/>small traffic slice<br/>see Chapter 13"]

    AB --> METRICS{"Online metrics"}
    METRICS --> M1["Human side-by-side rating ⬆"]
    METRICS --> M2["Long-click rate ⬆"]
    METRICS --> M3["Query reformulation ⬇"]
    METRICS --> M4["Latency within SLO"]

    M1 & M2 & M3 & M4 --> SHIP{"All green?"}
    SHIP -->|"Yes"| RAMP["Ramp 1% → 10% → 50% → 100%"]
    SHIP -->|"No"| ROLL["🛑 Roll back, analyze"]

    RAMP --> PROD(("Production ranking"))
    PROD -.->|"generates new logs"| CL

    classDef human fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef risky fill:#fecaca,stroke:#b91c1c,color:#1c1917
    classDef ml fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    class HR,M1 human
    class CL,FX risky
    class PW,PR2,LW,MODEL ml
```

### Why listwise objectives win

Ranking quality is measured by NDCG, which is a property of the *whole list*, not of individual documents. A pointwise model that predicts each document's relevance perfectly in isolation can still produce a poor list, because it has no notion that swapping positions 1 and 2 matters far more than swapping positions 9 and 10. LambdaMART works by directly weighting each training pair by *how much NDCG would change* if those two documents swapped — which aligns the gradient with the metric you actually care about.

### The two failure modes that actually bite

**Position bias.** Users click result #1 far more than result #5, regardless of relevance. A model trained naïvely on clicks learns "documents that were ranked first are good", which is circular: it entrenches the current ranking and can never discover that #7 was better. The fixes — inverse propensity weighting, and deliberately randomising results for a small traffic slice to collect unbiased data — are mandatory, not optional refinements.

**Training/serving skew.** If a feature is computed one way in the training pipeline and slightly differently in the serving binary, the model receives inputs it was never trained on. This produces no error and no alert; quality simply degrades. The only reliable defence is to **log features from production serving and train on exactly those logged vectors**, never recomputing them in the training pipeline.

---

## 8.5 Neural retrieval — beyond term matching

Lexical retrieval fails on the **vocabulary mismatch problem**: a document about "automobile maintenance" does not match a query for "car repair" even though it is exactly what the user wants. Dense retrieval solves this by mapping both queries and documents into a shared semantic vector space.

> **Diagram D-43 — Dense retrieval and hybrid fusion**

```mermaid
flowchart TB
    subgraph OFFLINE["Offline — at index time"]
        D["Document text"] --> DE["Document encoder<br/>transformer bi-encoder"]
        DE --> DV["Dense vector<br/>768-dim, quantized to int8"]
        DV --> ANN["ANN index construction"]
        ANN --> A1["HNSW: navigable small-world graph<br/>excellent recall, high RAM"]
        ANN --> A2["IVF-PQ: coarse cluster +<br/>product quantization<br/>lower RAM, tunable recall"]
    end

    subgraph ONLINE["Online — at query time"]
        Q["Query text"] --> QE["Query encoder<br/>same embedding space"]
        QE --> QV["Query vector"]
    end

    QV --> SEARCH["ANN search<br/>approximate nearest neighbours<br/>~1–5 ms for top-1000"]
    A1 & A2 --> SEARCH

    subgraph HYBRID["Hybrid retrieval — both paths run"]
        LEX["🔤 Lexical candidates<br/>BM25 / inverted index<br/>✅ exact terms, names, numbers, codes<br/>✅ interpretable, cheap<br/>❌ vocabulary mismatch"]
        SEM["🧠 Semantic candidates<br/>dense vector search<br/>✅ synonyms, paraphrase, cross-lingual<br/>❌ weak on rare exact strings<br/>❌ opaque, expensive to update"]
    end

    SEARCH --> SEM
    LEXIN["Inverted index<br/>Chapter 06"] --> LEX

    LEX & SEM --> FUSE["Fusion"]
    FUSE --> F1["Reciprocal Rank Fusion:<br/>score = Σ 1/(k + rank_i)<br/>no score calibration needed"]
    FUSE --> F2["Learned fusion:<br/>both scores as L2 features"]

    F1 & F2 --> CAND["Merged candidate set"]
    CAND --> XE["Cross-encoder re-ranking<br/>query and document read TOGETHER<br/>full attention between them<br/>far more accurate, far more expensive"]
    XE --> FINAL(("Final ranking"))

    classDef lex fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef sem fill:#e9d5ff,stroke:#7e22ce,color:#1c1917
    classDef fuse fill:#bbf7d0,stroke:#15803d,color:#1c1917
    class LEX,LEXIN lex
    class SEM,DE,QE,DV,QV,SEARCH sem
    class FUSE,F1,F2,CAND fuse
```

### Bi-encoder vs cross-encoder — why both exist

| | **Bi-encoder** (retrieval) | **Cross-encoder** (re-ranking) |
|---|---|---|
| Structure | Encode query and document *separately* | Encode query and document *together* |
| Document vectors | Precomputed offline, stored in the index | Must be computed per query-document pair |
| Cost at query time | One query encoding + ANN search | One full transformer pass **per document** |
| Accuracy | Good | Substantially better |
| Feasible over | 10¹¹ documents | ~10² documents |

This is exactly why the cascade exists. The bi-encoder can pre-compute document vectors because it never looks at the query while encoding — so retrieval over billions is possible. The cross-encoder is more accurate precisely *because* it reads both together, which is also why it can never be precomputed. Cheap-and-parallel retrieves; expensive-and-accurate re-ranks.

### When lexical retrieval still wins

Dense retrieval is not a replacement. It reliably loses on:

- **Exact identifiers** — part numbers, error codes, `NullPointerException`, phone numbers
- **Rare proper nouns** absent or under-represented in training data
- **Negation and precise logic** — embeddings notoriously blur "X" and "not X"
- **Freshness** — a brand-new entity is not in the embedding model's learned vocabulary until it is retrained; the inverted index has it the moment it is crawled

That last point is architecturally important: **the lexical index updates in minutes; the embedding model updates in weeks.** Hybrid retrieval is not hedging — it is combining two systems with genuinely complementary failure modes.

---

## 8.6 Personalisation and context

> **Diagram D-44 — Context application in ranking**

```mermaid
flowchart TB
    BASE["Base ranking<br/>from L2 + L3"]

    BASE --> CTX{"Apply context"}

    CTX --> LOC["📍 Location"]
    LOC --> L1["Country → corpus and locale bias"]
    LOC --> L2["City → local intent results"]
    LOC --> L3["⚠️ Legal: region-scoped removals"]

    CTX --> LANG["🗣️ Language"]
    LANG --> LG1["Interface language"]
    LANG --> LG2["Content language preference"]
    LANG --> LG3["Cross-lingual retrieval when<br/>the corpus is thin in-language"]

    CTX --> TIME["🕐 Time"]
    TIME --> T1["Freshness boost for news intent"]
    TIME --> T2["Seasonality and recurring events"]
    TIME --> T3["Breaking-event detection →<br/>temporarily raise recency weight"]

    CTX --> DEV["📱 Device"]
    DEV --> D1["Mobile-friendliness weighting"]
    DEV --> D2["Result count and layout"]

    CTX --> HIST["👤 Consented history"]
    HIST --> H1["Session context:<br/>previous query disambiguates<br/>'jaguar' after 'zoo tickets'"]
    HIST --> H2["Long-term interests<br/>— small weight, opt-out honoured"]

    L1 & L2 & L3 & LG1 & LG2 & LG3 & T1 & T2 & T3 & D1 & D2 & H1 & H2 --> ADJ["Adjusted ranking"]

    ADJ --> GUARD["Guardrails"]
    GUARD --> G1["Cap personalization influence<br/>— it is a tiebreaker, not a re-ranker"]
    GUARD --> G2["Preserve diversity<br/>avoid filter bubbles"]
    GUARD --> G3["Never personalize away<br/>authoritative results on<br/>health, safety, civic topics"]
    GUARD --> G4["Full opt-out must be honoured<br/>and must actually work"]

    G1 & G2 & G3 & G4 --> FINAL(("Personalized SERP"))

    classDef ctx fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef guard fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class LOC,LANG,TIME,DEV,HIST ctx
    class GUARD,G1,G2,G3,G4 guard
```

**Personalisation is architecturally expensive in a way that is easy to miss.** Every context dimension you add multiplies the result-cache key space. A cache keyed on `(query, country, language)` might hit 40 % of the time; a cache keyed on `(query, country, language, city, device, user_id)` will essentially never hit. Since [Chapter 03](03-capacity-estimation.md) showed the cache removes ~40 % of all serving cost, aggressive personalisation can *double the cost of the entire serving fleet*. This — far more than any privacy argument — is why production personalisation is applied as a light post-ranking adjustment rather than as a per-user retrieval.

---

## 8.7 Diversity, and the final adjustments

The top-10 by score is often a bad SERP. Ten near-identical pages from one site technically maximise relevance while serving the user poorly.

> **Diagram D-45 — Post-ranking result-set optimisation**

```mermaid
flowchart LR
    IN["Top ~50 by score"] --> DIV["Diversification"]

    DIV --> D1["Host clustering<br/>max 2 results per domain,<br/>collapse the rest under<br/>'more results from…'"]
    D1 --> D2["Intent coverage<br/>ambiguous query 'jaguar' →<br/>cover animal, car, team"]
    D2 --> D3["MMR-style selection<br/>score −λ × max similarity<br/>to already-selected results"]
    D3 --> D4["Content-type mix<br/>avoid ten forum threads"]

    D4 --> FRESH["Freshness adjustment<br/>boost recency if news intent<br/>or query deserves freshness"]

    FRESH --> FILT["Final filters"]
    FILT --> F1["SafeSearch"]
    FILT --> F2["Legal removals — region scoped"]
    FILT --> F3["Spam threshold"]
    FILT --> F4["Broken-link / soft-404 suppression"]

    F1 & F2 & F3 & F4 --> WHY["Explainability annotations<br/>why this result ranks here<br/>— for debugging and raters"]

    WHY --> OUT(("Final 10"))

    classDef div fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef filt fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class D1,D2,D3,D4 div
    class F1,F2,F3,F4 filt
```

**Diversification is an explicit trade of measured relevance for actual usefulness.** Every diversity constraint *lowers* the NDCG of the list as computed against per-document relevance labels, because you are deliberately not showing the highest-scoring document in some slot. It is justified only by user-level metrics — task success, reformulation rate, long clicks — which is a good illustration of why [Chapter 02](02-requirements.md) insists on multiple, disagreeing quality metrics rather than one optimisation target.

---

## 8.8 Summary — the ranking stack

| Stage | Runs on | Docs in → out | Model | Latency |
|---|---|---:|---|---:|
| **L0 Retrieval** | Leaf | 10¹¹ → 10⁵ | Inverted index + WAND | ~40 ms |
| **L1 Cheap scoring** | Leaf | 10⁵ → 10³ | BM25 + static signals | in the 40 ms |
| **L2 Feature scoring** | Ranking tier | 10³ → 10² | Gradient-boosted trees | ~30 ms |
| **L3 Neural re-rank** | Accelerators | 10² → 10¹ | Cross-encoder transformer | ~50 ms |
| **Post-processing** | Mixer | 10¹ → 10¹ | Rules + diversity | ~5 ms |

---

<div align="center">

**← [07 · Serving](07-serving.md)** · [🏠 Home](../../README.md) · **[09 · Storage](09-storage.md) →** · [🇸🇦 العربية](../ar/08-ranking.md)

</div>
