```
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
 │        │  S3 ObjectCreated event                                                │
 │        ▼                                                                        │
 │  ┌─────────────┐      ┌──────────────────────────────────────────────────────┐  │
 │  │ EventBridge │─────▶│  Ingestion Lambda  (~100–500 ms)                     │  │
 │  └─────────────┘      │  · Normalize to PDF  (PyMuPDF / headless Chromium)   │  │
 │                       │  · Compute SHA-256                                   │  │
 │                       │  · Query pdf_registry → early-exit on hash match     │  │
 │                       │  · Route: fast/ → fast-queue · slow/ → slow-queue    │  │
 │                       │  · Write status=PENDING to DynamoDB                  │  │
 │                       └──────────────────────────────────────────────────────┘  │
 │                                 │                      │                        │
 │                        fast-queue (FIFO)          slow-queue (FIFO)             │
 └─────────────────────────────────────────────────────────────────────────────────┘
                                    │                      │
 ┌──────────────────────────────────┼──────────────────────┼────────────────────────┐
 │  PLANE 2 · EXTRACTION            │                      │                        │
 │                                  ▼                      ▼                        │
 │                    ┌─────────────────────┐  ┌───────────────────────────────┐    │
 │                    │  FAST PATH          │  │  SLOW PATH                    │    │
 │                    │  Lambda             │  │  Fargate Task (ephemeral ECS) │    │
 │                    │  SLA: ≤ 5 sec       │  │  SLA: 1 min – 24 h            │    │
 │                    │  Gemini 2.5         │  │  Gemini 2.5 Flash-Lite        │    │
 │                    │  Flash-Lite         │  │  Flex / Batch  (50% off)      │    │
 │                    │  Standard/Priority  │  │                               │    │
 │                    └──────────┬──────────┘  └──────────────┬────────────────┘    │
 │                               │                             │                    │
 │                               └──────────┬──────────────────┘                    │
 │                                          ▼                                       │
 │                          ┌───────────────────────────────┐                       │
 │                          │  EXTRACTOR STAGE              │                       │
 │                          │  Gemini 2.5 Flash-Lite        │                       │
 │                          │  Pydantic schema · temp=0     │                       │
 │                          │  Single structured call       │                       │
 │                          │                               │                       │
 │                          │  In:  PDF (native doc input)  │                       │
 │                          │       + active schema         │                       │
 │                          │  Out: 1..N SKU records        │                       │
 │                          │       + image classification  │                       │
 │                          │                               │                       │
 │                          │  Fields: Present / Inferred   │                       │
 │                          │          / Empty              │                       │
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
 │          │  Triggered: Inferred +    │                 │                         │
 │          │  Empty fields (~10% PDFs) │                 │                         │
 │          │  Unresolved → Empty       │                 │                         │
 │          └─────────────┬─────────────┘                 │                         │
 │                        │                               │                         │
 │                        └───────────────┬───────────────┘                         │
 │                                        │                                         │
 └────────────────────────────────────────┼─────────────────────────────────────────┘
                                          │
                                          ├──────────────┐
                                          │              │
                                          │   ┌──────────┴──────────────┐
                                          │   │  DynamoDB               │
                                          │   │  status=COMPLETE        │
                                          │   │  + pdf_registry update  │
                                          │   └─────────────────────────┘
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
 │                           │  Query Rewriter         │                             │
 │                           │  Gemini 2.5 Flash-Lite  │                             │
 │                           │  Redis-cached           │                             │
 │                           │  NL → structured filter │                             │
 │                           └─────────────────────────┘                             │
 │                                         ▲                                         │
 │                                         │  search query                           │
 │                                   [ CLIENT ]                                      │
 │                                                                                   │
 │                           Clicks → DynamoDB preference memory                     │
 └───────────────────────────────────────────────────────────────────────────────────┘
```
