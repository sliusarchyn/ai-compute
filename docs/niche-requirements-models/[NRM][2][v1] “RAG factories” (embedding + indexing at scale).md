# “RAG factories” (embedding \+ indexing at scale)

Here’s a **Niche Requirements Model v1** for **“RAG factories” (embedding \+ indexing at scale)** — same structure as before, with **realistic starting priors** you’ll later replace with interview \+ telemetry.

---

## **1\) Workload anatomy (the “physics”)**

### **Job types you must support**

* **Dev/Test**: connector \+ chunking sanity checks, small sample indexing, retrieval eval.  
* **Backfill indexing**: embed \+ index the whole corpus (hours → days).  
* **Incremental indexing**: continuous updates (minutes → always-on).  
* **Re-embedding**: model upgrade or new chunking strategy (expensive “rebuild”).  
* **Batch retrieval eval**: run test sets, measure recall/precision/latency.  
* **Online retrieval**: production query traffic (latency-sensitive).

### **Workload profile (p50 / p90 shape)**

Assume corp data: docs/wiki/tickets/emails/PDFs, often **10k → 50M chunks**.

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test | 1k–100k chunks | **0–1×24GB → 1×48GB** | 8–16 vCPU, **32–128GB** | 100–300GB | **10–60 min → 2–6h** | Heavy parsing, small embedding writes, lots of logs |
| Backfill embedding | 1M–50M chunks | **1×48GB → 4×80GB** | 16–64 vCPU, **64–512GB** | 0.5–4TB | **3–24h → 2–10 days** | Large sequential reads; massive write of vectors \+ metadata |
| Backfill indexing (vector DB build) | 1M–50M vectors | GPU optional | **32–128 vCPU**, **128GB–1TB RAM** | 1–8TB | **2–16h → 1–7 days** | Index build is write-heavy, may need big RAM spikes |
| Incremental updates | 100–1M chunks/day | **0–1×24GB → 1×48GB** | 8–32 vCPU, 32–256GB | 200GB–2TB | **seconds–minutes per batch** | Bursty; read from source \+ write vectors; needs idempotency |
| Re-embedding / re-chunking | same as corpus | **1×48GB → 8×80GB** | 32–128 vCPU, 128GB–1TB | 1–8TB | **1–7 days → weeks** | Biggest cost driver; often needs parallel sharding |
| Online retrieval | QPS varies | usually CPU | 4–32 vCPU, 16–128GB | 100–500GB | continuous | Read-heavy; tail latency is key |

**Resource envelope reality**

* Embedding generation is often **GPU-bound**, but ingestion/chunking can be **CPU-bound** (PDF parsing, OCR, HTML cleanup).  
* Index building/search can become **RAM-bound** depending on engine and index type.  
* Storage can dominate: vectors \+ metadata \+ caches.

### **I/O patterns (what you must design for)**

* **Reads:** source docs (object store / DB / SaaS APIs); lots of small files is worst-case.  
* **Writes:**  
  * embeddings as arrays (float32/float16/int8) \+ metadata  
  * index structures (can be huge) \+ WAL/segments  
* **Typical sizes (priors):**  
  * chunk text: \~0.5–4 KB average  
  * embedding vectors: often 384–3072 dims  
  * storage per vector (very rough): **0.5–5 KB** depending on dtype \+ metadata \+ index overhead  
    \=\> **10M vectors** can easily become **10s–100s of GB** total in practice.

### **Parallelism**

* Most common: **single node \+ sharding by document/chunk range**.  
* Scale-out: multiple workers embedding in parallel writing to a queue or directly to vector DB ingestion API.  
* Multi-node networking: less about NCCL, more about **coordinated writes and backpressure** (but you still need good networking if you ingest fast).

### **Failure tolerance**

RAG factories *must* be restartable:

* **Idempotent chunk IDs** (doc\_id \+ chunk\_index \+ version hash).  
* Resume from last processed offset.  
* Preemption is acceptable if you checkpoint progress frequently; painful if index builds require long uninterrupted runs.

**Output artifact**

* “Workload profile” table above \+ your chosen p50/p90 targets after interviews.

---

## **2\) Stack \+ environment constraints**

### **Common stacks (what you should assume)**

**Ingestion & chunking**

* Python: text extraction (PDF/HTML), cleaning, chunking, language detection  
* OCR sometimes (expensive) for scanned PDFs

**Embeddings**

* SentenceTransformers / Transformers embeddings  
* Sometimes OpenAI-like APIs, but for “private corp data” they often want **self-hosted embeddings**  
* Quantized embedding models appear for cost (int8/FP16)

**Indexing**

* Vector DB options vary a lot:  
  * managed vs self-hosted  
  * HNSW vs IVF/PQ variants  
* Also hybrid retrieval: BM25 \+ vector

**Orchestration**

* Batch pipelines: queues \+ workers; often Airflow/Temporal-like patterns  
* Streaming updates: webhook → queue → embedding worker → upsert

### **CUDA/driver constraints**

* If embedding uses GPU: standard CUDA 12.x base works for most modern setups.  
* Some customers require “frozen” images for reproducibility.

### **Container vs VM**

* **Containers are typical**.  
* VMs show up when:  
  * corporate policy requires VM isolation  
  * they want custom filesystem/performance tuning for DB nodes

### **Storage expectations**

* Embedding workers love **local NVMe cache** for:  
  * downloaded source artifacts  
  * intermediate chunk files  
  * batching writes  
* Vector DB nodes need:  
  * durable disks (fast SSD), sometimes RAID  
  * snapshots/backups to object storage

### **Networking needs**

* Often: **private-only access** to corp sources (VPN/peering).  
* Outbound restrictions are common (no public internet, allowlist only).  
* High ingest rates stress object store \+ DB network; plan for **10–25GbE** baseline, higher for large rebuilds.

**Output artifact**

* “Reference runtime image set” (pinned):  
  1. `rag-ingest` (parsers \+ chunkers)  
  2. `rag-embed-cu12` (embedding workers GPU-enabled)  
  3. `rag-indexer` (bulk loader / compaction tools)  
  4. `rag-retrieval-eval` (benchmark \+ evaluation tooling)  
  5. Optional per-vectorDB images (admin tools, backups, compaction)

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Corp docs almost always include **IP \+ PII** (names, emails, customer info).  
* Sometimes regulated (contracts, finance, healthcare).

### **Key requirements you’ll hear**

* **Data residency / region lock**  
* **Isolation**:  
  * dedicated nodes for embedding/index (common in enterprise)  
  * dedicated network \+ private endpoints  
* **Encryption**:  
  * at rest (disks \+ backups)  
  * in transit (TLS, private links)  
* **Audit**:  
  * who ran indexing jobs  
  * who accessed search endpoints  
  * retention policies

### **Logging rules**

* Logs cannot contain raw document text by default.  
* Store only:  
  * document IDs  
  * chunk IDs  
  * hashes  
  * size counters  
* Retrieval queries may be sensitive too (users paste private text). Many customers want query logging disabled or redacted.

**Output artifact**

* “Security/compliance checklist” \+ tenancy requirements (dedicated node, private network, encryption, retention, audit/export).

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What they care about**

* **Freshness SLA**: “doc updated → searchable within X minutes”  
* **Index correctness**: missing docs, duplicates, wrong versions  
* **Tail latency** on retrieval (p95/p99)  
* **Cost** of rebuilds (re-embedding can be brutal)  
* **Ingestion reliability**: connectors throttling, rate limits, auth failures

### **Practical SLO starting points (priors)**

* Dev/test pipeline start: **\<10 min p90**  
* Incremental update latency: **p50 \<5 min**, **p90 \<30 min** (depends on sources)  
* Retrieval latency targets (depends on DB \+ size):  
  * p50: **\<100–300ms**, p95: **\<500–1500ms** (rough priors)  
* Backfill completion reliability: “no data loss”, resumable, progress visible

### **Support expectations (top recurring tickets)**

* “Why is indexing slow?” (CPU parsing bottleneck, source throttling, DB compactions)  
* “Why are results stale?” (webhook gaps, failed connector runs)  
* “Why duplicates?” (no stable chunk IDs/versioning)  
* “DB is huge/costly” (metadata bloat, high-dim vectors, poor compaction)  
* “Latency spiked” (index rebuild, compaction, cache miss)

### **Billing preference**

* Customers want pricing separated into:  
  * **embedding compute** (GPU-hours)  
  * **index storage** (GB-month)  
  * **query serving** (CPU/RAM \+ QPS tier)  
* They often accept spot/preemptible for **backfills**, not for **freshness-critical incremental**.

**Output artifact**

* Per-niche SLOs \+ support playbook \+ pricing sensitivity notes.

---

## **5\) User workflow & integrations**

### **How they run RAG factories today**

* Notebooks for experiments (chunking \+ eval)  
* Batch pipelines (cron/Airflow/CI) for backfills  
* Event-driven (webhooks) for incremental updates  
* Tools they integrate with:  
  * document stores (S3/GCS), Confluence/Notion, Google Drive, Slack, Git repos, ticket systems  
  * observability \+ alerting  
  * model registries sometimes (but embeddings are usually treated as infra)

### **Auth patterns**

* Org/project RBAC  
* Service accounts for connectors  
* API keys for retrieval endpoints  
* Enterprise: SSO later; audit logs now

### **Artifact flow**

* Inputs: source docs \+ connector metadata  
* Outputs:  
  * chunk manifests  
  * embedding shards  
  * vector DB snapshots  
  * eval reports (recall@k, MRR, latency histograms)  
* Destination: managed storage or customer-owned buckets

### **Golden flows (end-to-end)**

1. **Backfill from corp storage**  
   * Connect S3/GCS → chunk+embed → bulk load into vector DB → run eval suite → publish retrieval endpoint  
2. **Incremental freshness pipeline**  
   * Webhook/source polling → enqueue changes → embed worker upserts → validate counts/versioning → freshness dashboard  
3. **Rebuild with new embedding model**  
   * Spin parallel index v2 → re-embed in shards → backfill \+ validate → switch traffic (blue/green) → retire v1