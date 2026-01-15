# **Enterprise document AI (OCR \+ extraction \+ assistants)**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: test on a sample doc set, tweak extraction schemas, prompt templates, eval  
* **OCR (batch)**: scanned PDFs/images → text \+ layout  
* **Classification & routing**: doc type detection (invoice, contract, KYC, etc.)  
* **Field extraction**: structured extraction (tables, key-values) \+ validation rules  
* **RAG indexing**: chunking, embeddings, building indexes, metadata enrichment  
* **Assistant Q\&A**: interactive chat over doc sets (often internal)  
* **Batch “doc-to-JSON” pipelines**: process large archives/nightly drops  
* **Fine-tuning / adapters** (optional): domain-specific extraction accuracy improvements  
* **Eval/regression**: accuracy by field, latency, cost, drift

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (schema \+ prompt iteration) | 100–5k docs | 0–1×24GB → 1×48GB | 16–32 vCPU, 64–256GB | 0.5–2TB | 1–6h → 1–3 days | Many small reads; frequent reruns; lots of logs |
| OCR batch (scan-heavy) | 1k–10M pages/day | 0–1×24GB → 2×48GB | 32–256 vCPU, 128GB–2TB | 1–16TB | 0.2×–2× realtime → 2×–10× realtime | Decode-heavy; many small writes (text+layout); CPU-bound often |
| Classification & routing | 10k–100M docs/day | 0–1×24GB → 1×48GB | 16–128 vCPU, 64GB–1TB | 0.5–8TB | ms–seconds/doc → seconds–minutes/doc | Read text; write labels; high QPS |
| Field extraction (doc→JSON) | 10k–10M docs/day | 1×24GB → 4×48GB | 16–256 vCPU, 64GB–2TB | 1–16TB | seconds/doc → minutes/doc (complex) | Read OCR/layers; write JSON; sometimes table-heavy |
| Batch archive processing | 100k–1B docs/job | 1×24GB → 8×80GB | 64–512 vCPU, 256GB–4TB | 2–64TB | 6–48h → days–weeks | Massive read/write; artifact explosion; queueing dominates |
| RAG indexing (embeddings) | 10k–1B chunks | 1×24GB → 4×48GB | 16–256 vCPU, 64GB–2TB | 1–32TB | 1–12h → days–weeks | Read text; write vectors \+ metadata; many small ops |
| Assistant Q\&A (interactive) | 10–100k concurrent | 1×24GB → 8×80GB (scale-out) | 16–256 vCPU, 64GB–2TB | 200GB–4TB | continuous | Tail latency sensitive; retrieval-heavy; caching helps |
| Fine-tuning/adapters (optional) | curated labeled docs | 1×48GB → 4×80GB | 32–256 vCPU, 128GB–2TB | 1–16TB | 6–48h → 2–14 days | Dataset reads; checkpoints; eval suites |

### **Parallelism expectations**

* OCR \+ extraction scale by **doc/page sharding** across many workers.  
* Indexing scales by **chunk sharding**.  
* Assistants scale by **replica count** \+ caching; retrieval is often the bottleneck.

### **Failure tolerance**

* Pipelines must be restartable/idempotent:  
  * by `doc_id`, `page_id`, `batch_id`, `schema_version`  
* Spot/preemptible is fine for OCR/indexing/batch processing if resumable.  
* Interactive assistants need redundancy and warm pools.

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined later via telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **OCR**: CPU-heavy OCR engines; sometimes GPU acceleration for layout models  
* **Layout parsing**: table extraction, form understanding, bounding boxes  
* **LLM extraction**: prompt templates \+ structured output validation (JSON schema)  
* **RAG**: chunking \+ embeddings \+ vector DB \+ metadata store  
* **Serving**: batch workers \+ interactive API; queues for large drops  
* **Search**: full-text \+ vector \+ filters

### **CUDA/driver constraints**

* Needed for GPU LLM inference / embeddings / some layout models.  
* Stable CUDA 12.x baseline; compat images for pinned stacks.  
* Version pinning is crucial (model behavior affects extraction).

### **Container vs VM requirements**

* Containers default.  
* VMs/dedicated nodes show up for compliance, private networking, and strict isolation.

### **Storage expectations**

* Local NVMe for:  
  * fast decode cache (PDF rendering)  
  * intermediate artifacts (images per page, layout JSON)  
  * vector indexing scratch  
* Durable object storage for:  
  * source docs, OCR outputs, extracted JSON, embeddings, audit logs  
* Many-small-files problem is common (pages/chunks).

### **Networking needs**

* Private connectivity (VPN/peering) for enterprise doc stores.  
* Egress controls are common (docs often sensitive).  
* Assistants may need multi-region for low latency.

**Output artifact:** reference runtime image set

* `doc-ocr`  
* `doc-layout`  
* `doc-extract-llm-cu12`  
* `doc-index-embed-cu12`  
* `doc-assistant-serve-cu12`  
* `doc-batch-runner`  
* `compat`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Often contains PII, contracts, invoices, HR docs, legal docs.  
* Strong compliance expectations even if not formally regulated.

### **Common requirements**

* Region lock / residency  
* Encryption at rest \+ TLS in transit  
* Strong RBAC (department/project level)  
* Audit trails (who accessed which doc, who extracted/exported)  
* Retention policies \+ legal holds

### **Logging rules**

* Do not store doc content in application logs by default.  
* Store metadata only: doc IDs, schema versions, timings, error codes, aggregate stats.  
* If storing content for QA, it must be opt-in with redaction controls.

**Output artifact:** security/compliance checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* Extraction accuracy by field (precision/recall, “critical fields”)  
* Throughput for nightly/weekly drops  
* Predictable costs per page/doc  
* Reproducibility (schema \+ model version pinned)  
* Security posture (audits, access control)

### **SLO starting points (priors)**

* Batch completion reliability: \>99% with resume/retry  
* Assistant p95 latency targets per customer  
* Artifact durability: extracted JSON and audit logs never disappear  
* Observability: queue time, pages/min, error rates by doc type, cost per doc

### **Support playbook themes**

* “Accuracy dropped” (template drift, new doc formats, model version drift)  
* “Tables are wrong” (layout parsing issues)  
* “Too slow” (PDF render bottleneck, small files overhead, storage saturation)  
* “Costs exploded” (over-processing, no caching, too-large LLM)  
* “Security review” (audit logs, residency, isolation)

### **Pricing sensitivity & billing preference**

* Common pricing axes:  
  * per page (OCR)  
  * per doc (extraction)  
  * per token / per query (assistant)  
  * storage \+ retention  
* Enterprises want predictable invoices; reservations/commits are common.

**Output artifact:** SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Upload/drop docs → OCR → classify → extract → validate → export to ERP/CRM  
* Index doc corpora → assistant Q\&A for employees  
* Continuous improvement loop: label errors → tune prompts/models → regression tests

### **Auth patterns**

* Org/project RBAC, service accounts  
* SSO later; audit logs now

### **Artifact flow**

* Inputs: PDFs/images \+ metadata (vendor, date, department)  
* Outputs: OCR text/layout, extracted JSON, tables, embeddings, indexes, audit logs  
* Destinations: customer bucket, DMS, ERP/CRM, search systems

### **Golden flows (end-to-end)**

1. **Invoice/contract pipeline**  
   * Ingest → OCR → classify → extract fields → validate schema → export to ERP → store audit trail  
2. **Archive backfill**  
   * Batch ingest millions of docs → parallel OCR/extraction → write outputs \+ indexes → QA sampling  
3. **Internal assistant**  
   * Index corpora → chat with retrieval \+ citations → RBAC enforcement → log access metadata \+ monitor quality

