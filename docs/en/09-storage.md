<div align="right">

**[🇸🇦 اقرأ بالعربية](../ar/09-storage.md)**

</div>

# Chapter 09 · Storage Infrastructure

**← [08 · Ranking](08-ranking.md)** · [🏠 Home](../../README.md) · **[10 · Caching](10-caching.md) →**

---

## 9.1 Four storage problems, four different systems

A common beginner instinct is to reach for one database. This system cannot: it has four workloads whose requirements are mutually contradictory.

| Workload | Volume | Access pattern | Consistency | Right tool |
|---|---:|---|---|---|
| Raw crawled pages, index segments | 10s of PB | Sequential write, sequential read, immutable | Eventual | **Distributed file system** |
| Per-URL crawl metadata, document attachments | 10s of PB | Random read/write by key, sparse columns | Eventual, single-row atomic | **Wide-column store** |
| Shard assignment, index version, config | KB–GB | Read-heavy, tiny writes, must be correct | **Strong / linearizable** | **Coordination service** |
| Legal removals, webmaster settings, quotas | GB–TB | Transactional, cross-region | **Strong, external consistency** | **Globally consistent database** |

> **Diagram D-46 · The storage stack**

```mermaid
flowchart TB
    subgraph APP["Application layer"]
        A1["Crawler"]
        A2["Content processor"]
        A3["Index builder"]
        A4["Serving tier"]
        A5["Control plane"]
    end

    subgraph COMPUTE["Compute frameworks"]
        C1["Batch: MapReduce-style"]
        C2["Streaming: incremental processors"]
        C3["Graph: PageRank engine"]
    end

    subgraph STRUCTURED["Structured storage"]
        B["Wide-column store · Bigtable-class<br/>row key → column families → versioned cells<br/>sorted by row key · range-partitioned into tablets<br/>LSM: memtable + SSTables + compaction"]
        SP["Globally consistent DB: Spanner-class<br/>Paxos groups · synchronized clocks<br/>externally consistent transactions"]
    end

    subgraph COORD["Coordination"]
        CH["Lock service: Chubby-class<br/>Paxos-replicated<br/>leader election · config · naming<br/>small files, high read volume"]
    end

    subgraph FS["Distributed file system"]
        G["GFS / Colossus-class<br/>append-optimized · chunked<br/>metadata master + chunkservers<br/>replication or Reed-Solomon erasure coding"]
    end

    subgraph HW["Physical"]
        H1["Commodity servers with local disks"]
        H2["Clos datacenter fabric"]
        H3["Cluster scheduler: Borg-class"]
    end

    A1 & A2 --> B
    A3 --> C1 & C3
    C1 & C2 & C3 --> B
    C1 --> G
    A3 --> G
    A4 --> G
    A5 --> CH & SP
    A4 --> CH

    B --> G
    SP --> G
    CH --> HW
    G --> HW
    COMPUTE --> H3

    classDef app fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef struct fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef coord fill:#fecaca,stroke:#b91c1c,color:#1c1917
    classDef fs fill:#e9d5ff,stroke:#7e22ce,color:#1c1917
    class A1,A2,A3,A4,A5 app
    class B,SP struct
    class CH,SP coord
    class G fs
```

**Note the layering:** the wide-column store does not manage disks; it stores its files *on the distributed file system*. The file system does not manage consistency of metadata: it uses the coordination service for master election. Each layer solves exactly one problem and delegates the rest. This is why the stack has survived two decades of hardware change.

---

## 9.2 The distributed file system

Designed around one observation from the Google File System paper (2003): at this scale, **component failure is the norm, not the exception**. With 10,000 machines and a 3-year MTBF per machine, something fails roughly every 15 minutes. The file system must treat failure as an ordinary event, not an emergency.

> **Diagram D-47 · Write path and failure handling**

```mermaid
sequenceDiagram
    autonumber
    participant CL as Client
    participant M as Metadata Master
    participant P as Primary Chunkserver
    participant S1 as Replica 2
    participant S2 as Replica 3

    Note over CL,M: Metadata path · small, cached
    CL->>M: append to /crawl/segment-4711
    M->>M: allocate chunk, pick 3 replicas<br/>across distinct racks and power domains
    M->>P: grant lease (primary for 60 s)
    M-->>CL: chunk handle + replica locations
    Note over CL,M: Client caches this:<br/>master is never in the data path

    Note over CL,S2: Data path · bulk, master-free
    CL->>P: push data (pipelined)
    P->>S1: forward data
    S1->>S2: forward data
    Note over CL,S2: Linear pipeline uses full<br/>outbound bandwidth of every node

    CL->>P: commit append
    P->>P: assign serial order (offset)
    P->>S1: apply at offset
    P->>S2: apply at offset
    S1-->>P: ack
    S2-->>P: ack
    P-->>CL: success + offset

    alt A replica fails to ack
        P-->>CL: partial failure
        CL->>P: retry append
        Note over CL,S2: Record may appear twice<br/>→ at-least-once semantics.<br/>Readers MUST dedupe by<br/>record checksum or id.
    end

    Note over M: Master continuously:<br/>heartbeats chunkservers,<br/>re-replicates under-replicated chunks,<br/>rebalances hot chunks
```

### The design choices that matter

| Choice | Rationale |
|---|---|
| **Huge chunks (64–128 MB)** | Cuts metadata volume by ~1000×, letting all metadata live in the master's RAM |
| **Master out of the data path** | Clients cache locations and stream directly to chunkservers; the master never becomes a bandwidth bottleneck |
| **Append-optimised, not random-write** | Matches the actual workload: crawl output and index segments are written once, read many times |
| **Relaxed consistency: at-least-once append** | Enormously simpler and faster than exactly-once; applications dedupe by record checksum |
| **Replication across failure domains** | Three replicas on distinct racks and power domains, correlated failure is what kills you |
| **Erasure coding for cold data** | Reed-Solomon gives comparable durability at ~1.4× overhead instead of 3× |

**The at-least-once semantics deserve emphasis** because it is the trade most often missed. The file system does *not* promise a record appears exactly once. It promises the record appears *at least* once, atomically, at some offset. Every consumer must be idempotent. That single relaxation is what makes the write path fast and the failure handling simple, and it is only acceptable because [Chapter 02](02-requirements.md) established that the data plane tolerates eventual consistency.

---

## 9.3 The wide-column store

The crawl database is the canonical use case, and it is exactly the workload that motivated Bigtable: **sparse, versioned, enormous, keyed by URL.**

> **Diagram D-48 · Crawl table schema and physical layout**

```mermaid
flowchart TB
    subgraph LOGICAL["Logical model: sparse multidimensional map"]
        RK["Row key: REVERSED hostname + path<br/>com.example.www/products/item-42<br/><br/>⚠️ Reversal is the critical trick:<br/>all pages of one site sort together,<br/>so a site scan is a contiguous range read"]

        RK --> CF1["Column family: contents<br/>contents:html @ t1, t2, t3<br/>(versioned · keeps crawl history)"]
        RK --> CF2["Column family: metadata<br/>metadata:fetch_time<br/>metadata:http_status<br/>metadata:content_hash<br/>metadata:simhash<br/>metadata:language"]
        RK --> CF3["Column family: anchor<br/>anchor:com.cnn.www = 'CNN homepage'<br/>anchor:com.bbc = 'see also'<br/>(one column per linking site:<br/>sparse, unbounded, no schema change)"]
        RK --> CF4["Column family: signals<br/>signals:pagerank<br/>signals:spam_score<br/>signals:quality"]
    end

    subgraph PHYSICAL["Physical layout"]
        T["Table sorted by row key"]
        T --> TB1[("Tablet 1<br/>com.a… → com.f…")]
        T --> TB2[("Tablet 2<br/>com.f… → com.m…")]
        T --> TB3[("Tablet N<br/>…")]

        TB1 --> TS["Tablet server owns tablet<br/>splits when too large,<br/>merges when too small"]
        TS --> MEM["Memtable: sorted, in RAM<br/>absorbs all writes"]
        TS --> WAL["Write-ahead log on DFS<br/>durability before ack"]
        MEM -->|"flush"| SST1[("SSTable 1: immutable")]
        MEM -->|"flush"| SST2[("SSTable 2: immutable")]
        SST1 & SST2 -->|"compaction"| SST3[("Merged SSTable")]
        SST3 --> DFS[("All SSTables live on the<br/>distributed file system:<br/>tablet servers hold NO durable state")]
    end

    LOGICAL --> PHYSICAL

    classDef key fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef fam fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef phys fill:#e9d5ff,stroke:#7e22ce,color:#1c1917
    class RK key
    class CF1,CF2,CF3,CF4 fam
    class TB1,TB2,TB3,SST1,SST2,SST3,DFS phys
```

### Why the reversed row key is the whole design

`www.example.com/page` becomes `com.example.www/page`. Because the table is sorted by row key:

- All pages from one host land in **one contiguous key range** → crawling or reprocessing a whole site is a single efficient range scan, not 10,000 random reads.
- All hosts under one registrable domain are adjacent → domain-level aggregation (spam scoring, quota enforcement) is also a range scan.
- Load distributes reasonably across tablets, because hostnames are diverse.

**Row key design is the single highest-leverage decision** in a wide-column store. It determines which queries are cheap range scans and which are expensive scatter-gathers, and unlike an index in a relational database, it cannot be added afterwards.

### Why tablet servers hold no durable state

All SSTables live on the distributed file system. A tablet server is pure cache and coordination. If one dies, another simply takes ownership of the tablet, replays the write-ahead log, and continues: **no data movement is required**. Recovery is seconds, not hours. This separation of compute from storage is the reason the system tolerates constant machine churn, and it is the same principle that later became standard in cloud data warehouses.

### The consistency contract

| Guarantee | Provided? |
|---|---|
| Single-row read/write atomicity | ✅ Yes, across all column families |
| Multi-row transactions | ❌ No: deliberately omitted |
| Cross-table transactions | ❌ No |
| Ordered scans by row key | ✅ Yes: the core capability |
| Per-cell versioning with TTL/GC | ✅ Yes |

**Dropping multi-row transactions is what buys the scale.** Distributed transactions require coordination across tablet servers, which introduces the very cross-machine dependency that limits throughput. The application layer compensates: the crawler is designed so that every operation is a single-row update, and the few genuinely transactional needs are pushed to the globally consistent database.

---

## 9.4 Strong consistency where it is genuinely needed

A small fraction of data cannot tolerate eventual consistency. From [Chapter 02](02-requirements.md): shard assignment, index version, legal removals, webmaster authorisation.

> **Diagram D-49 · Choosing a store**

```mermaid
flowchart TB
    START["New data to store"] --> Q1{"Size?"}

    Q1 -->|"PB scale"| Q2{"Access pattern?"}
    Q1 -->|"GB–TB"| Q5{"Needs cross-row<br/>transactions?"}
    Q1 -->|"KB–MB"| Q7{"Needs consensus<br/>and watch/notify?"}

    Q2 -->|"Sequential, whole files,<br/>write-once"| DFS["📁 Distributed file system<br/>index segments · raw crawl archives ·<br/>MapReduce intermediates"]
    Q2 -->|"Random access by key,<br/>sparse columns, versioned"| WC["🗄️ Wide-column store<br/>crawl DB · doc attachments ·<br/>link graph · seen-URL exact store"]

    Q5 -->|"Yes, and must be<br/>correct across regions"| SPAN["🌍 Globally consistent DB<br/>legal removals · webmaster settings ·<br/>quotas · billing-grade data"]
    Q5 -->|"No"| WC

    Q7 -->|"Yes"| LOCK["🔒 Lock / coordination service<br/>shard→server assignment ·<br/>live index version pointer ·<br/>leader election · service discovery"]
    Q7 -->|"No"| CFG["📋 Config distribution<br/>pushed with the binary"]

    DFS & WC --> EV["Eventual consistency<br/>✅ cheap · scales horizontally<br/>⚠️ readers must be idempotent"]
    SPAN & LOCK --> ST["Strong consistency<br/>✅ correct by construction<br/>⚠️ costs a consensus round trip<br/>⚠️ capacity is bounded"]

    classDef ev fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef st fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class DFS,WC,EV ev
    class SPAN,LOCK,ST st
```

### The coordination service is small on purpose

A Chubby-class lock service holds only kilobytes: which server owns which shard, which index version is live, who is the current master of each subsystem. It is Paxos-replicated across five nodes, so writes cost a consensus round trip: expensive, but there are almost none.

Its most important property is not locking but **watch/notify with a consistent view**: every serving node watches the "live index version" node, and when it changes, all nodes learn within milliseconds and switch together. Without this, a rolling index update would leave different servers answering from different index versions: producing results that are individually valid but collectively incoherent, and nearly impossible to debug.

**The failure mode to respect:** the coordination service is a *hard* dependency for the control plane. If it is unavailable, no shard can be reassigned and no index version can be published. Systems that depend on it must be designed to keep serving with their last-known-good state indefinitely, so that a coordination outage degrades *change* rather than degrading *service*.

---

## 9.5 Replication and geography

> **Diagram D-50 · Data placement across regions**

```mermaid
flowchart TB
    subgraph GLOBAL["Global: written once, read everywhere"]
        G1[("Index segments<br/>immutable · pushed to every region")]
        G2[("Lexicon and global term statistics")]
        G3[("Ranking model artifacts")]
    end

    subgraph REGIONAL["Regional: one authoritative copy per region"]
        R1[("Crawl database<br/>partitioned by host geography")]
        R2[("Link graph")]
        R3[("Raw content archive")]
    end

    subgraph STRONG["Globally consistent: small, transactional"]
        S1[("Legal removals: region-scoped rules")]
        S2[("Webmaster verification and settings")]
        S3[("Shard assignment and live index version")]
    end

    subgraph LOCAL["Cell-local: regenerable, never replicated"]
        L1[("Result caches")]
        L2[("Snippet caches")]
        L3[("In-memory index replicas")]
    end

    G1 --> DC1["🌎 Region: Americas"]
    G1 --> DC2["🌍 Region: EMEA"]
    G1 --> DC3["🌏 Region: APAC"]

    S1 & S2 & S3 -.Paxos replicated.-> DC1 & DC2 & DC3
    R1 --> DC1
    DC1 & DC2 & DC3 --> L1

    subgraph RULES["Placement rules"]
        P1["Immutable + large → copy everywhere, cache forever"]
        P2["Mutable + large → keep one authoritative regional copy"]
        P3["Mutable + small + must be correct → consensus-replicate"]
        P4["Regenerable → never replicate, just rebuild"]
        P5["Regulated data → residency constraints override all of the above"]
    end

    classDef glob fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef reg fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef str fill:#fecaca,stroke:#b91c1c,color:#1c1917
    classDef loc fill:#e5e7eb,stroke:#4b5563,color:#1c1917
    class G1,G2,G3 glob
    class R1,R2,R3 reg
    class S1,S2,S3 str
    class L1,L2,L3 loc
```

**The rule that does most of the work is P1.** Index segments are immutable, so replicating them globally has no consistency cost whatsoever: a copy can never be stale relative to another copy, because neither will ever change. Immutability, chosen back in [Chapter 06](06-indexing.md) for lock-free reads, turns out to also make global geo-replication trivial. This is a recurring pattern worth internalising: **immutability converts a distributed-consistency problem into a distribution problem**, and distribution is much easier.

---

## 9.6 Failure handling summary

| Failure | Detection | Response | User impact |
|---|---|---|---|
| Single disk | Checksum mismatch on read | Read from another replica; re-replicate the chunk | None |
| Chunkserver | Missed heartbeats | Master re-replicates its chunks elsewhere | None |
| Tablet server | Lock service lease expiry | Another server loads the tablet, replays the WAL | Seconds of unavailability for that key range |
| Metadata master | Lease expiry in coordination service | Standby promoted, replays operation log | Seconds; existing reads continue from cached locations |
| Rack | Correlated heartbeat loss | Replicas on other racks serve; re-replicate | None: this is why placement spans racks |
| Whole region | Health checks + traffic monitoring | Global LB drains traffic to other regions | Higher latency for affected users; see [Ch 12](12-reliability.md) |
| Coordination service | Quorum loss | **Control plane freezes** | Serving continues on last-known-good state |
| Silent data corruption | Per-block checksums, background scrubbing | Repair from a good replica | None if caught by scrubbing |

**Silent corruption deserves the last word.** At petabyte scale, undetectable bit rot is a statistical certainty rather than a possibility. Every block carries a checksum, checksums are verified on every read, and a background scrubber continuously re-reads cold data to find corruption *before* anyone requests it. Without background scrubbing, you discover corruption only when you need the data, which is precisely the worst moment.

---

<div align="center">

**← [08 · Ranking](08-ranking.md)** · [🏠 Home](../../README.md) · **[10 · Caching](10-caching.md) →** · [🇸🇦 العربية](../ar/09-storage.md)

</div>
