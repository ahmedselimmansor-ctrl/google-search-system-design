<div align="right">

**[🇸🇦 اقرأ بالعربية](../ar/05-content-processing.md)**

</div>

# Chapter 05 · Content Processing

**← [04 · Crawling](04-crawling.md)** · [🏠 Home](../../README.md) · **[06 · Indexing](06-indexing.md) →**

---

## 5.1 From bytes to indexable signal

The crawler hands over compressed HTML. The indexer needs clean tokens, per-document features, and a link graph. Everything in between is content processing — the least glamorous and most quality-determining stage in the pipeline.

Its importance is easy to underrate. **A ranking model can only rank what the processor extracted.** If boilerplate stripping keeps the navigation menu, every page on a site looks identical for its nav terms. If language detection is wrong, the wrong tokenizer runs and the document becomes unfindable. Errors here are silent, systematic, and invisible to A/B tests that only measure the ranking layer.

> **Diagram D-21 — Content processing pipeline**

```mermaid
flowchart TB
    IN["Raw fetched bytes<br/>from content store"]

    IN --> CT["Content-type & charset detection<br/>headers → BOM → heuristics"]
    CT --> DEC["Decode to Unicode<br/>normalize to NFC"]

    DEC --> ROUTE{"Media type?"}
    ROUTE -->|"HTML"| HTML["HTML parser<br/>error-tolerant, spec-compliant"]
    ROUTE -->|"PDF"| PDF["PDF text extraction"]
    ROUTE -->|"Office / plain"| OFF["Format-specific extractors"]
    ROUTE -->|"Image / video"| MED["Metadata + alt text +<br/>surrounding context only"]
    ROUTE -->|"Unsupported"| SKIP["Record type, skip body"]

    HTML --> DOM["DOM tree"]
    DOM --> JS{"Requires JS<br/>rendering?"}
    JS -->|"Yes"| REND["Headless render queue<br/>expensive — budgeted"]
    JS -->|"No"| STRUCT
    REND --> STRUCT["Structural analysis"]

    STRUCT --> BOIL["Boilerplate removal<br/>nav · footer · ads · sidebars"]
    STRUCT --> META["Metadata extraction<br/>title · description · og: · schema.org"]
    STRUCT --> LINKS["Outlink extraction<br/>href + anchor text + rel"]
    STRUCT --> FIELDS["Field segmentation<br/>title · h1-h6 · body · lists · tables"]

    PDF & OFF & MED --> BOIL

    BOIL --> LANG["Language identification<br/>per document and per block"]
    LANG --> NORM["Language-specific normalization"]
    NORM --> TOK["Tokenization<br/>+ stemming / lemmatization"]

    TOK --> QUAL["Quality & safety classifiers<br/>spam · adult · thin content · readability"]
    TOK --> DEDUP["SimHash & canonical clustering"]

    LINKS --> LG[("Link graph store")]
    QUAL --> DOCS[("Processed document store")]
    DEDUP --> DOCS
    META --> DOCS
    FIELDS --> DOCS

    DOCS --> OUT(("→ Index builder<br/>Chapter 06"))
    LG --> OUT

    classDef parse fill:#fde68a,stroke:#b45309,color:#1c1917
    classDef text fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    classDef store fill:#e9d5ff,stroke:#7e22ce,color:#1c1917
    classDef expensive fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class CT,DEC,HTML,PDF,OFF,MED,DOM,STRUCT parse
    class BOIL,LANG,NORM,TOK,QUAL,DEDUP,META,FIELDS,LINKS text
    class LG,DOCS store
    class REND expensive
```

---

## 5.2 Parsing HTML you did not write

Real-world HTML is broken. Unclosed tags, mismatched nesting, invalid character references, and 40 MB of inline JSON are all normal. The parser must be **error-tolerant in exactly the way browsers are** — because if your parser reads a document differently from Chrome, you index something the user will never see.

| Hazard | Handling |
|---|---|
| Malformed / unclosed tags | HTML5-spec-compliant recovery, identical to browser behaviour |
| Wrong or missing charset | Header → BOM → `<meta charset>` → statistical detection, in that order |
| Enormous documents | Hard cap (e.g. 10 MB parsed); truncate, do not fail |
| Deeply nested DOM | Depth cap; malformed nesting is a spam signal |
| Content injected by JavaScript | Route to a rendering queue — but only for a budgeted subset |
| `<noscript>` / hidden text | Extract but flag; hidden text is a classic spam vector |
| Frames and iframes | Follow as separate documents, not as inline content |

### The JavaScript rendering problem

A large share of the modern web renders its main content client-side. Fetching the HTML gives you an empty `<div id="root">`. To index it you must run a headless browser: execute JavaScript, wait for network idle, then serialise the DOM.

This costs **100× to 1000× more CPU than parsing static HTML** and introduces unbounded latency and security exposure. So rendering is *rationed*:

```mermaid
flowchart LR
    DOC["Fetched document"] --> CHK{"Static HTML has<br/>meaningful text?"}
    CHK -->|"Yes"| FAST["✅ Index directly<br/>cost ≈ 1"]
    CHK -->|"No / thin"| VAL{"Worth rendering?<br/>PageRank · demand ·<br/>site history"}
    VAL -->|"No"| THIN["Index thin version<br/>flag as JS-dependent"]
    VAL -->|"Yes"| QUEUE["Render queue<br/>cost ≈ 100–1000"]
    QUEUE --> RENDER["Headless browser<br/>sandboxed, timeout-capped"]
    RENDER --> IDX["Index rendered DOM"]

    classDef cheap fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef exp fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class FAST cheap
    class QUEUE,RENDER exp
```

This is why JavaScript-heavy sites are indexed later and less completely than server-rendered ones. It is not a policy preference; it is a queueing consequence of a 1000× cost ratio.

---

## 5.3 Boilerplate removal — finding the actual content

An average web page is mostly *not* its content. Navigation, headers, footers, sidebars, cookie banners, related-article widgets and advertising can be 80 % of the tokens. Indexing them causes two concrete failures:

1. **Every page on a site looks the same** for the terms in its shared chrome, destroying intra-site discrimination.
2. **Ranking signals are diluted** — term frequencies are dominated by template text rather than subject matter.

> **Diagram D-22 — Main-content extraction**

```mermaid
flowchart TB
    DOM["Parsed DOM tree"]

    DOM --> B1["Signal 1 — Text density<br/>text chars ÷ tag count per block"]
    DOM --> B2["Signal 2 — Link density<br/>anchor text ÷ total text<br/>high ⇒ navigation"]
    DOM --> B3["Signal 3 — Structural role<br/>main · article · nav · footer · aside"]
    DOM --> B4["Signal 4 — Cross-page templates<br/>blocks identical across the site"]
    DOM --> B5["Signal 5 — Visual position<br/>from render, when available"]
    DOM --> B6["Signal 6 — Repetition entropy<br/>low-entropy blocks are chrome"]

    B1 & B2 & B3 & B4 & B5 & B6 --> CLS["Block classifier<br/>per DOM subtree"]

    CLS --> MAIN["🟢 Main content<br/>full weight"]
    CLS --> SUPP["🟡 Supplementary<br/>captions, tables — reduced weight"]
    CLS --> CHROME["🔴 Boilerplate<br/>dropped from body index"]

    MAIN --> FIELDS["Field-weighted representation"]
    SUPP --> FIELDS
    CHROME --> SITE["Retained as site-level template<br/>useful for site structure, not doc content"]

    FIELDS --> W["Field weights at index time<br/>title ≫ h1 > h2-h6 > body > alt/caption"]

    classDef good fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef mid fill:#fef08a,stroke:#a16207,color:#1c1917
    classDef drop fill:#fecaca,stroke:#b91c1c,color:#1c1917
    class MAIN good
    class SUPP mid
    class CHROME drop
```

**Signal 4 is the most powerful and the most often forgotten.** If you have crawled 10,000 pages from one host, the blocks that are byte-identical across all of them are, by definition, template. Cross-page differencing beats any per-page heuristic — but it only works because you crawl whole sites, not isolated pages. It is a good example of a capability that emerges from scale rather than from cleverness.

---

## 5.4 Language identification and normalisation

You cannot tokenize before you know the language, and you cannot detect the language reliably from a domain suffix. Detection runs on the extracted text, per document *and* per block, because multilingual pages are common.

| Method | Strength | Weakness |
|---|---|---|
| Character n-gram profiles | Fast, tiny, works on short text | Confuses closely related languages |
| Unicode script detection | Instant for Arabic, CJK, Cyrillic, Greek | Useless for Latin-script languages |
| Stopword frequency | Cheap and interpretable | Needs reasonably long text |
| Neural classifiers | Best accuracy, handles code-switching | Higher CPU cost per document |
| HTML `lang` attribute | Free | Frequently wrong or copy-pasted |

The practical stack: **script detection first** (it resolves Arabic, Chinese, Japanese, Korean, Cyrillic, Greek, Hebrew, Thai in one pass at near-zero cost), then n-grams for Latin-script disambiguation, then a neural model only for short or ambiguous text.

---

## 5.5 Tokenization is language-specific — a worked comparison

This is where a genuinely global search engine gets hard, and where a bilingual document like this one can be most useful.

> **Diagram D-23 — Language-specific tokenization paths**

```mermaid
flowchart TB
    TEXT["Clean extracted text"] --> SCRIPT{"Detected script<br/>and language"}

    SCRIPT -->|"Latin: EN, FR, DE, ES"| LAT["Whitespace + punctuation split"]
    LAT --> LAT2["Lowercase · strip diacritics<br/>Porter/Snowball stemming<br/>compound splitting for DE"]

    SCRIPT -->|"Arabic"| AR["Arabic pipeline"]
    AR --> AR1["Strip tatweel ـ and diacritics ً ٌ ٍ َ ُ ِ ّ ْ"]
    AR --> AR2["Normalize alef forms<br/>أ إ آ ٱ → ا"]
    AR --> AR3["Normalize teh marbuta ة → ه<br/>and alef maqsura ى → ي"]
    AR --> AR4["Normalize Arabic-Indic digits<br/>٠١٢٣ → 0123"]
    AR --> AR5["Clitic segmentation<br/>و ف ب ك ل ال prefixes<br/>ه ها هم ك ي suffixes"]
    AR --> AR6["Light stemming or<br/>root-pattern morphological analysis"]

    SCRIPT -->|"CJK: ZH, JA"| CJK["No whitespace boundaries"]
    CJK --> CJK1["Dictionary + statistical<br/>word segmentation"]
    CJK --> CJK2["Also index character bigrams<br/>as a recall safety net"]

    SCRIPT -->|"Korean"| KO["Agglutinative:<br/>morphological analysis required"]
    SCRIPT -->|"Thai"| TH["No spaces:<br/>dictionary segmentation"]

    LAT2 & AR6 & CJK2 & KO & TH --> COMMON["Common post-processing"]
    COMMON --> C1["Position assignment<br/>for phrase queries"]
    COMMON --> C2["Field tagging<br/>title · heading · body"]
    COMMON --> C3["Entity and number recognition"]
    COMMON --> OUT(("Token stream<br/>→ index builder"))

    classDef ar fill:#bbf7d0,stroke:#15803d,color:#1c1917
    classDef cjk fill:#fed7aa,stroke:#c2410c,color:#1c1917
    classDef lat fill:#bfdbfe,stroke:#1d4ed8,color:#1c1917
    class AR,AR1,AR2,AR3,AR4,AR5,AR6 ar
    class CJK,CJK1,CJK2,KO,TH cjk
    class LAT,LAT2 lat
```

### Why Arabic is genuinely harder than English

| Property | Consequence for retrieval |
|---|---|
| **Optional diacritics** | `كتب` may be *kataba* (he wrote), *kutub* (books), or *kutiba* (it was written). The written form is identical; the index must either normalise away diacritics entirely or handle several readings. |
| **Root-and-pattern morphology** | One triliteral root ك-ت-ب generates كتاب، كاتب، مكتبة، مكتوب، كتابة. Suffix-stripping stemmers (Porter-style) fail; you need either light stemming or a real morphological analyser. |
| **Agglutinative clitics** | `وبالمكتبة` = و + ب + ال + مكتبة ("and at the library") is one orthographic token containing four morphemes. Without clitic segmentation, the query مكتبة will not match it. |
| **Orthographic variation** | أحمد / احمد, مكتبة / مكتبه — writers are inconsistent. Normalisation must collapse these, at the cost of some precision. |
| **Bidirectional text** | Mixed Arabic-Latin content requires correct handling of Unicode bidi control characters, which are also a cloaking vector. |
| **Arabic-Indic digits** | ٢٠٢٦ and 2026 mean the same thing and must be unified. |

**The design tension:** aggressive normalisation raises *recall* (many spellings match one index term) and lowers *precision* (distinct words collapse together). Production systems usually normalise aggressively at index time and then recover precision at ranking time, by keeping the original surface form as a feature and boosting exact matches. This is a general principle worth remembering: **be permissive during retrieval, be strict during ranking.**

---

## 5.6 The link graph

Outlinks are extracted during parsing and accumulated into a global graph — arguably the single most valuable derived asset in the whole system, since it powers PageRank, anchor-text indexing, spam detection and crawl prioritisation simultaneously.

> **Diagram D-24 — Link graph and derived document data model**

```mermaid
erDiagram
    DOCUMENT ||--o{ OUTLINK : "contains"
    DOCUMENT ||--o{ INLINK : "receives"
    DOCUMENT ||--|| CONTENT : "has"
    DOCUMENT ||--o{ ANCHOR_TEXT : "described by"
    DOCUMENT }o--|| HOST : "belongs to"
    DOCUMENT }o--o| DUP_CLUSTER : "member of"
    HOST }o--|| REG_DOMAIN : "part of"
    HOST ||--o{ ROBOTS_RULE : "declares"
    DUP_CLUSTER ||--|| DOCUMENT : "canonical is"

    DOCUMENT {
        bytes  docid_hash    PK
        string url_canonical
        int64  fetch_time
        int32  http_status
        int64  simhash
        float  pagerank
        float  spam_score
        float  quality_score
        string language
        int32  content_length
        bool   is_canonical
    }

    CONTENT {
        bytes  docid_hash    FK
        string title
        string main_text
        string meta_description
        blob   structured_data
        blob   field_offsets
    }

    OUTLINK {
        bytes  src_docid     FK
        bytes  dst_docid     FK
        string anchor_text
        bool   is_nofollow
        int32  position_in_doc
        bool   is_internal
    }

    INLINK {
        bytes  dst_docid     FK
        bytes  src_docid     FK
        float  src_pagerank
        string anchor_text
    }

    ANCHOR_TEXT {
        bytes  docid         FK
        string text
        int32  frequency
        int32  distinct_domains
    }

    HOST {
        string hostname      PK
        float  host_pagerank
        float  host_spam_score
        int32  crawl_delay_ms
        int64  known_url_count
    }

    REG_DOMAIN {
        string domain        PK
        float  trust_score
        int32  host_count
    }

    DUP_CLUSTER {
        int64  cluster_id    PK
        bytes  canonical_doc FK
        int32  member_count
    }

    ROBOTS_RULE {
        string hostname      FK
        string user_agent
        string path_pattern
        bool   allow
    }
```

### Anchor text — describing a page in other people's words

The text inside a link (`<a href="...">click here for the tutorial</a>`) is a description of the *target* page written by a third party. This is extraordinarily valuable:

- It describes pages in the **vocabulary users actually search with**, not the vocabulary the page author chose.
- It works for documents whose own text is unindexable — images, videos, PDFs, JavaScript apps.
- It aggregates many independent opinions about what a page is about.

It is also the **single most abused signal on the web**, because anyone can write a link to your page saying anything. Defences: cap contribution per source domain, weight by the source's own trust score, discount `rel="nofollow"` / `rel="ugc"` / `rel="sponsored"`, and require diversity across distinct registrable domains before a phrase counts. See [Chapter 14](14-security-abuse.md).

---

## 5.7 Quality classification

Not every fetched document deserves an index entry. Cheap classifiers run at processing time and attach scores used later for tiering, ranking and filtering.

| Classifier | Detects | Used for |
|---|---|---|
| **Spam** | Keyword stuffing, hidden text, cloaking, link schemes | Demotion or exclusion |
| **Thin content** | Auto-generated, scraped, near-empty pages | Tier assignment, demotion |
| **Adult / sensitive** | NSFW material | SafeSearch filtering |
| **Readability** | Text complexity and coherence | Snippet quality, tiering |
| **Machine translation** | Low-quality bulk-translated pages | Demotion |
| **Page structure quality** | Ad-to-content ratio, layout stability | Page-experience signals |
| **Topical classification** | Subject area | Vertical routing, freshness policy |

**Order matters for cost.** These run *before* index construction so that documents scoring badly enough are never indexed at all — saving index bytes, which [Chapter 03](03-capacity-estimation.md) showed is the dominant cost in the system.

---

## 5.8 The processed-document contract

> **Diagram D-25 — Document state through processing**

```mermaid
stateDiagram-v2
    [*] --> RawBytes

    RawBytes --> Decoded: charset resolved
    RawBytes --> Unprocessable: binary / unsupported / corrupt

    Decoded --> Parsed: DOM built
    Decoded --> RenderQueued: JS-dependent and worth rendering
    RenderQueued --> Parsed: headless render complete
    RenderQueued --> Parsed: render timeout — use static DOM

    Parsed --> Extracted: boilerplate removed, fields segmented
    Extracted --> Tokenized: language detected, tokens emitted

    Tokenized --> Classified: quality and safety scores attached
    Classified --> Fingerprinted: SimHash computed

    Fingerprinted --> CanonicalDoc: chosen as cluster representative
    Fingerprinted --> DuplicateDoc: another doc is canonical

    CanonicalDoc --> ReadyToIndex
    DuplicateDoc --> SignalsOnly: links and anchors merged into canonical

    ReadyToIndex --> [*]: → Chapter 06
    SignalsOnly --> [*]
    Unprocessable --> [*]

    note right of DuplicateDoc
        A duplicate is not discarded.
        Its inbound links and anchor text
        are merged into the canonical
        document — otherwise mirrors
        would silently destroy PageRank.
    end note
```

The output contract consumed by the index builder:

| Field | Purpose |
|---|---|
| `docid` | Stable 64-bit id, assigned by hash of canonical URL |
| `tokens[]` | Token, field tag, position — the raw material of the inverted index |
| `title`, `meta` | Displayed on the SERP, weighted heavily in ranking |
| `main_text` | Boilerplate-stripped body, kept for snippet generation |
| `language`, `locale` | Routing and matching |
| `outlinks[]`, `anchors[]` | Link graph contribution |
| `simhash`, `cluster_id` | Deduplication |
| `quality_scores{}` | Spam, thin, adult, readability |
| `structured_data` | schema.org / JSON-LD for rich results |
| `last_modified`, `change_rate` | Freshness scheduling |

---

<div align="center">

**← [04 · Crawling](04-crawling.md)** · [🏠 Home](../../README.md) · **[06 · Indexing](06-indexing.md) →** · [🇸🇦 العربية](../ar/05-content-processing.md)

</div>
