# PDF Extraction Solution

**Information Extraction from Non-Standardized PDF Datasheets**
*Technical Solution, Architecture & Unit Economics*

A scalable, VLM-first pipeline for 1.5M+ PDFs refreshed bi-weekly, extendable from 34 to 600 categories and 10 to 600 attributes.

---

# 1. Solution Overview

## 1.1 Core Architectural Bets

- **Parse once, extract many.** The expensive VLM parse runs exactly once per content hash. Re-extracting a new attribute across all 1.5M PDFs costs ~$30 against cached Markdown — not $3,000 for a full re-parse.

- **VLM as the default extractor.** Gemini 2.5 Flash-Lite ($0.10/$0.40 per MTok input/output at standard rate; 50% off on Flex/Batch) replaces the PyMuPDF4LLM + LayoutParser + LayoutReader + PaddleOCR + Agentic stack. One model handles scanned pages, multi-column layouts, tables, and images.

- **Two processing buckets.** Source bucket alone determines processing tier — no classification logic, no urgency tagger. PDFs in `fast/` use real-time Gemini on Lambda; PDFs in `slow/` use Fargate with 50%-discounted Gemini Flex/Batch.

- **Event-driven, serverless.** S3 event → EventBridge → SQS → compute. Fast-bucket PDFs run on Lambda (sub-15-min, synchronous Gemini). Slow-bucket PDFs run on Fargate (no timeout ceiling, 50% cheaper Gemini Flex/Batch). SQS absorbs bursts, deduplicates by content hash, and retries failures.

- **Diff-based incremental extraction.** For the ~20% of PDFs that change each cycle, a section-level diff is computed and only changed sections are sent to the LLM. Unchanged sections retain previously extracted values — cutting token cost ~70–85% on typical churn PDFs.

- **Bigger LLM for Inferred and Empty fields.** Fields tagged Inferred or Empty are re-run through Gemini 2.5 Flash ($0.15/$1.25 per MTok input/output) with a targeted prompt to resolve uncertain inferences and recover genuinely absent fields.

- **One PDF ≠ one product.** The pipeline emits 1..N product SKU records per PDF (ordering matrices expand into many orderable SKUs).

---

# 2. Problem Framing & Key Assumptions

## 2.1 What Makes This Hard

- **Non-standardized layouts.** ~2,000 manufacturers means ~2,000 template families. Attributes appear in title blocks, ordering tables, footnotes, image callouts, photometric charts, and free-form prose.
- **Multi-SKU PDFs.** A single datasheet can describe a product family whose ordering matrix expands into 10–100 SKUs. Two SKUs sharing 8 of 10 attributes are variants, not duplicates.
- **Scanned and hybrid PDFs.** Deterministic parsers (PyMuPDF) silently drop to zero text on these.
- **Attribute scope expansion.** Going from 10 to 600 attributes changes the economics entirely. Adding an attribute must be a prompt change, not a pipeline change.
- **Bi-weekly churn.** ~300K PDFs reprocess every two weeks. Early-exit on unchanged content-hash; diff-based partial re-extraction for changed PDFs.

---

# 3. End-to-End Architecture

## 3.1 Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                          PARSPEC PDF EXTRACTION PIPELINE                             ║
╚══════════════════════════════════════════════════════════════════════════════════════╝

 ┌─────────────────────────────────────────────────────────────────────────────────┐
 │  PLANE 1 · INGESTION                                                            │
 │                                                                                 │
 │   PDF / DOCX / HTML                                                             │
 │        │                                                                        │
 │        ▼                                                                        │
 │  ┌───────────────────────────────────────────────────────┐                      │
 │  │  S3  parspec-raw/                                     │                      │
 │  │  ├── fast/<yyyy-mm-dd>/<sha256>.pdf   (~3% of vol)    │                      │
 │  │  └── slow/<yyyy-mm-dd>/<sha256>.pdf   (~97% of vol)   │                      │
 │  └───────────────────────────────────────────────────────┘                      │
 │        │  S3 ObjectCreated event                                                 │
 │        ▼                                                                        │
 │  ┌─────────────┐      ┌──────────────────────────────────────────────────────┐  │
 │  │ EventBridge │─────▶│  Ingestion Lambda  (~100–500 ms)                     │  │
 │  └─────────────┘      │                                                      │  │
 │                        │  1. Normalize to PDF (PyMuPDF / headless Chromium)  │  │
 │                        │  2. Compute SHA-256                                  │  │
 │                        │  3. Query DynamoDB pdf_registry                      │  │
 │                        │     └─ Hash exists? → early-exit, link URL, return   │  │
 │                        │  4. Route by bucket prefix:                          │  │
 │                        │     ├─ fast/ → parspec-fast-queue (FIFO)             │  │
 │                        │     └─ slow/ → parspec-slow-queue (FIFO)             │  │
 │                        │  5. Write status=PENDING to DynamoDB                 │  │
 │                        └──────────────────────────────────────────────────────┘  │
 │                                  │                      │                        │
 │                         fast queue                 slow queue                    │
 └─────────────────────────────────────────────────────────────────────────────────┘
                                    │                      │
 ┌──────────────────────────────────┼──────────────────────┼────────────────────────┐
 │  PLANE 2 · EXTRACTION            │                      │                        │
 │                                  ▼                      ▼                        │
 │                    ┌─────────────────────┐  ┌───────────────────────────────┐    │
 │                    │  FAST PATH          │  │  SLOW PATH                    │    │
 │                    │  Lambda             │  │  Fargate Task (ephemeral ECS) │    │
 │                    │  SLA: ≤ 5 sec       │  │  SLA: 1 min – 24 h            │    │
 │                    │                     │  │                               │    │
 │                    │  Gemini 2.5         │  │  Gemini 2.5 Flash-Lite        │    │
 │                    │  Flash-Lite         │  │  Flex / Batch  (50% off)      │    │
 │                    │  Standard/Priority  │  │                               │    │
 │                    └──────────┬──────────┘  └──────────────┬────────────────┘    │
 │                               │                             │                    │
 │                               └──────────┬──────────────────┘                   │
 │                                          ▼                                       │
 │                          ┌───────────────────────────────┐                       │
 │                          │  PARSER STAGE  (parse-once)   │                       │
 │                          │  Gemini Document Processor     │                       │
 │                          │  · Scanned + native PDFs      │                       │
 │                          │  · Multi-column / tables      │                       │
 │                          │  · Photometric charts         │                       │
 │                          │  · Compliance symbols         │                       │
 │                          │                               │                       │
 │                          │  Output → S3 parspec-canonical│                       │
 │                          │  ├── <sha256>/document.md     │                       │
 │                          │  ├── <sha256>/images/*.webp   │                       │
 │                          │  ├── <sha256>/raw.json        │                       │
 │                          │  └── <sha256>/schema-ver.txt  │                       │
 │                          └───────────────┬───────────────┘                       │
 │                                          │                                       │
 │                          ┌───────────────▼───────────────┐                       │
 │                          │  DIFF CHECK  (changed PDFs)   │                       │
 │                          │                               │                       │
 │                          │  changed_fraction < 30%?      │                       │
 │                          │  ├─ YES → Diff-Extract        │                       │
 │                          │  │        pass changed secs   │                       │
 │                          │  │        + existing SKU JSON │                       │
 │                          │  │        LLM returns delta   │                       │
 │                          │  │        merge into record   │                       │
 │                          │  │        ~70-85% cost saving │                       │
 │                          │  └─ NO  → Full Extract        │                       │
 │                          └───────────────┬───────────────┘                       │
 │                                          │                                       │
 │                          ┌───────────────▼───────────────┐                       │
 │                          │  EXTRACTOR STAGE              │                       │
 │                          │  Gemini 2.5 Flash-Lite        │                       │
 │                          │  Pydantic schema · temp=0     │                       │
 │                          │                               │                       │
 │                          │  Input: Markdown + images     │                       │
 │                          │         + active schema       │                       │
 │                          │  Output: 1..N SKU records     │                       │
 │                          │  + product image classified   │                       │
 │                          │                               │                       │
 │                          │  Each field tagged:           │                       │
 │                          │  ● Present  — literal value   │                       │
 │                          │  ● Inferred — implied value   │                       │
 │                          │  ● Empty    — absent field    │                       │
 │                          └───────────────┬───────────────┘                       │
 │                                          │                                       │
 │                          ┌───────────────┴───────────────┐                       │
 │                          │                               │                       │
 │                          ▼                               ▼                       │
 │          ┌───────────────────────────┐   ┌───────────────────────────┐           │
 │          │  POST-PROCESS &           │   │  IMAGE EMBEDDINGS         │           │
 │          │  VALIDATION               │   │  Gemini Embedding 2       │           │
 │          │  Unit normalization       │   │  $0.00012 / image         │           │
 │          │  Range / enum validation  │   │  dense_vector → S3 +      │           │
 │          │  Outlier flagging         │   │  image_embedding in ES    │           │
 │          └─────────────┬─────────────┘   └─────────────┬─────────────┘           │
 │                        │                               │                         │
 │                        ▼                               │                         │
 │          ┌───────────────────────────┐                 │                         │
 │          │  BIGGER LLM FALLBACK      │                 │                         │
 │          │  Gemini 2.5 Flash         │                 │                         │
 │          │  $0.15 / $1.25 per MTok   │                 │                         │
 │          │                           │                 │                         │
 │          │  Triggered for Inferred + │                 │                         │
 │          │  Empty fields (~10% PDFs) │                 │                         │
 │          │                           │                 │                         │
 │          │  After pass: all fields   │                 │                         │
 │          │  Present / Inferred /     │                 │                         │
 │          │  Empty. Unresolved→Empty  │                 │                         │
 │          └─────────────┬─────────────┘                 │                         │
 │                        │                               │                         │
 │                        └───────────────┬───────────────┘                         │
 │                                        │                                         │
 └────────────────────────────────────────┼─────────────────────────────────────────┘
                                         │
                                         ▼
                              ┌─────────────────────────┐
                              │  DynamoDB               │
                              │  status=COMPLETE        │
                              │  + pdf_registry update  │
                              └────────────┬────────────┘
                                           │
 ┌─────────────────────────────────────────┼─────────────────────────────────────────┐
 │  PLANE 3 · SERVING                      │                                         │
 │                                         ▼                                         │
 │                           ┌─────────────────────────┐                             │
 │                           │  Elasticsearch Index    │                             │
 │                           │  products_v<schema>     │                             │
 │                           │  1 doc = 1 SKU          │                             │
 │                           │  keyword + range fields │                             │
 │                           │  + image_embedding vec  │                             │
 │                           └─────────────────────────┘                             │
 │                                         ▲                                         │
 │                                         │  structured filters                     │
 │                           ┌─────────────────────────┐                             │
 │                           │  Query Rewriter          │                             │
 │                           │  Gemini 2.5 Flash-Lite   │                             │
 │                           │  Redis-cached            │                             │
 │                           │  NL → structured filters │                             │
 │                           └─────────────────────────┘                             │
 │                                         ▲                                         │
 │                                         │                                         │
 │                           ┌─────────────────────────┐                             │
 │                           │  API Gateway             │                             │
 │                           │  FastAPI / Fargate       │                             │
 │                           └─────────────────────────┘                             │
 │                                         ▲                                         │
 │                                         │  search query                           │
 │                                   [ CLIENT ]                                      │
 │                                                                                   │
 │                           Clicks → DynamoDB preference memory                     │
 └───────────────────────────────────────────────────────────────────────────────────┘

 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │  SHARED INFRASTRUCTURE                                                            │
 │                                                                                   │
 │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                │
 │  │  SQS FIFO        │  │  DynamoDB        │  │  S3 Buckets      │                │
 │  │  parspec-fast-q  │  │  pdf_registry    │  │  parspec-raw/    │                │
 │  │  parspec-slow-q  │  │  preferences     │  │  parspec-canon/  │                │
 │  │  dedup SHA-256   │  │  status tracking │  │  parspec-derived/│                │
 │  │  14-day backlog  │  └──────────────────┘  └──────────────────┘                │
 │  │  Flex→Batch auto │                                                             │
 │  │  failover        │  ┌──────────────────┐  ┌──────────────────┐                │
 │  └──────────────────┘  │  Redis           │  │  Observability   │                │
 │                        │  Elasticache     │  │  OpenTelemetry   │                │
 │                        │  query cache     │  │  Datadog APM     │                │
 │                        └──────────────────┘  │  Langfuse (LLM)  │                │
 │                                              └──────────────────┘                │
 └───────────────────────────────────────────────────────────────────────────────────┘
```

## 3.2 Component Overview

The pipeline splits into three independently scalable planes: **Ingestion**, **Extraction**, and **Serving**.

| **Plane** | **Components (in order)** |
|---|---|
| **Ingestion** | S3 PUT → S3 Event → EventBridge → SQS FIFO (dedup on SHA-256) → Ingestion Lambda (~50–500 ms): normalize, hash, early-exit if hash seen, route to appropriate queue by source bucket |
| **Extraction (Fast)** | `parspec-fast-queue` → Lambda → Gemini Standard/Priority → canonical artifact → diff/full extract (incl. image classification) → postprocess + image embeddings (parallel) → index |
| **Extraction (Slow)** | `parspec-slow-queue` → Fargate task → Gemini Flex/Batch (50% off) → canonical artifact → diff/full extract (incl. image classification) → postprocess + image embeddings (parallel) → index |
| **Serving** | Query → API Gateway → Query Service → Query Rewriter (small LLM, Redis-cached) → ES search → top-20 → feedback loop → DynamoDB preference memory |

## 3.3 Data Flow — A PDF's Life

**Shared — all PDFs**

- **T+0:** PDF lands in either `s3://parspec-raw/fast/<id>.pdf` or `s3://parspec-raw/slow/<id>.pdf`. S3 emits ObjectCreated to EventBridge.
- **T+2s:** Ingestion Lambda fires. Computes SHA-256, queries DynamoDB `pdf_registry`. Hash exists → early-exit (link new URL to existing record). New or changed → normalize, route by source bucket to the appropriate SQS queue, write `status=PENDING` to DynamoDB.

**Changed PDF — diff-based extraction**

1. Fetch old canonical Markdown from S3 and parse new PDF to Markdown.
2. Compute section-level diff (git-diff style, splitting on headings and table rows).
3. If diff < 30% of doc → **partial re-extraction**: pass only changed sections + existing SKU JSON to the LLM with a targeted prompt. LLM returns a delta JSON containing only modified fields. Pipeline merges delta into existing SKU records; unchanged fields retain their values and provenance.
4. If diff ≥ 30% → **full re-extraction**: treat as new PDF.
5. Typical churn PDFs change 1–5 fields; diff payloads run ~5–15% of full-document token cost.

Diff-based extraction applies to both Fast and Slow paths. Any changed PDF — regardless of bucket — runs the diff check before extraction.

**New PDF — full extraction**

- Canonical Markdown + page images written permanently to `s3://parspec-canonical/<hash>/`.
- Extractor (Pydantic-validated SKU JSON, temperature=0; product image classified within the same call) runs first. From there, two steps proceed in parallel: (1) Postprocess (unit normalization, range validation, Present/Inferred/Empty tagging) → Bigger LLM for Inferred and Empty derived values; (2) Image Embeddings (Gemini Embedding 2 — $0.00012/image). Both branches converge before indexing into Elasticsearch.
- Fast path: Lambda calls Gemini Standard/Priority (~3–5 sec). Slow path: Fargate calls Gemini Flex/Batch (50% off, 1 min–24 h). Both converge at the same canonical artifact and extraction stages.

**Query time**

Query → Query Rewriter (small LLM) decomposes into structured filters → ES search returns top-20 → clicks captured in DynamoDB preference memory.

## 3.4 Design Decisions

### 3.4.1 Parser Choice

| **Capability** | **Tesseract OCR** | **PaddleOCR (+ layout)** | **PyMuPDF / PyMuPDF4LLM** | **Gemini Document Processor** |
|---|---|---|---|---|
| Native PDF text layer | N/A | N/A | Excellent, lossless | Excellent |
| Scanned/image-only PDFs | Yes | Yes — strong | Fails silently | Yes, unified |
| Multi-column layout | Poor | Good | Good | Excellent |
| Complex tables | Very poor | Good | Fair | Excellent |
| Image + caption alignment | No | Separate step | No semantic link | Native |
| Photometric charts | No | No | No | Yes |
| Compliance symbols | Poor | Fair | Dropped | Excellent |
| Output for LLM | Broken reflow | Needs post-assembly | Markdown | Structured Markdown + JSON |
| Hallucination risk | None | None | None | Mitigated |
| Speed | 5–15 sec | 2–5 sec | < 100 ms | 2–8 sec / 2–8 min Flex |
| Operational complexity | Low | High — GPU fleet | Very low | Low — managed API |
| Self-improving | No | Requires retraining | No | Yes |

**Gemini Document Processor** is the primary parser — the only option that handles every page type in one call, produces LLM-native output, carries minimal operational overhead, and improves for free with each Google release. Parse cost is amortized over the lifetime of the cached artifact.

#### VLM Approach — Current Validity & Migration Trigger

The VLM-first approach is viable today under two conditions that currently hold:

1. **Low page counts per PDF.** Lighting datasheets are typically 1–8 pages. Gemini 2.5 Flash-Lite token costs remain bounded and predictable at this scale.
2. **Stable, cheap model availability.** Gemini 2.5 Flash-Lite at $0.10/$0.40 per MTok (50% off on Flex/Batch) keeps the blended per-PDF cost well under the $0.002 target.

**If either condition breaks — pages grow significantly (e.g., 50+ page catalogs enter the corpus) or Gemini 2.5 Flash-Lite is deprecated/repriced — the pipeline must migrate to a hybrid deterministic + agent stack.**

#### Hybrid Architecture

The hybrid pipeline replaces the single VLM call with a three-stage deterministic pre-processing layer that feeds a targeted LangChain agent. The agent calls VLM tools only on layout-detected regions of interest (charts, tables) rather than full pages — decoupling token cost from page count.

![Hybrid Pipeline Architecture](hybrid_architecture.png)

**Stage breakdown:**

| **Stage** | **Component** | **Output** |
|---|---|---|
| Text extraction | PaddleOCR | Text strings, bounding boxes, confidence scores |
| Region detection | PaddleOCR LayoutDetect | Tables, charts, text block boundaries |
| Reading order | LayoutReader | Correct traversal order across multi-column and table regions |
| Attribute extraction | LangChain Agent + VLM tools | Structured Markdown — same format as VLM path |

**LangChain Agent system prompt receives:**
- All OCR text in LayoutReader-determined reading order
- Layout region IDs and types (table, chart, text block)
- Tool descriptions for `AnalyzeChart` and `AnalyzeTable`

**Agent tools:**

| **Tool** | **Input** | **Returns** |
|---|---|---|
| `AnalyzeChart` | Cropped chart image → VLM | Chart type, axes, data points, trends |
| `AnalyzeTable` | Cropped table image → VLM | Headers, rows, values, notes |

The agent invokes these tools only on regions flagged by LayoutDetect — a 50-page catalog may contain 3 charts and 5 tables, incurring VLM cost on 8 crops rather than 50 full pages.

The canonical Markdown artifact format is preserved — downstream extractor, diff, and serving stages require no changes. The tradeoff is operational complexity: a GPU Fargate fleet for PaddleOCR, three additional model dependencies, and assembly logic to stitch ordered OCR text with agent tool outputs.

**Migration signals to watch:**
- Average pages-per-PDF exceeds 5
- Gemini 2.5 Flash-Lite deprecation notice issued by Google
- Blended parse cost per PDF exceeds $0.002

### 3.4.2 Two Processing Buckets

Routing is determined entirely by source bucket path — no document classification, no urgency tag. The ingestion Lambda reads the bucket prefix and enqueues to the corresponding SQS queue.

| **Dimension** | **Fast Bucket** | **Slow Bucket** |
|---|---|---|
| Bucket | `s3://parspec-raw/fast/` | `s3://parspec-raw/slow/` |
| Processing need | Result needed immediately | Result can appear on next search |
| Volume | ~3% of total | ~97% of total |
| Gemini tier | Standard/Priority — real-time, guaranteed latency | Flex (1–15 min) or Batch (up to 24 h), 50% off |
| Compute | Lambda (sub-15 min ceiling) | Fargate tasks (no ceiling) |
| Target latency | ≤ 5 sec | 1–15 min (Flex) / up to 24 h (Batch) |
| SQS queue | `parspec-fast-queue` (FIFO) | `parspec-slow-queue` (FIFO) |

### 3.4.3 Bigger LLM Fallback — Inferred and Empty Fields Only

The bigger LLM (Gemini 2.5 Flash at $0.15/$1.25 per MTok input/output) is invoked specifically for fields tagged **Inferred** or **Empty** — not as a blanket confidence-threshold fallback. No human review queue sits in the hot path.

Every extracted field carries a provenance label:

- **Present:** Value literally written in the PDF. Indexed directly.
- **Inferred:** Value implied by strong context (e.g., "IP65 rated" → infer Damp + Wet). Bigger LLM invoked with a targeted prompt to resolve uncertain inferences.
- **Empty:** Genuinely absent and not inferable. Bigger LLM invoked to confirm absence and attempt recovery from broader context. If still unresolvable, stored as `Empty` — not a failure.

Fields unresolvable after the bigger LLM pass are stored as `Empty`. Human review runs as an offline periodic audit, never in the real-time pipeline.

### 3.4.4 Diff-Based Incremental Extraction

**Problem:** ~300K PDFs change every bi-weekly cycle. Most changes are minor (wattage correction, new SKU row) — yet full re-extraction consumes full token cost. This applies to both Fast and Slow path PDFs.

**Solution:**

```python
import difflib

def compute_section_diff(old_markdown: str, new_markdown: str) -> tuple[str, float]:
    old_lines = old_markdown.splitlines()
    new_lines = new_markdown.splitlines()
    diff = list(difflib.unified_diff(old_lines, new_lines, lineterm=""))
    changed_lines = sum(1 for l in diff if l.startswith(("+", "-")) and not l.startswith(("+++", "---")))
    changed_fraction = changed_lines / max(len(old_lines), 1)
    return "\n".join(diff), changed_fraction

diff_text, changed_fraction = compute_section_diff(old_md, new_md)

if changed_fraction < 0.30:
    delta = llm_extract_delta(diff_text, existing_sku_record)
    updated_sku_record = merge_delta(existing_sku_record, delta)
else:
    updated_sku_record = llm_extract_full(new_md)
```

Targeted partial re-extraction prompt:

```
You are updating an existing product record. Below are the only sections 
that changed. Update only the affected fields. Return a JSON delta 
containing only the modified fields.

Existing SKU record: <existing_json>
Changed sections: <diff_text>
```

`merge_delta` applies only returned fields; all others retain existing values and provenance. Fields removed in diff → `Empty`; new fields → `Present`/`Inferred`.

**Cost savings:**

| **Scenario** | **Token cost / changed PDF** | **Annual cost (300K × 26 cycles)** |
|---|---|---|
| Full re-extraction | ~7,500 tokens avg | ~$9,200 |
| Diff-based (typical ~10% of doc changes) | ~750–1,500 tokens avg | ~$1,100–$2,200 |
| **Estimated saving** | **~70–85% reduction** | **~$7,000–$8,100/year** |

The diff layer lives entirely within the existing Fargate extraction task — no new infrastructure.

### 3.4.5 Elasticsearch for Search

Parspec already operates Elasticsearch. The query rewriter (small LLM, Redis-cached) converts natural language into structured filters before hitting Elasticsearch — so the search index receives a well-formed structured query, not a raw natural-language string. ES 8.x keyword and range filters over structured fields deliver accurate, low-latency results at 1.5M-scale without requiring vector search for the primary query path. Revisit when the corpus exceeds ~50M SKU records.

### 3.4.6 Compute — Lambda for Fast, Fargate for Slow

| **Stage** | **Path** | **Compute** | **Why** |
|---|---|---|---|
| Ingestion: normalize, hash, dedup, route | All traffic | Lambda | Short-lived (~100–500 ms), event-triggered |
| Parse + Extract + Diff-Extract + Postprocess | Fast (~3%) | Lambda | Gemini Standard responds in ~3–5 sec, well inside Lambda's 15-min ceiling |
| Parse + Extract + Diff-Extract + Postprocess | Slow (~97%) | Fargate task (ephemeral ECS) | Flex up to 15 min; Batch polls up to 24 h — no ceiling needed |
| Fallback model | Any — inline in extraction task | Same Fargate/Lambda task | No separate compute needed |

### 3.4.7 SQS — Burst Buffering and Fault Tolerance

Two FIFO queues (one per path) sit between every producer and consumer, providing burst buffering, deduplication (MessageDeduplicationId = SHA-256), path-based routing, fault tolerance, and automatic Flex → Batch failover. SQS holds backlog for up to 14 days.

---

# 4. Pipeline Stages in Detail

## 4.1 S3 Layout

| **Bucket** | **Purpose** | **Key Pattern** |
|---|---|---|
| `parspec-raw/fast/` | Fast-path landing zone | `<yyyy-mm-dd>/<sha256>.pdf` |
| `parspec-raw/slow/` | Slow-path landing zone | `<yyyy-mm-dd>/<sha256>.pdf` |
| `parspec-canonical/` | Parse-once artifact (Markdown + images + raw JSON) | `<sha256>/document.md`, `/images/*.webp`, `/raw.json`, `/schema-version.txt` |
| `parspec-derived/` | Per-schema-version extracted outputs | `v<schema>/<sha256>/products.json` |

## 4.2 Ingestion Lambda

Triggered by EventBridge on S3 PUT:

1. **Normalize to PDF.** Input may be DOCX, HTML, or PPT — convert via PyMuPDF (DOC/DOCX) or headless Chromium (HTML).
2. **Compute SHA-256.** This is the idempotency key for the entire pipeline.
3. **Early-exit on hash collision.** If the hash exists in DynamoDB `pdf_registry`, append the new URL to `source_urls` and return.
4. **Route by source bucket.** `fast/` → `parspec-fast-queue`. `slow/` → `parspec-slow-queue`. No classification, no urgency tag.

Document categorization (lighting/plumbing/HVAC) is handled downstream at extraction via schema-subsetting — not at ingestion.

## 4.3 Parser Stage

The "parse-once" step. Input: PDF on S3. Output: canonical Markdown + page images + raw structured JSON skeleton. Runs exactly once per content hash and is stored permanently.

## 4.4 Diff Extraction Stage

For changed PDFs on either path (hash changed, prior canonical artifact exists). See §3.4.4 for implementation details.

## 4.5 Extractor Stage (Schema-Driven)

Input: canonical Markdown + page images + active attribute schema (Pydantic-serialized). Output: 1..N SKU records validated against the schema, with product images classified within the same call — no separate image classification step.

Every extracted field is tagged:

- **Present:** Value literally in the PDF.
- **Inferred:** Implied by strong context; bigger LLM invoked to resolve uncertain inferences.
- **Empty:** Absent and uninferable — bigger LLM invoked to attempt recovery; stored as Empty if still unresolvable.

## 4.6 Post-Processing & Validation

| **Check** | **Example Input** | **Normalized Output** |
|---|---|---|
| Decimal locale | 47,2" or 47.2" | 47.2 (inches, float) |
| Fraction → decimal | ⅝", 1-1/2" | 0.625, 1.5 |
| Unit disambiguation | "24V" (ambiguous) | Reject; force AC/DC tag |
| Range vs point | "120-277V" | {min: 120, max: 277, unit: V} |
| CCT validation | 35000K (typo) | Flag as outlier |
| Enum coercion | "rec. mtd." | "Recessed" (synonym map) |
| Multi-value parse | "0-10V, DALI" | ["0-10V", "DALI"] |
| NaN/inf on numeric | NaN from bad extraction | null |

## 4.7 Bigger LLM Fallback

Fields tagged Inferred or Empty are re-run through Gemini 2.5 Flash ($0.15/$1.25 per MTok input/output) with a targeted prompt scoped to those specific fields. After this pass, every field resolves to Present / Inferred / Empty. Fields that remain genuinely unresolvable are stored as `Empty`.

---

# 5. Search & Serving

## 5.1 Index Design

A single Elasticsearch index `products_v<schema_version>` — one document per SKU. A new schema version creates a new index; aliases flip atomically (zero-downtime migrations).

| **Field** | **Type** | **Role** |
|---|---|---|
| `model_number` | keyword + ngram | Exact + fuzzy match |
| `category`, `mounting_type` | keyword | Structured filter |
| `wattage_w`, `cct_k`, `lumens`, `cri` | float/integer | Range filters |
| `voltage_min`, `voltage_max` | integer | Range filter |
| `description_card` | text (BM25) | Free-text match component |
| `image_embedding` | dense_vector (Gemini Embedding 2) | Reverse image search |
| `provenance.<field>` | keyword | Per-field Present/Inferred/Empty tag |
| `source_pdf_hash`, `source_pages` | keyword, integer[] | Provenance/explainability |
| `extractor_version` | keyword | Re-extraction audit trail |

## 5.2 Query Flow

Query → Query Rewriter (small LLM, Redis-cached): NL → structured filters → Elasticsearch search (top-20) → clicks captured in DynamoDB per-tenant preference memory.

Because the query rewriter converts natural language into fully structured filters before Elasticsearch sees the request, the search layer operates on precise field-level constraints — no vector similarity needed for the primary retrieval path.

Example: "2-inch aperture downlights with DALI dimming, 3000K, 120V" → structured filters (aperture=2", category=downlight, dimming=DALI, CCT=3000K, voltage=120V).

## 5.3 Image Search

Gemini Embedding 2 generates image embeddings at $0.00012/image, stored as the `image_embedding` dense_vector. This step runs in parallel with post-processing and the bigger LLM fallback — not sequentially after them — reducing overall latency. Supports reverse image search (upload product photo → find matching SKUs). No GPU infrastructure required.

---

# 6. Unit Economics

## 6.1 Input Assumptions

| **Assumption** | **Value** |
|---|---|
| Total PDF corpus | 1,500,000 |
| New/changed PDFs per cycle (bi-weekly) | 300,000 (20%) |
| Changed PDFs: minor change (<30% of doc) | ~70% |
| Changed PDFs: major revision (≥30%) | ~30% |
| Avg diff payload (minor changes) | ~1,200 tokens input + 500 tokens output |
| Avg tokens for full extract | ~7,500 |
| Cycles per year | 26 |
| % PDFs triggering bigger LLM fallback (Inferred/Empty fields) | ~10% |
| Slow-path Gemini discount (Flex/Batch) | 50% |

## 6.2 Per-PDF Cost Breakdown

**Gemini 2.5 Flash-Lite pricing:** $0.10/$0.40 per MTok input/output (standard); 50% off on Flex/Batch.
**Gemini 2.5 Flash (bigger LLM) pricing:** $0.15/$1.25 per MTok input/output.

| **Cost Line** | **Unit Price** | **Fast Path** | **Slow Path (50% off)** | **Blended** |
|---|---|---|---|---|
| Parse: Gemini 2.5 Flash-Lite input | $0.10 / MTok | $0.000600 | $0.000300 | $0.000309 |
| Parse: Gemini 2.5 Flash-Lite output | $0.40 / MTok | $0.000800 | $0.000400 | $0.000412 |
| Extract (full): Gemini 2.5 Flash-Lite input | $0.10 / MTok | $0.000300 | $0.000150 | $0.000155 |
| Extract (full): Gemini 2.5 Flash-Lite output | $0.40 / MTok | $0.000300 | $0.000150 | $0.000155 |
| Diff-extract (minor changes, 70% of churn) | 15% of full extract | — | $0.000045 | $0.000044 |
| Image embedding (Gemini Embedding 2) | $0.00012/image | $0.000120 | $0.000120 | $0.000120 |
| Bigger LLM fallback — Inferred/Empty fields (10% of PDFs) | ~$0.008/call | $0.000800 | $0.000800 | $0.000800 |
| AWS: Lambda + SQS + EventBridge | amortized | $0.000080 | $0.000080 | $0.000080 |
| AWS: S3 storage + PUT/GET | amortized | $0.000040 | $0.000040 | $0.000040 |
| AWS: Elasticsearch indexing + storage | amortized | $0.000200 | $0.000200 | $0.000200 |
| Misc (monitoring, logging, DynamoDB) | amortized | $0.000050 | $0.000050 | $0.000050 |

**Totals (per PDF):**

| **Scenario** | **Cost / PDF** | **vs $0.002 target** |
|---|---|---|
| All Fast path (worst case) | ~$0.003 | 50% over |
| All Slow path, no fallback (best case) | ~$0.00106 | 47% under |
| **Blended (97% Slow + 3% Fast, 10% bigger LLM fallback, diff-based churn)** | **~$0.00104** | **48% under target** |

---

# 7. Tech Stack

| **Layer** | **Technology** | **Why** |
|---|---|---|
| Compute — short handlers | AWS Lambda | Sub-second event handlers; scales 0 → thousands instantly |
| Compute — Slow-path extraction | AWS Fargate tasks (ephemeral ECS) | No 15-min timeout; serverless, no idle cost |
| Orchestration | EventBridge (routing), SQS FIFO (queue + dedup + retry) | Simple linear DAG; sufficient for this topology |
| Storage | S3 (raw/canonical/derived), DynamoDB (registry, preferences), Redis Elasticache (query cache) | S3 tiered lifecycle; DynamoDB for low-latency KV; Redis for sub-ms cache hits |
| Diff computation | Python `difflib.unified_diff` (section-level) | Free, fast, runs inside the existing Fargate task |
| Search | Elasticsearch 8.x | Already in production at Parspec; LLM query rewriting eliminates need for hybrid vector search |
| Primary VLM | Gemini 2.5 Flash-Lite (Flex tier) | Best $/quality for document understanding at scale |
| Bigger LLM | Gemini 2.5 Flash | Handles all Inferred and Empty field resolution |
| Query-rewrite LLM | Gemini 2.5 Flash-Lite (Redis-cached) | Small, fast, cacheable; output feeds directly into structured ES filters |
| Image embeddings | Gemini Embedding 2 ($0.00012/image, API) | No GPU infrastructure; pay-per-use |
| Schema / validation | Pydantic v2, JSON Schema | Python-native, runtime validation |
| API layer | FastAPI on Fargate behind API Gateway | Type-safe, async, OpenAPI |
| IaC | Terraform | Standard |
| CI/CD | GitHub Actions → ECR → Lambda/Fargate deploy | Prompt and schema changes flow through the same pipeline as code |
| Observability | OpenTelemetry, Datadog APM/logs/metrics, Langfuse (LLM call tracing + eval) | Non-negotiable for per-call prompt/response/cost inspection |
| Feature flags / config | AWS AppConfig | Per-tenant schema versions, A/B extraction strategies |
| Secrets | AWS Secrets Manager | Rotatable, audited |

---

# 8. Risks & Future Considerations

## 8.1 Risk Register

| **Risk** | **Likelihood** | **Impact** | **Mitigation** |
|---|---|---|---|
| VLM hallucination on specific field types | High | Medium | Per-attribute F1 dashboards; periodic offline eval on stratified sample |
| Bigger LLM fallback rate exceeds 10% (cost spike) | Medium | Medium | Monthly threshold re-calibration; alert if fallback rate > 15% for 24h; still ~4× cheaper than human review |
| Diff misses semantic change (text looks similar, value updated) | Medium | Low | Conservative 30% threshold; any diff triggers field-level re-check; monitor per-attribute F1 on changed PDFs separately |
| Long-tail attributes stuck at 70–85% F1 | High | Medium | Per-attribute F1 targets; bigger LLM for rare-but-critical fields |
| Gemini pricing increases or model deprecation | Medium | High | Multi-vendor abstraction layer; Anthropic and OpenAI as fallbacks |
| Flex capacity scarce during demand spikes | Medium | Low | SQS holds backlog 14 days; automatic Flex → Batch failover |
| Elasticsearch saturates at 5M+ SKUs | Medium | Medium | Index sharding by category; Qdrant migration path for >50M vectors |
| PDF written to wrong bucket (bulk volume to fast bucket) | Low | Medium | Per-source rate limits on fast-queue SQS; IAM restricts which processes can write to which bucket |
| Prompt injection via PDF content | Medium | Medium | Explicit prompt shielding; Pydantic-validated outputs; sanitize canonical MD before reuse |

## 8.2 Future Considerations

- **Direct-to-manufacturer-website linking.** Store the original source URL on every SKU.
- **Template learning.** After ~50 PDFs from a manufacturer, build per-manufacturer extraction hint layers to drop token count ~40%.
- **Grounding / verifiable outputs.** Cite specific page and bounding box for every extracted attribute.
- **MCP / agentic search.** Expose the search index via Model Context Protocol.
- **RL from bigger LLM corrections.** Every delta JSON from the bigger LLM fallback is a training signal. Collect periodically to fine-tune the primary Gemini 2.5 Flash-Lite extractor prompt, reducing fallback rate over time.
- **Per-page hash tracking for large PDFs.** For PDFs > 50 pages, track per-page hashes for page-level diff granularity.
- **Per-chunk metadata.** Store extractor version, model version, page-level hash, and timestamp per SKU.
- **Inferred-value offline audit.** Periodic batch job routing a sample of `Inferred` values to offline evaluation to measure inference accuracy and tighten the inference prompt.

---

# 9. Appendix

## 9.1 Glossary

| **Term** | **Meaning** |
|---|---|
| Canonical artifact | Parse-once output stored permanently: Markdown + page images + raw JSON + schema version |
| SKU record | One row in the search index — one orderable product |
| Present | Field value literally written in the PDF |
| Inferred | Field value implied by strong context; not explicitly written; bigger LLM invoked to resolve |
| Empty | Field absent and uninferable — bigger LLM invoked to confirm; stored as Empty if unresolvable |
| Diff-extract | Partial re-extraction using only changed sections of a modified PDF, merged into the existing SKU record |
| Bigger LLM | Gemini 2.5 Flash ($0.15/$1.25 per MTok) — invoked for Inferred and Empty field resolution |
| Schema version | Pinned Pydantic model definition; re-extractions run against a specific version |
| F1 | Harmonic mean of precision and recall; per-attribute F1 is the primary accuracy metric |
| Fast bucket | `s3://parspec-raw/fast/` — landing zone for PDFs requiring real-time processing |
| Slow bucket | `s3://parspec-raw/slow/` — landing zone for PDFs processed on Flex/Batch at 50% cost |

## 9.2 Monitoring & Alerting

- Per-attribute F1 (weekly rolling; alert on >2% drop cycle-over-cycle)
- Cost per PDF (daily; alert if >$0.0025)
- Bigger LLM fallback rate — Inferred/Empty fields (alert if >15% for 24 hours)
- Diff-extract ratio (% of changed PDFs using partial vs full re-extraction; monitor for drift)
- Flex p95 per-PDF latency (alert if >30 min)
- Bi-weekly cycle completion (alert if 300K-PDF backlog >48 hours from cycle start)
- Search latency p95 (alert if >1 sec)
- Langfuse: per-prompt success rate, per-model token cost, regression on eval set after any prompt change

---
