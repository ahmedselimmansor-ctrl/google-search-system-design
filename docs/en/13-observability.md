<div align="right">

**[🇸🇦 اقرأ بالعربية](../ar/13-observability.md)**

</div>

# Chapter 13 · Observability & Experimentation

**← [12 · Reliability](12-reliability.md)** · [🏠 Home](../../README.md) · **[14 · Spam & Security](14-security-abuse.md) →**

---

## 13.1 You cannot debug what you cannot see

A query touches thousands of machines across a dozen services. When it is slow, "which one was it?" is not answerable by logging into a box. Observability is not an operational afterthought here — it is a functional requirement of the architecture, because the architecture made single-machine debugging impossible.

> **Diagram D-64 — The observability stack**

```mermaid
flowchart TB
    subgraph SOURCES["Instrumented services"]
        SV1["Frontend"]
        SV2["Query understanding"]
        SV3["Root / leaf"]
        SV4["Ranking"]
        SV5["Crawler"]
        SV6["Index builder"]
    end

    SOURCES --> M["📊 METRICS<br/>numeric time series<br/>cheap · aggregated · always on<br/>counters · gauges · histograms<br/><br/>answers: IS something wrong?"]
    SOURCES --> L["📝 LOGS<br/>structured events<br/>expensive at full volume<br/>→ sampled + tail-based retention<br/><br/>answers: WHAT exactly happened?"]
    SOURCES --> T["🔗 TRACES<br/>causally linked spans<br/>sampled ~0.01–1%<br/>propagated trace id<br/><br/>answers: WHERE is the time going?"]

    M --> TS[("Time-series database<br/>downsampled by age")]
    L --> LS[("Log store<br/>indexed, short retention")]
    T --> TR[("Trace store<br/>sampled, searchable")]

    TS --> D1["Dashboards"]
    TS --> D2["SLO burn-rate alerts"]
    TS --> D3["Anomaly detection"]
    LS --> D4["Incident forensics"]
    TR --> D5["Latency attribution"]
    TR --> D6["Dependency mapping"]

    subgraph QUALITY["Search-specific signals — the ones generic APM misses"]
        Q1["Result quality: NDCG on golden queries,<br/>continuously evaluated in production"]
        Q2["Index coverage: % of shards<br/>that answered each query"]
        Q3["Index freshness: age distribution<br/>of served documents"]
        Q4["Crawl health: fetch success rate,<br/>politeness violations (must be 0)"]
        Q5["Cache effectiveness by segment"]
        Q6["Behavioural: CTR, long-click rate,<br/>reformulation rate, abandonment"]
    end

    SOURCES --> QUALITY
    QUALITY --> D1 & D2

    classDef metric fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef log fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef trace fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef qual fill:#e9d5ff,stroke:#7e22ce,color:#1c1917
    class M,TS metric
    class L,LS log
    class T,TR trace
    class Q1,Q2,Q3,Q4,Q5,Q6 qual
```

**The QUALITY box is what distinguishes search observability from generic monitoring.** A search engine can be perfectly healthy by every infrastructure metric — 100 % availability, p99 within SLO, zero errors — while returning terrible results. Latency and error rate cannot detect a bad ranking model or a corrupted index. Continuous quality evaluation against a fixed golden-query set is the only monitor that catches that class of failure, and it deserves the same alerting severity as an availability breach.

---

## 13.2 Distributed tracing

> **Diagram D-65 — A single query's trace**

```mermaid
gantt
    title Distributed trace: one slow query (units: milliseconds)
    dateFormat X
    axisFormat %s

    section frontend
    HandleSearch (root span)          :active, fe, 0, 340
    parseRequest                      :fe1, 2, 6

    section query-understanding
    understand                        :qu, 6, 34
    spellCheck                        :qu1, 8, 18
    intentClassify                    :qu2, 18, 28
    embedQuery                        :qu3, 28, 34

    section cache
    resultCacheLookup (MISS)          :ca, 34, 37

    section retrieval
    rootFanout                        :active, rt, 37, 195
    intermediate-07                   :im, 39, 193
    leaf-0712 (normal)                :lf1, 41, 96
    leaf-0713 (normal)                :lf2, 41, 92
    leaf-0714 SLOW - GC pause         :crit, lf3, 41, 191
    leaf-0714-hedge (replica)         :done, lf4, 130, 188

    section ranking
    L2 rescore                        :l2, 195, 228
    L3 neural rerank                  :l3, 228, 288

    section assembly
    snippetGeneration                 :sn, 288, 322
    blendAndSerialize                 :bl, 322, 338
```

Reading this trace, the diagnosis is immediate and unambiguous: **`leaf-0714` took 150 ms instead of ~55 ms**, and because the root must wait for all shards, that single leaf set the latency of the entire query. The hedged request fired at ~130 ms but its replica did not return early enough to help much.

Without tracing, this appears in metrics only as "p99 latency is up" — with a thousand candidate leaves and no way to attribute it. This is exactly the problem Dapper (2010) was built to solve, and its three design decisions are worth stating because they are what make tracing affordable at this scale:

| Decision | Why |
|---|---|
| **Sample aggressively** (0.01–1 %) | Tracing every request would cost more than serving it. Rare events still appear at high volume. |
| **Propagate a trace id through every RPC** | Causality cannot be reconstructed after the fact; it must be carried. |
| **Instrument the RPC layer, not the application** | Developers get tracing for free and cannot forget to add it. This is why coverage is complete. |

**Adaptive sampling** improves on uniform sampling significantly: always trace slow requests, errors, and a small uniform baseline. That way the traces you actually keep are disproportionately the interesting ones.

---

## 13.3 Experimentation

Every ranking change, UI change and infrastructure change is validated by controlled experiment. At any moment a large search engine is running *hundreds* of concurrent experiments — which creates a problem: there is not enough traffic to give each one an exclusive slice.

> **Diagram D-66 — Overlapping layered experiments**

```mermaid
flowchart TB
    REQ["Incoming request"] --> ID["Compute a stable diversion id<br/>hash(cookie or session)<br/>— MUST be stable so a user<br/>sees a consistent experience"]

    ID --> LAYERS["Assign independently in each layer"]

    subgraph L1["Layer 1 — Retrieval"]
        A1["Control: current retrieval"]
        A2["Exp 1a: wider candidate set"]
        A3["Exp 1b: hybrid dense weight +10%"]
    end

    subgraph L2["Layer 2 — Ranking"]
        B1["Control: current L3 model"]
        B2["Exp 2a: new cross-encoder"]
        B3["Exp 2b: freshness prior tweak"]
    end

    subgraph L3["Layer 3 — Presentation"]
        C1["Control: current SERP"]
        C2["Exp 3a: longer snippets"]
        C3["Exp 3b: new vertical layout"]
    end

    LAYERS --> L1 & L2 & L3

    L1 & L2 & L3 --> COMBINE["A user may be in one experiment<br/>from EACH layer simultaneously"]

    COMBINE --> WHY["Why layers work:<br/>layers are chosen to be<br/>approximately independent,<br/>so their effects do not confound.<br/><br/>⚠️ If two experiments COULD interact,<br/>they must be placed in the SAME layer<br/>so a user gets at most one."]

    WHY --> MEASURE["Measurement"]

    MEASURE --> ME1["Quality: human side-by-side ratings"]
    MEASURE --> ME2["Behaviour: CTR, long clicks,<br/>reformulation, abandonment"]
    MEASURE --> ME3["Performance: latency, error rate"]
    MEASURE --> ME4["Cost: CPU, RAM, cache hit rate"]

    ME1 & ME2 & ME3 & ME4 --> STAT["Statistical analysis"]
    STAT --> ST1["Sufficient power before reading results"]
    STAT --> ST2["Multiple-comparison correction<br/>— with 100s of experiments,<br/>5% of them show p<0.05 by chance"]
    STAT --> ST3["Guardrail metrics must not regress,<br/>even if the primary metric improves"]
    STAT --> ST4["Novelty effects: re-measure after<br/>the initial curiosity decays"]

    ST1 & ST2 & ST3 & ST4 --> DECIDE{"Ship?"}
    DECIDE -->|"Yes"| RAMP["Progressive ramp — Chapter 12"]
    DECIDE -->|"No"| LEARN["Record the negative result<br/>— it has real value"]

    classDef exp fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef stat fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef warn fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class A2,A3,B2,B3,C2,C3 exp
    class ST1,ST2,ST3,ST4 stat
    class WHY,ST2 warn
```

### The two statistical traps that matter most

**Multiple comparisons.** Running 400 experiments at p < 0.05 means ~20 will appear significant purely by chance. Without correction (Benjamini–Hochberg or similar), an organisation systematically ships noise and slowly degrades its own product while every individual decision looked justified. This is not a theoretical concern; it is the default outcome of running many experiments without correction.

**Guardrail metrics.** A change that improves CTR by 2 % while increasing latency by 30 ms and reducing long-click rate is a *loss*, not a win. Every experiment must declare guardrails in advance — latency, error rate, human-rated quality, revenue-neutral checks — and a guardrail regression blocks the launch regardless of how good the primary metric looks. Declaring them *in advance* is the part that matters: post-hoc guardrails are indistinguishable from rationalisation.

---

## 13.4 Alerting on symptoms, not causes

> **Diagram D-67 — SLO-based alerting**

```mermaid
flowchart TB
    subgraph GOOD["✅ Alert on these — user-visible symptoms"]
        A1["SLO burn rate<br/>'error budget will exhaust in 4 h'"]
        A2["p99 latency above SLO<br/>sustained over a window"]
        A3["Result-quality score drop<br/>on golden queries"]
        A4["Index coverage below threshold"]
        A5["Index freshness stalled<br/>— age distribution shifting"]
        A6["🚨 Politeness violations > 0<br/>— a trust breach, page immediately"]
    end

    subgraph BAD["⚠️ Do NOT page on these — causes, not symptoms"]
        B1["Single machine down<br/>→ expected, handled automatically"]
        B2["CPU above 80%<br/>→ that is called utilization"]
        B3["A cache miss occurred"]
        B4["One leaf slow<br/>→ hedging exists for exactly this"]
        B5["Disk 70% full<br/>→ ticket, not a page"]
    end

    A1 & A2 & A3 & A4 & A5 & A6 --> PAGE["📟 Page on-call"]
    B1 & B2 & B3 & B4 & B5 --> DASH["📊 Dashboard / ticket only"]

    PAGE --> RB["Runbook<br/>every alert links to one"]
    RB --> ACT{"Actionable?"}
    ACT -->|"No"| DELETE["🗑️ Delete the alert.<br/>An unactionable page is<br/>pure alert fatigue."]
    ACT -->|"Yes"| RESP["Respond, mitigate, then<br/>investigate root cause"]

    RESP --> PM["Blameless postmortem"]
    PM --> FIX["System change so the same<br/>failure cannot recur"]
    FIX --> GOOD

    subgraph BURN["Multi-window burn-rate alerting"]
        W1["Fast burn: 2% of budget in 1 h<br/>→ page immediately"]
        W2["Slow burn: 10% of budget in 6 h<br/>→ ticket, investigate today"]
        W3["Two windows together suppress<br/>both false pages and slow leaks"]
    end

    A1 -.implemented as.-> BURN

    classDef good fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef bad fill:#fed7aa,stroke:#c2410c,color:#1c1917
    classDef crit fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class A1,A2,A3,A4,A5 good
    class B1,B2,B3,B4,B5 bad
    class A6,DELETE crit
```

**"CPU above 80 %" is the canonical bad alert.** In a system designed for graceful degradation, high utilisation is *the goal* — idle capacity is wasted money. Users do not experience CPU; they experience latency and result quality. Alert on what users experience, and let the system handle causes automatically.

**The `DELETE` node is the most under-used tool in operations.** Every alert that fires without a clear action trains the on-call engineer to ignore alerts. Alert fatigue is not a personal failing; it is a predictable consequence of unactionable alerts, and the fix is deletion, not discipline.

---

## 13.5 The metrics that matter, by audience

| Audience | Primary metrics |
|---|---|
| **On-call** | SLO burn rate · error rate · p99 latency · coverage |
| **Search quality** | NDCG on golden set · side-by-side wins · long-click rate · reformulation rate |
| **Crawl** | Fetch success rate · politeness violations (0) · discovery latency · frontier depth |
| **Index** | Build success · publish latency · index size · freshness age distribution |
| **Capacity** | QPS vs provisioned · cache hit rate · cost per 1,000 queries · N-1 headroom |
| **Leadership** | Availability · task success rate · cost per query trend |

**Politeness violations get a hard zero target** and belong on the paging list, which is unusual for a metric with no direct user impact. The justification: crawling is a privilege extended by the entire web on the assumption that the crawler behaves. A violation is a breach of that trust, it damages the operator's ability to crawl at all, and unlike a latency regression it cannot be fixed retroactively.

---

<div align="center">

**← [12 · Reliability](12-reliability.md)** · [🏠 Home](../../README.md) · **[14 · Spam & Security](14-security-abuse.md) →** · [🇸🇦 العربية](../ar/13-observability.md)

</div>
