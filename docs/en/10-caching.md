<div align="right">

**[🇸🇦 اقرأ بالعربية](../ar/10-caching.md)**

</div>

# Chapter 10 · Caching Strategy

**← [09 · Storage](09-storage.md)** · [🏠 Home](../../README.md) · **[11 · Freshness](11-freshness.md) →**

---

## 10.1 Why caching is an economic decision, not a latency one

The instinct is that caches make things fast. In this system they mostly make things **cheap**, and the distinction changes how you design them.

From [Chapter 03](03-capacity-estimation.md): peak traffic is 3 × 10⁵ QPS, and a 40 % result-cache hit rate means only 1.8 × 10⁵ QPS reach the index. Without that cache the serving fleet would need to be ~67 % larger. The cache is not shaving milliseconds off an already-fast path — it is deleting two-thirds of a fleet's worth of hardware.

This reframes the design question. It is not "how do we make cached queries faster?" but **"how do we maximise the fraction of traffic that never touches the index, without ever serving something wrong?"**

---

## 10.2 Query traffic is extremely skewed — and that is what makes caching work

> **Diagram D-51 — Query popularity distribution and cacheability**

```mermaid
flowchart TB
    ALL["All queries<br/>~10⁵ QPS average"]

    ALL --> HEAD["🔥 HEAD<br/>~10³ distinct queries<br/>≈ 15–20% of all traffic<br/>'facebook' · 'weather' · 'youtube'"]
    ALL --> TORSO["📊 TORSO<br/>~10⁷ distinct queries<br/>≈ 40–50% of traffic<br/>common but not universal"]
    ALL --> TAIL["🌊 TAIL<br/>~10⁹⁺ distinct queries<br/>≈ 30–40% of traffic<br/>15% of daily queries are<br/>never-before-seen strings"]

    HEAD --> HC["Cache: ✅ essentially always hit<br/>tiny memory footprint<br/>hit rate ~99%"]
    TORSO --> TC["Cache: ✅ good hit rate<br/>most of the cache's value lives here<br/>hit rate ~50%"]
    TAIL --> TLC["Cache: ❌ near-zero hit rate<br/>caching it wastes memory<br/>hit rate ~2%"]

    HC & TC --> SAVE["Combined ~40% hit rate<br/>= ~40% of serving cost removed"]
    TLC --> FULL["Tail queries always pay<br/>the full retrieval + ranking cost<br/>→ this is what the fleet is sized for"]

    subgraph INSIGHT["The design consequence"]
        I1["Cache capacity should target the TORSO.<br/>The head fits in almost no memory;<br/>the tail is uncacheable by nature."]
        I2["Admission policy matters more than eviction policy:<br/>do not admit a query on first sight —<br/>most tail queries are seen exactly once."]
    end

    classDef head fill:#fecaca,stroke:#b91c1c,color:#1c1917
    classDef torso fill:#fed7aa,stroke:#c2410c,color:#1c1917
    classDef tail fill:#dbeafe,stroke:#1e40af,color:#1c1917
    classDef good fill:#bbf7d0,stroke:#15803d,color:#1c1917
    class HEAD,HC head
    class TORSO,TC torso
    class TAIL,TLC tail
    class SAVE good
```

**The second insight box is the practically important one.** A naïve LRU cache admits every query on first request. Since ~15 % of daily queries have never been seen before and will likely never be seen again, a naïve cache spends a large share of its memory on entries that will never be read. Requiring a query to be seen twice within a window before admitting it (a "TinyLFU"-style admission filter) can raise effective hit rate substantially at zero extra memory.

---

## 10.3 The cache hierarchy

There is no single cache. There are six, at different layers, with different keys, sizes and TTLs.

> **Diagram D-52 — Multi-layer cache hierarchy**

```mermaid
flowchart TB
    U(("👤 User"))

    U --> L1["① Browser cache<br/>key: full URL<br/>TTL: seconds–minutes<br/>saves: entire round trip"]
    L1 -->|"miss"| L2["② Edge / CDN cache<br/>static assets, JS, CSS, logos<br/>NOT personalized results<br/>saves: origin round trip"]

    L2 -->|"miss"| L3["③ Result cache (SERP cache)<br/>key: normalized_query + locale +<br/>country + safesearch + device_class<br/>value: full ranked result set<br/>TTL: 5 min – 24 h by intent<br/>hit rate ~40% · THE BIG ONE"]

    L3 -->|"miss"| L4["④ Query-understanding cache<br/>key: raw query string<br/>value: spelling, intent, expansion, embedding<br/>TTL: hours<br/>hit rate ~60% — cheap and very effective"]

    L4 --> L5["⑤ Posting-list cache<br/>key: term id<br/>value: decompressed hot posting blocks<br/>on the leaf server, LFU<br/>hit rate ~85% for common terms"]

    L5 --> L6["⑥ Document / snippet cache<br/>key: docid + query-term set<br/>value: rendered snippet, title, metadata<br/>protects the doc store from<br/>the read amplification in Ch 07"]

    L6 --> IDX[("Index + document store")]

    subgraph TTL["TTL by query intent"]
        T1["Navigational 'facebook' → 24 h<br/>answer essentially never changes"]
        T2["Informational → 1–6 h"]
        T3["Local → 1 h<br/>hours and availability change"]
        T4["News / fresh → 1–5 min"]
        T5["Trending / breaking → 30 s or bypass"]
        T6["Personalized → not cached globally"]
    end

    L3 -.governed by.-> TTL

    classDef client fill:#e5e7eb,stroke:#4b5563,color:#1c1917
    classDef big fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef server fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    class L1,L2 client
    class L3,L4 big
    class L5,L6 server
```

### The cache key is the hardest part

Every dimension added to the key multiplies the key space and divides the hit rate:

| Key | Approximate distinct keys | Hit rate impact |
|---|---|---|
| `query` | 10⁹ | Baseline |
| `query + country` | 10⁹ × 200 | Necessary — results genuinely differ |
| `+ language` | × 50 | Necessary |
| `+ safesearch` | × 2 | Necessary — correctness |
| `+ device_class` | × 3 | Justifiable — layout differs |
| `+ city` | × 10⁴ | ❌ Destroys the cache |
| `+ user_id` | × 10⁹ | ❌ Cache becomes pointless |

This is the concrete mechanism behind the warning in [Chapter 08](08-ranking.md): **personalisation is expensive because it destroys cacheability**, not because personalising is computationally hard. The standard resolution is a **two-stage design** — cache the globally-computed result set under a coarse key, then apply light per-user reordering after the cache read. The expensive 200 ms of retrieval and ranking is shared; only the cheap final adjustment is per-user.

---

## 10.4 Invalidation

> **Diagram D-53 — Cache coherence and invalidation paths**

```mermaid
flowchart TB
    subgraph TRIGGERS["What invalidates a cached SERP"]
        E1["New index version published"]
        E2["Document removed — legal or policy"]
        E3["Breaking news detected for a topic"]
        E4["Ranking model rolled out"]
        E5["Natural TTL expiry"]
        E6["Spam demotion applied"]
    end

    E1 --> S1{"Strategy"}
    E2 --> S2{"Strategy"}
    E3 --> S3{"Strategy"}
    E4 --> S1
    E5 --> S4{"Strategy"}
    E6 --> S2

    S1 --> V["🏷️ Version-tagged keys<br/>cache key includes index_version<br/>publishing a new version makes<br/>every old entry unreachable —<br/>no explicit purge needed"]

    S2 --> P["📢 Active purge broadcast<br/>docid → cache tier<br/>must be FAST and RELIABLE<br/>this is the legal-compliance path"]

    S3 --> T["⏱️ TTL collapse<br/>drop TTL for the affected topic<br/>from hours to seconds"]

    S4 --> L["🔄 Lazy expiry + refresh-ahead<br/>serve stale while<br/>asynchronously recomputing"]

    V & P & T & L --> RISK{"Risk check"}

    RISK --> R1["⚠️ Thundering herd:<br/>if all entries expire together,<br/>the index tier is hit by a<br/>synchronized wave"]
    R1 --> F1["✅ Fix: randomized TTL jitter<br/>+ request coalescing<br/>(one recompute per key, others wait)"]

    RISK --> R2["⚠️ Cold cache after deploy:<br/>a restarted cache tier has<br/>0% hit rate and can<br/>overload the index tier"]
    R2 --> F2["✅ Fix: staged restarts,<br/>cache warming from a persistent<br/>snapshot, admission rate limits"]

    RISK --> R3["⚠️ Serving removed content:<br/>a purge that fails silently is a<br/>compliance failure, not a perf bug"]
    R3 --> F3["✅ Fix: purge acknowledgements,<br/>a deny-list checked at read time<br/>as a backstop, audit logging"]

    classDef strat fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef risk fill:#fecaca,stroke:#b91c1c,color:#1c1917
    classDef fix fill:#bbf7d0,stroke:#15803d,color:#1c1917
    class V,P,T,L strat
    class R1,R2,R3 risk
    class F1,F2,F3 fix
```

**Version-tagged keys are the elegant solution and should be the default.** Because index segments are immutable ([Chapter 06](06-indexing.md)), the live index has a version number. Including it in the cache key means a new index publish atomically invalidates the entire cache without sending a single purge message — old entries simply become unreachable and age out naturally. Immutability pays off a third time.

**But version tagging is not sufficient for legal removals.** Those cannot wait for the next index publish. They need the active purge path *and* a read-time deny-list backstop, because a purge message that is dropped silently produces a compliance violation rather than a stale result. This is a case where defence in depth is genuinely warranted: the fast path is the purge, and the correctness guarantee is the read-time check.

---

## 10.5 What must never be cached

| Data | Why not |
|---|---|
| Personalised result ordering | Serving one user's results to another is a privacy incident |
| Authenticated / private content | Never enters the public index in the first place |
| Legally removed URLs | Must disappear immediately and verifiably |
| Real-time data (live scores, stock prices) | Staleness is the failure, not an optimisation |
| Anything keyed on partial user identity | Cache poisoning becomes cross-user data leakage |

**Cache key collisions are a security vulnerability, not a bug.** If two different logical requests can map to the same cache key, one user can receive another's results. This is why the cache key must include *every* dimension that affects the response — including SafeSearch state and country, which are correctness dimensions rather than performance ones. The safe design principle: **when in doubt, add the dimension to the key and accept the lower hit rate.**

---

## 10.6 Measuring cache effectiveness

> **Diagram D-54 — Metrics that actually matter**

```mermaid
flowchart LR
    subgraph GOOD["✅ Metrics worth optimizing"]
        G1["Hit rate by layer<br/>and by query segment"]
        G2["Cost avoided<br/>= hits × cost per index query<br/>THE metric — it is money"]
        G3["Staleness distribution<br/>p50 / p99 age of served entries"]
        G4["Origin QPS reduction"]
        G5["Correctness incidents<br/>must be zero"]
    end

    subgraph BAD["⚠️ Metrics that mislead"]
        B1["Raw hit rate alone<br/>→ gameable by caching<br/>cheap queries and<br/>lengthening TTLs"]
        B2["Cache latency<br/>→ already ~1 ms;<br/>optimizing it changes nothing"]
        B3["Memory utilization<br/>→ a full cache is not<br/>necessarily an effective one"]
    end

    G2 --> DECIDE{"Decision"}
    DECIDE --> D1["Add memory if<br/>marginal cost avoided ><br/>marginal RAM cost"]
    DECIDE --> D2["Lengthen TTL only if<br/>staleness stays within<br/>the intent's tolerance"]
    DECIDE --> D3["Improve admission policy<br/>— usually the cheapest win"]

    classDef good fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef bad fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class G1,G2,G3,G4,G5 good
    class B1,B2,B3 bad
```

**Hit rate alone is a trap.** You can raise it trivially by extending TTLs, which trades a metric improvement for silently staler results. The metric that cannot be gamed is **cost avoided per unit of staleness introduced** — and it must always be evaluated alongside the correctness-incident count, which has a hard target of zero.

---

<div align="center">

**← [09 · Storage](09-storage.md)** · [🏠 Home](../../README.md) · **[11 · Freshness](11-freshness.md) →** · [🇸🇦 العربية](../ar/10-caching.md)

</div>
