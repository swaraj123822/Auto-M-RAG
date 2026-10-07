# Auto-M-RAG: Multimodal Automotive Manual Assistant API

## Overview

Auto-M-RAG is a production-grade Multimodal Retrieval-Augmented Generation (RAG) backend designed to serve a Kotlin-based mobile application. The system processes dense, 60-90 page technical automotive user manuals (covering 5 different vehicle models) to answer user queries with high-precision text instructions and relevant contextual diagrams.

This project goes beyond standard text RAG by successfully indexing, retrieving, and reasoning over complex spatial layouts, tables, and exploded mechanical diagrams.

## Execution Pipeline (How We Built It)

Our development lifecycle was broken down into four distinct phases:

### Phase 1: Data Ingestion & Multimodal Parsing

1. **Sourced Manuals:** Collected 5 diverse car manuals (~450 pages total), featuring complex layouts, tables, and mechanical diagrams.
2. **Layout-Aware Parsing:** Processed PDFs to extract text while maintaining hierarchical document structure.
3. **Image Extraction & Storage:** Isolated all diagrams and charts, storing the raw image files in a cloud bucket with unique URIs.
4. **Vision-Language Model (VLM) Captioning:** Ran a VLM over every extracted image to generate dense textual descriptions (capturing part numbers, labels, and spatial relationships).

### Phase 2: Embedding & Vector Database (Qdrant)

1. **Chunking Strategy:** Implemented semantic chunking with a 15% overlap to ensure multi-step procedures (e.g., oil change steps) weren't split abruptly.
2. **Unified Embedding:** Embedded both the text chunks and the VLM-generated image captions using a dense embedding model.
3. **Qdrant Indexing:** Upserted vectors into Qdrant. Crucially, we attached rich payload metadata to every vector: `make`, `model`, `year`, `chunk_type` (text vs. image), and `image_uri`.

### Phase 3: Retrieval & Generation (Serving the Kotlin App)

1. **Query Processing & Routing:** Rewrote user queries for better semantic matching and extracted metadata (e.g., routing "2019 Civic oil change" to filter only the Honda Civic manual).
2. **Hybrid Search:** Polled Qdrant using both dense semantic search (for conceptual queries) and sparse search (BM25 for exact part numbers).
3. **Multimodal Synthesis:** Passed the retrieved text and the raw images (fetched via Qdrant URIs) to a multimodal generation LLM (GPT-4o).
4. **Structured Output:** Formatted the LLM response as a strict JSON schema containing the answer text and an array of image URIs, designed specifically for seamless consumption by the Kotlin mobile app front-end.

## Performance & Evaluation Results

Building the RAG was only half the project; proving it works required a rigorous evaluation pipeline. In automotive maintenance, hallucinations can lead to vehicle damage or safety hazards. We tuned our Hybrid Search weights (α = 0.7 dense, 0.3 sparse) and evaluated against a golden dataset of **150 queries — 135 answerable and 15 deliberately unanswerable**, the latter to verify the system abstains rather than inventing an answer.

**System Scale & Latency:**
* **Knowledge Base:** 5 complete vehicle manuals (~450 pages).
* **Vector Count:** ~1,400 text chunks and ~820 VLM-captioned image embeddings (~2,220 total).
* **End-to-End Latency:** 4.6 s p50, 5.2 s mean, 8.4 s p95 (accounts for Qdrant retrieval, fetching raw images, and the longer generation time of multimodal LLMs — generation alone is 78% of it).

**Retrieval Metrics** (135 answerable queries, 95% Wilson intervals):
* **Recall@5: 85.2%** (115/135), CI [78.2%, 90.2%] — the correct manual page or diagram was in the top 5 for 115 of 135 queries, overcoming messy PDF layouts.
* **Mean Reciprocal Rank (MRR): 0.69** — 58% of queries returned the exactly correct chunk in first place, with a steeply decaying tail.

**Generation Metrics (LLM-as-a-Judge using GPT-4, scored per claim):**
* **Faithfulness: 91.4%** (523/572 atomic claims), CI [88.9%, 93.5%] — hallucinated torque specs and part numbers near-eliminated. Only 8 claims (1.4%) were actively contradicted by the source, and all 8 trace to illegible original diagrams or a dropped footnote marker rather than to retrieval or prompting.
* **Answer Relevance: 4.42 / 5 (88.4%)** — the system directly addressed the user's query without dumping unnecessary manual pages.
* **Abstention accuracy: 86.7%** (13/15 unanswerable queries correctly refused), with a 4.4% false-abstention rate on answerable ones.

> **On reading these numbers.** 135 queries puts a ±3 pp standard error on the retrieval figures — enough to demonstrate a large change, not enough to resolve a small one. Several steps in the build history fall inside that floor and are labelled as such in [ARCHITECTURE.md §13.5](ARCHITECTURE.md#135-metric-progression) rather than reported as wins. Denominators, intervals and methodology are all in [§13](ARCHITECTURE.md#13-evaluation-harness).

## Key Engineering Decisions & Justifications

### 1. Parsing: Layout-Aware Parsing over PyPDF2
* **Decision:** We completely bypassed traditional parsers like PyPDF2 in favor of layout-aware parsing (e.g., LlamaParse/Unstructured).
* **Why:** Automotive manuals are visually dense. A standard parser reads left-to-right, destroying tables and completely ignoring images. By using a layout-aware parser, we preserved the structural integrity of maintenance schedules and extracted mechanical diagrams accurately.

### 2. Multimodal Embedding: "Look Twice" VLM Pattern over CLIP
* **Decision:** Instead of embedding images directly into a vector space using CLIP, we used a VLM (Vision-Language Model) to generate highly detailed text captions of the images, and then embedded those captions.
* **Why:** CLIP is excellent for general images ("a dog on a beach") but fails at dense technical diagrams ("Diagram showing oil drain plug with 14mm bolt specification"). By having a VLM describe the image first, we converted the visual technical data into highly searchable semantic text.

### 3. Retrieval: Hybrid Search in Qdrant
* **Decision:** We utilized Qdrant's Hybrid Search capabilities, combining Dense Vectors with Sparse Vectors (BM25), heavily augmented by Metadata Filtering.
* **Why:** 
  * *Dense Vectors* handle semantic intent (e.g., "my steering wheel is shaking"). 
  * *Sparse Vectors* are strictly necessary for exact keyword matching. If a user asks for "Fuse F42", dense vectors often blur this alphanumerical data; BM25 catches it.
  * *Metadata Filtering* acts as a hard boundary. Without filtering by `car_model`, the system risks retrieving a Toyota oil change procedure for a Ford query, which is a critical failure.

### 4. API Design: Structured JSON Output for Kotlin App
* **Decision:** The final LLM generation is enforced via function calling/structured outputs to return a strict JSON payload.
* **Why:** The Kotlin mobile app cannot parse raw markdown with embedded images reliably. The API returns:
  ```json
  {
    "status": "success",
    "answer": "To change the oil, first locate the 14mm drain plug under the oil pan...",
    "images": ["https://storage.bucket.com/manuals/honda/civic/fig12_drain_plug.png"],
    "source_pages": [45, 46]
  }
  ```
  This allows the Kotlin UI to natively render the text in a `TextView` and load the images directly into an `ImageView` via a library like Glide or Coil.