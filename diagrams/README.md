<div align="center">

# 📐 Diagram Index · فهرس المخططات

**78 diagrams · 9 diagram types · rendered natively by GitHub**

[🏠 Home](../README.md) · [🇸🇦 الرئيسية](../README.ar.md) · [📚 Start reading](../docs/en/01-overview.md)

</div>

---

## About this folder

Every diagram in this repository is written in [Mermaid](https://mermaid.js.org/) and renders directly on GitHub: no images, no build step, no external assets. Each one also lives here as a standalone `.mmd` file so it can be reused in slides, papers or talks.

Diagram IDs (`D-01` … `D-78`) are **stable**. They are safe to cite from issues, notes or other documents.

### Rendering a single diagram to SVG or PNG

```bash
npm install -g @mermaid-js/mermaid-cli
```

```bash
mmdc -i diagrams/D-01-end-to-end-system-context.mmd -o D-01.svg -b transparent
```

### Rendering all of them at once

```bash
for f in diagrams/*.mmd; do mmdc -i "$f" -o "assets/rendered/$(basename "${f%.mmd}").svg" -b transparent; done
```

---

## Colour conventions

| Colour | Meaning | Used for |
|---|---|---|
| 🟨 Amber | Offline / batch | Crawling, processing, precomputation |
| 🟦 Blue | Index construction & data structures | Index build, posting lists, storage internals |
| 🟩 Green | Online serving path | Frontend, retrieval, ranking, good outcomes |
| 🟪 Purple | Persistent storage | Databases, file systems, index segments |
| 🟥 Red | Risk, failure, or cost | Failure modes, attacks, expensive operations |
| ⬜ Grey | External / physical | Hardware, the open web, client-side |

---

## Diagram types used

| Type | Mermaid keyword | Count |
|---|---|---:|
| Flowchart | `flowchart` | 55 |
| Sequence | `sequenceDiagram` | 7 |
| State machine | `stateDiagram-v2` | 5 |
| Mindmap | `mindmap` | 3 |
| Timeline | `gantt` | 3 |
| Pie | `pie` | 2 |
| Quadrant | `quadrantChart` | 1 |
| ER model | `erDiagram` | 1 |
| Class model | `classDiagram` | 1 |

---

## Full index

| ID | Title | Type | Chapter | Source |
|---|---|---|---|---|
| `D-01` | End-to-end system context | Flowchart | [Home](../README.md) | [`.mmd`](D-01-end-to-end-system-context.mmd) |
| `D-02` | The three clock speeds | Flowchart | [Home](../README.md) | [`.mmd`](D-02-the-three-clock-speeds.mmd) |
| `D-03` | Mindmap of every subsystem covered in this repo | Mindmap | [Home](../README.md) | [`.mmd`](D-03-mindmap-of-every-subsystem-covered-in-this-repo.mmd) |
| `D-04` | Why the naïve design collapses | Flowchart | [Ch 01 · Overview](../docs/en/01-overview.md) | [`.mmd`](D-04-why-the-na-ve-design-collapses.mmd) |
| `D-05` | Layered architecture | Flowchart | [Ch 01 · Overview](../docs/en/01-overview.md) | [`.mmd`](D-05-layered-architecture.mmd) |
| `D-06` | Life of a query, end to end | Sequence | [Ch 01 · Overview](../docs/en/01-overview.md) | [`.mmd`](D-06-life-of-a-query-end-to-end.mmd) |
| `D-07` | Document lifecycle state machine | State machine | [Ch 01 · Overview](../docs/en/01-overview.md) | [`.mmd`](D-07-document-lifecycle-state-machine.mmd) |
| `D-08` | Requirements decomposition | Mindmap | [Ch 02 · Requirements](../docs/en/02-requirements.md) | [`.mmd`](D-08-requirements-decomposition.mmd) |
| `D-09` | Latency budget allocation (p99 = 300 ms) | Pie | [Ch 02 · Requirements](../docs/en/02-requirements.md) | [`.mmd`](D-09-latency-budget-allocation-p99-300-ms.mmd) |
| `D-10` | SLO hierarchy and error-budget policy | Flowchart | [Ch 02 · Requirements](../docs/en/02-requirements.md) | [`.mmd`](D-10-slo-hierarchy-and-error-budget-policy.mmd) |
| `D-11` | Design pressure map | Quadrant | [Ch 02 · Requirements](../docs/en/02-requirements.md) | [`.mmd`](D-11-design-pressure-map.mmd) |
| `D-12` | Estimation dependency chain | Flowchart | [Ch 03 · Capacity](../docs/en/03-capacity-estimation.md) | [`.mmd`](D-12-estimation-dependency-chain.mmd) |
| `D-13` | Where the bytes go | Pie | [Ch 03 · Capacity](../docs/en/03-capacity-estimation.md) | [`.mmd`](D-13-where-the-bytes-go.mmd) |
| `D-14` | Fleet sizing derivation | Flowchart | [Ch 03 · Capacity](../docs/en/03-capacity-estimation.md) | [`.mmd`](D-14-fleet-sizing-derivation.mmd) |
| `D-15` | Crawler fleet architecture | Flowchart | [Ch 04 · Crawling](../docs/en/04-crawling.md) | [`.mmd`](D-15-crawler-fleet-architecture.mmd) |
| `D-16` | Two-stage frontier: priority in front, politeness behind | Flowchart | [Ch 04 · Crawling](../docs/en/04-crawling.md) | [`.mmd`](D-16-two-stage-frontier-priority-in-front-politeness-.mmd) |
| `D-17` | Per-host politeness state machine | State machine | [Ch 04 · Crawling](../docs/en/04-crawling.md) | [`.mmd`](D-17-per-host-politeness-state-machine.mmd) |
| `D-18` | Life of a single fetch | Sequence | [Ch 04 · Crawling](../docs/en/04-crawling.md) | [`.mmd`](D-18-life-of-a-single-fetch.mmd) |
| `D-19` | SimHash near-duplicate pipeline | Flowchart | [Ch 04 · Crawling](../docs/en/04-crawling.md) | [`.mmd`](D-19-simhash-near-duplicate-pipeline.mmd) |
| `D-20` | Adaptive recrawl scheduling | Flowchart | [Ch 04 · Crawling](../docs/en/04-crawling.md) | [`.mmd`](D-20-adaptive-recrawl-scheduling.mmd) |
| `D-21` | Content processing pipeline | Flowchart | [Ch 05 · Processing](../docs/en/05-content-processing.md) | [`.mmd`](D-21-content-processing-pipeline.mmd) |
| `D-22` | Main-content extraction | Flowchart | [Ch 05 · Processing](../docs/en/05-content-processing.md) | [`.mmd`](D-22-main-content-extraction.mmd) |
| `D-23` | Language-specific tokenization paths | Flowchart | [Ch 05 · Processing](../docs/en/05-content-processing.md) | [`.mmd`](D-23-language-specific-tokenization-paths.mmd) |
| `D-24` | Link graph and derived document data model | ER model | [Ch 05 · Processing](../docs/en/05-content-processing.md) | [`.mmd`](D-24-link-graph-and-derived-document-data-model.mmd) |
| `D-25` | Document state through processing | State machine | [Ch 05 · Processing](../docs/en/05-content-processing.md) | [`.mmd`](D-25-document-state-through-processing.mmd) |
| `D-26` | Anatomy of the inverted index | Flowchart | [Ch 06 · Indexing](../docs/en/06-indexing.md) | [`.mmd`](D-26-anatomy-of-the-inverted-index.mmd) |
| `D-27` | Posting list compression pipeline | Flowchart | [Ch 06 · Indexing](../docs/en/06-indexing.md) | [`.mmd`](D-27-posting-list-compression-pipeline.mmd) |
| `D-28` | Document partitioning vs term partitioning | Flowchart | [Ch 06 · Indexing](../docs/en/06-indexing.md) | [`.mmd`](D-28-document-partitioning-vs-term-partitioning.mmd) |
| `D-29` | Index construction pipeline | Flowchart | [Ch 06 · Indexing](../docs/en/06-indexing.md) | [`.mmd`](D-29-index-construction-pipeline.mmd) |
| `D-30` | Segment lifecycle and tiered merging | State machine | [Ch 06 · Indexing](../docs/en/06-indexing.md) | [`.mmd`](D-30-segment-lifecycle-and-tiered-merging.mmd) |
| `D-31` | Tiered index and fallthrough | Flowchart | [Ch 06 · Indexing](../docs/en/06-indexing.md) | [`.mmd`](D-31-tiered-index-and-fallthrough.mmd) |
| `D-32` | Index structures | Class model | [Ch 06 · Indexing](../docs/en/06-indexing.md) | [`.mmd`](D-32-index-structures.mmd) |
| `D-33` | Root, intermediate and leaf serving tree | Flowchart | [Ch 07 · Serving](../docs/en/07-serving.md) | [`.mmd`](D-33-root-intermediate-and-leaf-serving-tree.mmd) |
| `D-34` | Query understanding pipeline | Flowchart | [Ch 07 · Serving](../docs/en/07-serving.md) | [`.mmd`](D-34-query-understanding-pipeline.mmd) |
| `D-35` | Serving path including hedging and degradation | Sequence | [Ch 07 · Serving](../docs/en/07-serving.md) | [`.mmd`](D-35-serving-path-including-hedging-and-degradation.mmd) |
| `D-36` | Latency budget timeline (cache miss, p99 path) | Timeline | [Ch 07 · Serving](../docs/en/07-serving.md) | [`.mmd`](D-36-latency-budget-timeline-cache-miss-p99-path.mmd) |
| `D-37` | Vertical triggering and blending | Flowchart | [Ch 07 · Serving](../docs/en/07-serving.md) | [`.mmd`](D-37-vertical-triggering-and-blending.mmd) |
| `D-38` | Query-biased snippet generation | Flowchart | [Ch 07 · Serving](../docs/en/07-serving.md) | [`.mmd`](D-38-query-biased-snippet-generation.mmd) |
| `D-39` | The retrieval and ranking funnel | Flowchart | [Ch 08 · Ranking](../docs/en/08-ranking.md) | [`.mmd`](D-39-the-retrieval-and-ranking-funnel.mmd) |
| `D-40` | Where ranking signals come from | Mindmap | [Ch 08 · Ranking](../docs/en/08-ranking.md) | [`.mmd`](D-40-where-ranking-signals-come-from.mmd) |
| `D-41` | PageRank computation at scale | Flowchart | [Ch 08 · Ranking](../docs/en/08-ranking.md) | [`.mmd`](D-41-pagerank-computation-at-scale.mmd) |
| `D-42` | Learning-to-rank training and deployment loop | Flowchart | [Ch 08 · Ranking](../docs/en/08-ranking.md) | [`.mmd`](D-42-learning-to-rank-training-and-deployment-loop.mmd) |
| `D-43` | Dense retrieval and hybrid fusion | Flowchart | [Ch 08 · Ranking](../docs/en/08-ranking.md) | [`.mmd`](D-43-dense-retrieval-and-hybrid-fusion.mmd) |
| `D-44` | Context application in ranking | Flowchart | [Ch 08 · Ranking](../docs/en/08-ranking.md) | [`.mmd`](D-44-context-application-in-ranking.mmd) |
| `D-45` | Post-ranking result-set optimisation | Flowchart | [Ch 08 · Ranking](../docs/en/08-ranking.md) | [`.mmd`](D-45-post-ranking-result-set-optimisation.mmd) |
| `D-46` | The storage stack | Flowchart | [Ch 09 · Storage](../docs/en/09-storage.md) | [`.mmd`](D-46-the-storage-stack.mmd) |
| `D-47` | Write path and failure handling | Sequence | [Ch 09 · Storage](../docs/en/09-storage.md) | [`.mmd`](D-47-write-path-and-failure-handling.mmd) |
| `D-48` | Crawl table schema and physical layout | Flowchart | [Ch 09 · Storage](../docs/en/09-storage.md) | [`.mmd`](D-48-crawl-table-schema-and-physical-layout.mmd) |
| `D-49` | Choosing a store | Flowchart | [Ch 09 · Storage](../docs/en/09-storage.md) | [`.mmd`](D-49-choosing-a-store.mmd) |
| `D-50` | Data placement across regions | Flowchart | [Ch 09 · Storage](../docs/en/09-storage.md) | [`.mmd`](D-50-data-placement-across-regions.mmd) |
| `D-51` | Query popularity distribution and cacheability | Flowchart | [Ch 10 · Caching](../docs/en/10-caching.md) | [`.mmd`](D-51-query-popularity-distribution-and-cacheability.mmd) |
| `D-52` | Multi-layer cache hierarchy | Flowchart | [Ch 10 · Caching](../docs/en/10-caching.md) | [`.mmd`](D-52-multi-layer-cache-hierarchy.mmd) |
| `D-53` | Cache coherence and invalidation paths | Flowchart | [Ch 10 · Caching](../docs/en/10-caching.md) | [`.mmd`](D-53-cache-coherence-and-invalidation-paths.mmd) |
| `D-54` | Metrics that actually matter | Flowchart | [Ch 10 · Caching](../docs/en/10-caching.md) | [`.mmd`](D-54-metrics-that-actually-matter.mmd) |
| `D-55` | Two-track indexing | Flowchart | [Ch 11 · Freshness](../docs/en/11-freshness.md) | [`.mmd`](D-55-two-track-indexing.mmd) |
| `D-56` | Observer-driven incremental indexing | Flowchart | [Ch 11 · Freshness](../docs/en/11-freshness.md) | [`.mmd`](D-56-observer-driven-incremental-indexing.mmd) |
| `D-57` | Change discovery channels, ranked by latency | Flowchart | [Ch 11 · Freshness](../docs/en/11-freshness.md) | [`.mmd`](D-57-change-discovery-channels-ranked-by-latency.mmd) |
| `D-58` | Merging real-time and batch results | Sequence | [Ch 11 · Freshness](../docs/en/11-freshness.md) | [`.mmd`](D-58-merging-real-time-and-batch-results.mmd) |
| `D-59` | Global serving topology | Flowchart | [Ch 12 · Reliability](../docs/en/12-reliability.md) | [`.mmd`](D-59-global-serving-topology.mmd) |
| `D-60` | Multi-layer request routing | Sequence | [Ch 12 · Reliability](../docs/en/12-reliability.md) | [`.mmd`](D-60-multi-layer-request-routing.mmd) |
| `D-61` | Failure domains and blast radius | Flowchart | [Ch 12 · Reliability](../docs/en/12-reliability.md) | [`.mmd`](D-61-failure-domains-and-blast-radius.mmd) |
| `D-62` | Degradation ladder | State machine | [Ch 12 · Reliability](../docs/en/12-reliability.md) | [`.mmd`](D-62-degradation-ladder.mmd) |
| `D-63` | Cell drain and progressive rollout | Sequence | [Ch 12 · Reliability](../docs/en/12-reliability.md) | [`.mmd`](D-63-cell-drain-and-progressive-rollout.mmd) |
| `D-64` | The observability stack | Flowchart | [Ch 13 · Observability](../docs/en/13-observability.md) | [`.mmd`](D-64-the-observability-stack.mmd) |
| `D-65` | A single query's trace | Timeline | [Ch 13 · Observability](../docs/en/13-observability.md) | [`.mmd`](D-65-a-single-query-s-trace.mmd) |
| `D-66` | Overlapping layered experiments | Flowchart | [Ch 13 · Observability](../docs/en/13-observability.md) | [`.mmd`](D-66-overlapping-layered-experiments.mmd) |
| `D-67` | SLO-based alerting | Flowchart | [Ch 13 · Observability](../docs/en/13-observability.md) | [`.mmd`](D-67-slo-based-alerting.mmd) |
| `D-68` | Web spam techniques and where each is caught | Flowchart | [Ch 14 · Spam & Security](../docs/en/14-security-abuse.md) | [`.mmd`](D-68-web-spam-techniques-and-where-each-is-caught.mmd) |
| `D-69` | Link graph anomaly detection | Flowchart | [Ch 14 · Spam & Security](../docs/en/14-security-abuse.md) | [`.mmd`](D-69-link-graph-anomaly-detection.mmd) |
| `D-70` | Attack surfaces and controls | Flowchart | [Ch 14 · Spam & Security](../docs/en/14-security-abuse.md) | [`.mmd`](D-70-attack-surfaces-and-controls.mmd) |
| `D-71` | Query data lifecycle and privacy controls | Flowchart | [Ch 14 · Spam & Security](../docs/en/14-security-abuse.md) | [`.mmd`](D-71-query-data-lifecycle-and-privacy-controls.mmd) |
| `D-72` | CAP positioning by subsystem | Flowchart | [Ch 15 · Trade-offs](../docs/en/15-tradeoffs.md) | [`.mmd`](D-72-cap-positioning-by-subsystem.mmd) |
| `D-73` | Rejected architectures and their failure points | Flowchart | [Ch 15 · Trade-offs](../docs/en/15-tradeoffs.md) | [`.mmd`](D-73-rejected-architectures-and-their-failure-points.mmd) |
| `D-74` | Architecture by corpus size | Flowchart | [Ch 15 · Trade-offs](../docs/en/15-tradeoffs.md) | [`.mmd`](D-74-architecture-by-corpus-size.mmd) |
| `D-75` | A 45-minute plan | Timeline | [Ch 16 · Interview](../docs/en/16-interview-guide.md) | [`.mmd`](D-75-a-45-minute-plan.mmd) |
| `D-76` | Whiteboard drawing order | Flowchart | [Ch 16 · Interview](../docs/en/16-interview-guide.md) | [`.mmd`](D-76-whiteboard-drawing-order.mmd) |
| `D-77` | Evaluation dimensions | Flowchart | [Ch 16 · Interview](../docs/en/16-interview-guide.md) | [`.mmd`](D-77-evaluation-dimensions.mmd) |
| `D-78` | Concept map | Flowchart | [Ch 17 · Glossary](../docs/en/17-glossary.md) | [`.mmd`](D-78-concept-map.mmd) |

---

<div dir="rtl" align="right">

## الفهرس الكامل بالعربية

كل مخطط في هذا المستودع مكتوب بلغة [Mermaid](https://mermaid.js.org/) ويُعرض مباشرةً داخل GitHub: بلا صور ولا خطوة بناء ولا أصول خارجية. وكل مخطط موجود هنا أيضًا كملف `.mmd` مستقل ليُعاد استخدامه في العروض التقديمية أو الأوراق أو المحاضرات.

ومعرّفات المخططات (`D-01` … `D-78`) **ثابتة**، فيمكن الإشارة إليها بأمان من التذاكر أو الملاحظات أو أي وثيقة أخرى.

### اصطلاحات الألوان

| اللون | المعنى | الاستخدام |
|---|---|---|
| 🟨 كهرماني | دون اتصال / دفعي | الزحف والمعالجة والحساب المسبق |
| 🟦 أزرق | بناء الفهرس وبنى البيانات | بناء الفهرس وقوائم الترحيل وتفاصيل التخزين |
| 🟩 أخضر | مسار الخدمة المباشر | الواجهة والاسترجاع والترتيب والنتائج الجيدة |
| 🟪 بنفسجي | تخزين دائم | قواعد البيانات وأنظمة الملفات ومقاطع الفهرس |
| 🟥 أحمر | خطر أو فشل أو تكلفة | أنماط الفشل والهجمات والعمليات المكلفة |
| ⬜ رمادي | خارجي / فيزيائي | العتاد والويب المفتوح وجانب العميل |

### أنواع المخططات المستخدمة

| النوع | كلمة Mermaid | العدد |
|---|---|---:|
| مخطط انسيابي | `flowchart` | 55 |
| مخطط تتابع | `sequenceDiagram` | 7 |
| آلة حالات | `stateDiagram-v2` | 5 |
| خريطة ذهنية | `mindmap` | 3 |
| خط زمني | `gantt` | 3 |
| قطاعية | `pie` | 2 |
| رباعية | `quadrantChart` | 1 |
| نموذج كيانات | `erDiagram` | 1 |
| نموذج أصناف | `classDiagram` | 1 |

### الفهرس

| المعرّف | العنوان | النوع | الفصل | المصدر |
|---|---|---|---|---|
| `D-01` | End-to-end system context | مخطط انسيابي | [الرئيسية](../README.ar.md) | [`.mmd`](D-01-end-to-end-system-context.mmd) |
| `D-02` | The three clock speeds | مخطط انسيابي | [الرئيسية](../README.ar.md) | [`.mmd`](D-02-the-three-clock-speeds.mmd) |
| `D-03` | Mindmap of every subsystem covered in this repo | خريطة ذهنية | [الرئيسية](../README.ar.md) | [`.mmd`](D-03-mindmap-of-every-subsystem-covered-in-this-repo.mmd) |
| `D-04` | Why the naïve design collapses | مخطط انسيابي | [الفصل 01 · نظرة عامة](../docs/ar/01-overview.md) | [`.mmd`](D-04-why-the-na-ve-design-collapses.mmd) |
| `D-05` | Layered architecture | مخطط انسيابي | [الفصل 01 · نظرة عامة](../docs/ar/01-overview.md) | [`.mmd`](D-05-layered-architecture.mmd) |
| `D-06` | Life of a query, end to end | مخطط تتابع | [الفصل 01 · نظرة عامة](../docs/ar/01-overview.md) | [`.mmd`](D-06-life-of-a-query-end-to-end.mmd) |
| `D-07` | Document lifecycle state machine | آلة حالات | [الفصل 01 · نظرة عامة](../docs/ar/01-overview.md) | [`.mmd`](D-07-document-lifecycle-state-machine.mmd) |
| `D-08` | Requirements decomposition | خريطة ذهنية | [الفصل 02 · المتطلبات](../docs/ar/02-requirements.md) | [`.mmd`](D-08-requirements-decomposition.mmd) |
| `D-09` | Latency budget allocation (p99 = 300 ms) | قطاعية | [الفصل 02 · المتطلبات](../docs/ar/02-requirements.md) | [`.mmd`](D-09-latency-budget-allocation-p99-300-ms.mmd) |
| `D-10` | SLO hierarchy and error-budget policy | مخطط انسيابي | [الفصل 02 · المتطلبات](../docs/ar/02-requirements.md) | [`.mmd`](D-10-slo-hierarchy-and-error-budget-policy.mmd) |
| `D-11` | Design pressure map | رباعية | [الفصل 02 · المتطلبات](../docs/ar/02-requirements.md) | [`.mmd`](D-11-design-pressure-map.mmd) |
| `D-12` | Estimation dependency chain | مخطط انسيابي | [الفصل 03 · السعة](../docs/ar/03-capacity-estimation.md) | [`.mmd`](D-12-estimation-dependency-chain.mmd) |
| `D-13` | Where the bytes go | قطاعية | [الفصل 03 · السعة](../docs/ar/03-capacity-estimation.md) | [`.mmd`](D-13-where-the-bytes-go.mmd) |
| `D-14` | Fleet sizing derivation | مخطط انسيابي | [الفصل 03 · السعة](../docs/ar/03-capacity-estimation.md) | [`.mmd`](D-14-fleet-sizing-derivation.mmd) |
| `D-15` | Crawler fleet architecture | مخطط انسيابي | [الفصل 04 · الزحف](../docs/ar/04-crawling.md) | [`.mmd`](D-15-crawler-fleet-architecture.mmd) |
| `D-16` | Two-stage frontier: priority in front, politeness behind | مخطط انسيابي | [الفصل 04 · الزحف](../docs/ar/04-crawling.md) | [`.mmd`](D-16-two-stage-frontier-priority-in-front-politeness-.mmd) |
| `D-17` | Per-host politeness state machine | آلة حالات | [الفصل 04 · الزحف](../docs/ar/04-crawling.md) | [`.mmd`](D-17-per-host-politeness-state-machine.mmd) |
| `D-18` | Life of a single fetch | مخطط تتابع | [الفصل 04 · الزحف](../docs/ar/04-crawling.md) | [`.mmd`](D-18-life-of-a-single-fetch.mmd) |
| `D-19` | SimHash near-duplicate pipeline | مخطط انسيابي | [الفصل 04 · الزحف](../docs/ar/04-crawling.md) | [`.mmd`](D-19-simhash-near-duplicate-pipeline.mmd) |
| `D-20` | Adaptive recrawl scheduling | مخطط انسيابي | [الفصل 04 · الزحف](../docs/ar/04-crawling.md) | [`.mmd`](D-20-adaptive-recrawl-scheduling.mmd) |
| `D-21` | Content processing pipeline | مخطط انسيابي | [الفصل 05 · المعالجة](../docs/ar/05-content-processing.md) | [`.mmd`](D-21-content-processing-pipeline.mmd) |
| `D-22` | Main-content extraction | مخطط انسيابي | [الفصل 05 · المعالجة](../docs/ar/05-content-processing.md) | [`.mmd`](D-22-main-content-extraction.mmd) |
| `D-23` | Language-specific tokenization paths | مخطط انسيابي | [الفصل 05 · المعالجة](../docs/ar/05-content-processing.md) | [`.mmd`](D-23-language-specific-tokenization-paths.mmd) |
| `D-24` | Link graph and derived document data model | نموذج كيانات | [الفصل 05 · المعالجة](../docs/ar/05-content-processing.md) | [`.mmd`](D-24-link-graph-and-derived-document-data-model.mmd) |
| `D-25` | Document state through processing | آلة حالات | [الفصل 05 · المعالجة](../docs/ar/05-content-processing.md) | [`.mmd`](D-25-document-state-through-processing.mmd) |
| `D-26` | Anatomy of the inverted index | مخطط انسيابي | [الفصل 06 · الفهرسة](../docs/ar/06-indexing.md) | [`.mmd`](D-26-anatomy-of-the-inverted-index.mmd) |
| `D-27` | Posting list compression pipeline | مخطط انسيابي | [الفصل 06 · الفهرسة](../docs/ar/06-indexing.md) | [`.mmd`](D-27-posting-list-compression-pipeline.mmd) |
| `D-28` | Document partitioning vs term partitioning | مخطط انسيابي | [الفصل 06 · الفهرسة](../docs/ar/06-indexing.md) | [`.mmd`](D-28-document-partitioning-vs-term-partitioning.mmd) |
| `D-29` | Index construction pipeline | مخطط انسيابي | [الفصل 06 · الفهرسة](../docs/ar/06-indexing.md) | [`.mmd`](D-29-index-construction-pipeline.mmd) |
| `D-30` | Segment lifecycle and tiered merging | آلة حالات | [الفصل 06 · الفهرسة](../docs/ar/06-indexing.md) | [`.mmd`](D-30-segment-lifecycle-and-tiered-merging.mmd) |
| `D-31` | Tiered index and fallthrough | مخطط انسيابي | [الفصل 06 · الفهرسة](../docs/ar/06-indexing.md) | [`.mmd`](D-31-tiered-index-and-fallthrough.mmd) |
| `D-32` | Index structures | نموذج أصناف | [الفصل 06 · الفهرسة](../docs/ar/06-indexing.md) | [`.mmd`](D-32-index-structures.mmd) |
| `D-33` | Root, intermediate and leaf serving tree | مخطط انسيابي | [الفصل 07 · الخدمة](../docs/ar/07-serving.md) | [`.mmd`](D-33-root-intermediate-and-leaf-serving-tree.mmd) |
| `D-34` | Query understanding pipeline | مخطط انسيابي | [الفصل 07 · الخدمة](../docs/ar/07-serving.md) | [`.mmd`](D-34-query-understanding-pipeline.mmd) |
| `D-35` | Serving path including hedging and degradation | مخطط تتابع | [الفصل 07 · الخدمة](../docs/ar/07-serving.md) | [`.mmd`](D-35-serving-path-including-hedging-and-degradation.mmd) |
| `D-36` | Latency budget timeline (cache miss, p99 path) | خط زمني | [الفصل 07 · الخدمة](../docs/ar/07-serving.md) | [`.mmd`](D-36-latency-budget-timeline-cache-miss-p99-path.mmd) |
| `D-37` | Vertical triggering and blending | مخطط انسيابي | [الفصل 07 · الخدمة](../docs/ar/07-serving.md) | [`.mmd`](D-37-vertical-triggering-and-blending.mmd) |
| `D-38` | Query-biased snippet generation | مخطط انسيابي | [الفصل 07 · الخدمة](../docs/ar/07-serving.md) | [`.mmd`](D-38-query-biased-snippet-generation.mmd) |
| `D-39` | The retrieval and ranking funnel | مخطط انسيابي | [الفصل 08 · الترتيب](../docs/ar/08-ranking.md) | [`.mmd`](D-39-the-retrieval-and-ranking-funnel.mmd) |
| `D-40` | Where ranking signals come from | خريطة ذهنية | [الفصل 08 · الترتيب](../docs/ar/08-ranking.md) | [`.mmd`](D-40-where-ranking-signals-come-from.mmd) |
| `D-41` | PageRank computation at scale | مخطط انسيابي | [الفصل 08 · الترتيب](../docs/ar/08-ranking.md) | [`.mmd`](D-41-pagerank-computation-at-scale.mmd) |
| `D-42` | Learning-to-rank training and deployment loop | مخطط انسيابي | [الفصل 08 · الترتيب](../docs/ar/08-ranking.md) | [`.mmd`](D-42-learning-to-rank-training-and-deployment-loop.mmd) |
| `D-43` | Dense retrieval and hybrid fusion | مخطط انسيابي | [الفصل 08 · الترتيب](../docs/ar/08-ranking.md) | [`.mmd`](D-43-dense-retrieval-and-hybrid-fusion.mmd) |
| `D-44` | Context application in ranking | مخطط انسيابي | [الفصل 08 · الترتيب](../docs/ar/08-ranking.md) | [`.mmd`](D-44-context-application-in-ranking.mmd) |
| `D-45` | Post-ranking result-set optimisation | مخطط انسيابي | [الفصل 08 · الترتيب](../docs/ar/08-ranking.md) | [`.mmd`](D-45-post-ranking-result-set-optimisation.mmd) |
| `D-46` | The storage stack | مخطط انسيابي | [الفصل 09 · التخزين](../docs/ar/09-storage.md) | [`.mmd`](D-46-the-storage-stack.mmd) |
| `D-47` | Write path and failure handling | مخطط تتابع | [الفصل 09 · التخزين](../docs/ar/09-storage.md) | [`.mmd`](D-47-write-path-and-failure-handling.mmd) |
| `D-48` | Crawl table schema and physical layout | مخطط انسيابي | [الفصل 09 · التخزين](../docs/ar/09-storage.md) | [`.mmd`](D-48-crawl-table-schema-and-physical-layout.mmd) |
| `D-49` | Choosing a store | مخطط انسيابي | [الفصل 09 · التخزين](../docs/ar/09-storage.md) | [`.mmd`](D-49-choosing-a-store.mmd) |
| `D-50` | Data placement across regions | مخطط انسيابي | [الفصل 09 · التخزين](../docs/ar/09-storage.md) | [`.mmd`](D-50-data-placement-across-regions.mmd) |
| `D-51` | Query popularity distribution and cacheability | مخطط انسيابي | [الفصل 10 · التخزين المؤقت](../docs/ar/10-caching.md) | [`.mmd`](D-51-query-popularity-distribution-and-cacheability.mmd) |
| `D-52` | Multi-layer cache hierarchy | مخطط انسيابي | [الفصل 10 · التخزين المؤقت](../docs/ar/10-caching.md) | [`.mmd`](D-52-multi-layer-cache-hierarchy.mmd) |
| `D-53` | Cache coherence and invalidation paths | مخطط انسيابي | [الفصل 10 · التخزين المؤقت](../docs/ar/10-caching.md) | [`.mmd`](D-53-cache-coherence-and-invalidation-paths.mmd) |
| `D-54` | Metrics that actually matter | مخطط انسيابي | [الفصل 10 · التخزين المؤقت](../docs/ar/10-caching.md) | [`.mmd`](D-54-metrics-that-actually-matter.mmd) |
| `D-55` | Two-track indexing | مخطط انسيابي | [الفصل 11 · الحداثة](../docs/ar/11-freshness.md) | [`.mmd`](D-55-two-track-indexing.mmd) |
| `D-56` | Observer-driven incremental indexing | مخطط انسيابي | [الفصل 11 · الحداثة](../docs/ar/11-freshness.md) | [`.mmd`](D-56-observer-driven-incremental-indexing.mmd) |
| `D-57` | Change discovery channels, ranked by latency | مخطط انسيابي | [الفصل 11 · الحداثة](../docs/ar/11-freshness.md) | [`.mmd`](D-57-change-discovery-channels-ranked-by-latency.mmd) |
| `D-58` | Merging real-time and batch results | مخطط تتابع | [الفصل 11 · الحداثة](../docs/ar/11-freshness.md) | [`.mmd`](D-58-merging-real-time-and-batch-results.mmd) |
| `D-59` | Global serving topology | مخطط انسيابي | [الفصل 12 · الموثوقية](../docs/ar/12-reliability.md) | [`.mmd`](D-59-global-serving-topology.mmd) |
| `D-60` | Multi-layer request routing | مخطط تتابع | [الفصل 12 · الموثوقية](../docs/ar/12-reliability.md) | [`.mmd`](D-60-multi-layer-request-routing.mmd) |
| `D-61` | Failure domains and blast radius | مخطط انسيابي | [الفصل 12 · الموثوقية](../docs/ar/12-reliability.md) | [`.mmd`](D-61-failure-domains-and-blast-radius.mmd) |
| `D-62` | Degradation ladder | آلة حالات | [الفصل 12 · الموثوقية](../docs/ar/12-reliability.md) | [`.mmd`](D-62-degradation-ladder.mmd) |
| `D-63` | Cell drain and progressive rollout | مخطط تتابع | [الفصل 12 · الموثوقية](../docs/ar/12-reliability.md) | [`.mmd`](D-63-cell-drain-and-progressive-rollout.mmd) |
| `D-64` | The observability stack | مخطط انسيابي | [الفصل 13 · المراقبة](../docs/ar/13-observability.md) | [`.mmd`](D-64-the-observability-stack.mmd) |
| `D-65` | A single query's trace | خط زمني | [الفصل 13 · المراقبة](../docs/ar/13-observability.md) | [`.mmd`](D-65-a-single-query-s-trace.mmd) |
| `D-66` | Overlapping layered experiments | مخطط انسيابي | [الفصل 13 · المراقبة](../docs/ar/13-observability.md) | [`.mmd`](D-66-overlapping-layered-experiments.mmd) |
| `D-67` | SLO-based alerting | مخطط انسيابي | [الفصل 13 · المراقبة](../docs/ar/13-observability.md) | [`.mmd`](D-67-slo-based-alerting.mmd) |
| `D-68` | Web spam techniques and where each is caught | مخطط انسيابي | [الفصل 14 · السبام والأمان](../docs/ar/14-security-abuse.md) | [`.mmd`](D-68-web-spam-techniques-and-where-each-is-caught.mmd) |
| `D-69` | Link graph anomaly detection | مخطط انسيابي | [الفصل 14 · السبام والأمان](../docs/ar/14-security-abuse.md) | [`.mmd`](D-69-link-graph-anomaly-detection.mmd) |
| `D-70` | Attack surfaces and controls | مخطط انسيابي | [الفصل 14 · السبام والأمان](../docs/ar/14-security-abuse.md) | [`.mmd`](D-70-attack-surfaces-and-controls.mmd) |
| `D-71` | Query data lifecycle and privacy controls | مخطط انسيابي | [الفصل 14 · السبام والأمان](../docs/ar/14-security-abuse.md) | [`.mmd`](D-71-query-data-lifecycle-and-privacy-controls.mmd) |
| `D-72` | CAP positioning by subsystem | مخطط انسيابي | [الفصل 15 · المفاضلات](../docs/ar/15-tradeoffs.md) | [`.mmd`](D-72-cap-positioning-by-subsystem.mmd) |
| `D-73` | Rejected architectures and their failure points | مخطط انسيابي | [الفصل 15 · المفاضلات](../docs/ar/15-tradeoffs.md) | [`.mmd`](D-73-rejected-architectures-and-their-failure-points.mmd) |
| `D-74` | Architecture by corpus size | مخطط انسيابي | [الفصل 15 · المفاضلات](../docs/ar/15-tradeoffs.md) | [`.mmd`](D-74-architecture-by-corpus-size.mmd) |
| `D-75` | A 45-minute plan | خط زمني | [الفصل 16 · المقابلة](../docs/ar/16-interview-guide.md) | [`.mmd`](D-75-a-45-minute-plan.mmd) |
| `D-76` | Whiteboard drawing order | مخطط انسيابي | [الفصل 16 · المقابلة](../docs/ar/16-interview-guide.md) | [`.mmd`](D-76-whiteboard-drawing-order.mmd) |
| `D-77` | Evaluation dimensions | مخطط انسيابي | [الفصل 16 · المقابلة](../docs/ar/16-interview-guide.md) | [`.mmd`](D-77-evaluation-dimensions.mmd) |
| `D-78` | Concept map | مخطط انسيابي | [الفصل 17 · المعجم](../docs/ar/17-glossary.md) | [`.mmd`](D-78-concept-map.mmd) |

</div>

---

<div align="center">

[🏠 Home](../README.md) · [🇸🇦 الرئيسية](../README.ar.md) · [📚 Chapter 01](../docs/en/01-overview.md)

</div>
