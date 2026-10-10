# Automotive Manual RAG: System Diagrams

This document shows the two main flows of the vehicle-manual RAG system:

1. **[Online Query Flow](#1-online-query-flow)**: how a user's question becomes a validated answer with images.
2. **[Offline Ingestion Pipeline](#2-offline-ingestion-pipeline)**: how PDF manuals are parsed, captioned, embedded and indexed in Qdrant.

---

## 1. Online Query Flow

When a user asks a question in the app, the API first resolves the vehicle. It then runs **hybrid retrieval**: a dense search on an LLM-rewritten query and a sparse search on the original query. Both searches are hard-filtered to the user's make, model and year. Questions classified as out of scope skip retrieval entirely. The fused results are de-duplicated and expanded with neighbouring chunks and images, then sent to the generation LLM. Its structured JSON output is validated, and repaired once if needed, before signed image URLs are attached and the response is returned.

| Phase | What happens |
| --- | --- |
| **Understand** | Vehicle validation, query rewriting, intent classification |
| **Retrieve** | Dense + sparse Qdrant search with a hard vehicle filter, weighted fusion |
| **Enrich** | Near-duplicate suppression, neighbour expansion, up to 3 cached images |
| **Generate** | Multimodal prompt → LLM (temp 0.1, 800 tokens) → structured JSON |
| **Validate** | Pydantic + reference allowlists, one repair retry, degraded fallback |

```mermaid
flowchart TD
    A["User asks a question in APP"] --> B["API receives query and vehicle context"]
    B --> C["Resolve and validate vehicle identity"]
    C --> D["Query rewriting: one small LLM call"]
    C --> E["Intent classification: keyword rules + embedding classifier"]
    D --> F["Rewritten query"]
    F --> G["Dense embedding"]
    C --> H["Original query"]
    H --> I["Custom automotive tokenizer"]
    I --> J["Sparse query vector"]
    G --> K["Qdrant dense search"]
    J --> L["Qdrant sparse search"]
    C --> M["Hard vehicle filter: make + model + year"]
    M --> K
    M --> L
    K --> N["Application-layer weighted fusion"]
    L --> N
    E --> O{"Intent?"}
    O -->|Out of scope| P["Skip retrieval; return out-of-scope result"]
    O -->|Answerable| N
    N --> Q["Top-k hits"]
    Q --> R["Near-duplicate suppression"]
    R --> S["Expand neighboring text chunks"]
    S --> T["Select up to 3 unique images"]
    T --> U["Fetch web images concurrently"]
    U --> V["Process-local TTLCache: 256 entries, 1-hour TTL"]
    V --> W["Assemble multimodal prompt"]
    W --> X["Generation LLM: temperature 0.1, max_tokens 800"]
    X --> Y["Structured JSON output"]
    Y --> Z["Pydantic validation + reference allowlists"]
    Z --> AA{"Valid?"}
    AA -->|Yes| AB["Map image IDs to backend-controlled signed URLs"]
    AA -->|No, first failure| AC["One repair retry with validation error"]
    AC --> Z
    AA -->|Retry fails| AD["Degraded response"]
    AB --> AE["Return response to APP"]
    AD --> AE
    P --> AE
```

---

## 2. Offline Ingestion Pipeline

Each manual listed in the registry goes through an eight-stage batch pipeline. The PDF is parsed with its layout preserved, and figures are extracted, filtered and stored in private S3. A VLM captions each figure using its page context. Text is split into semantic chunks that keep lists and tables intact. Text chunks and caption chunks are then embedded as both dense and sparse vectors. The new index is built in a **versioned Qdrant collection**, and the `manuals` alias switches to it only after validation, so online retrieval never sees a half-built index.

| Stage | Step | Output |
| --- | --- | --- |
| 0 | Manual registry | `manuals.yaml` |
| 1 | Layout-aware PDF parsing | `parsed.md`, `blocks.json` |
| 2 | Image and diagram extraction | `figures/*.png`, `figures.json` |
| 3 | Image storage | Private S3: `full/` and `web/` |
| 4 | VLM captioning (GPT-4o) | `captions.json` (SHA-256 cached) |
| 5 | Semantic text chunking | Text chunks with breadcrumbs |
| 6 | Dense + sparse embedding | 1024-d dense vectors, sparse vectors |
| 7 | Indexing and alias switch | Validated Qdrant collection |

```mermaid
flowchart TD
    A["Stage 0: manuals.yaml registry"] --> B["Stage 1: Layout-aware PDF parsing"]
    B --> C["parsed.md + blocks.json"]
    B --> D["Stage 2: Extract raster images and vector diagrams"]
    D --> E["Filter small, extreme, and duplicate candidates"]
    E --> F["figures/*.png + figures.json"]
    F --> G["Stage 3: Store images in private S3"]
    G --> H["full/ original images + web/ optimized images"]
    H --> I["Stage 4: GPT-4o VLM captioning"]
    C --> I
    I --> J["Caption each figure with local page context"]
    J --> K["captions.json; SHA-256 cache"]
    C --> L["Stage 5: Semantic text chunking"]
    L --> M["Breadcrumbs + overlap + intact lists and tables"]
    M --> N["Text chunks"]
    K --> O["Image-caption chunks"]
    N --> P["Stage 6: Dense embeddings"]
    O --> P
    N --> Q["Custom automotive tokenizer"]
    O --> Q
    P --> R["Dense vectors: 1024 dimensions"]
    Q --> S["Sparse vector representations"]
    R --> T["Stage 7: Deterministic point IDs + payloads"]
    S --> T
    T --> U["Qdrant versioned collection"]
    U --> V["Create payload indexes"]
    V --> W["Validate new index"]
    W --> X["Switch manuals alias"]
    X --> Y["Online retrieval uses new index"]
```
