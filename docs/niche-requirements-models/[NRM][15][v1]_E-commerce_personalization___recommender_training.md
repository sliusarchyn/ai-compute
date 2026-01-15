# **E-commerce personalization / recommender training**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: small offline experiments, feature sanity checks, metric validation  
* **Data ingest \+ sessionization**: clickstream/events, catalog updates, joins, identity resolution  
* **Feature engineering**: user/item features, sequence features, negative sampling datasets  
* **Embedding training**: item/user embeddings, retrieval models (two-tower)  
* **Ranking model training**: learning-to-rank, deep CTR/CVR prediction, sequence recommenders  
* **Batch inference**: nightly recompute of embeddings, candidate generation, segment scoring  
* **Online inference**: realtime recommendations (latency \+ cache critical)  
* **A/B testing \+ eval**: offline metrics (NDCG, AUC) \+ online lift, drift monitoring  
* **Backfills / replays**: replay historical data after schema/model changes

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (offline experiments) | 1–50GB data / small windows | 0 GPUs → 1×24GB | 16–32 vCPU, 64–256GB | 0.5–2TB | 1–6h → 1–3 days | Many small reads; frequent reruns; lots of logs |
| Data ingest \+ sessionization | 10GB–100TB/day | 0 GPUs → 1×24GB | 32–512 vCPU, 128GB–4TB | 1–32TB | continuous / hourly-daily | Heavy joins/writes; schema evolution; backfills common |
| Feature engineering \+ negatives | 100GB–100TB/job | 0 GPUs → 1×24GB | 64–512 vCPU, 256GB–4TB | 2–64TB | 6–48h → days–weeks | Write-heavy (parquet shards); large intermediates |
| Embedding / retrieval training | 10M–10B interactions | 1×24GB → 4×48GB | 32–256 vCPU, 128GB–2TB | 1–32TB | 6–48h → 2–14 days | Streaming reads; checkpointing; embedding tables big |
| Ranking model training (deep CTR) | 10M–10B examples | 1×48GB → 8×80GB | 64–512 vCPU, 256GB–4TB | 2–64TB | 1–7 days → 2–8 weeks | High-throughput reads; big checkpoints; DDP scaling |
| Batch inference (embeddings refresh) | 1M–1B users/items | 0–1×24GB → 2×48GB | 32–256 vCPU, 128GB–2TB | 1–32TB | 1–12h → 1–7 days | Read-heavy; writes large embedding dumps/indexes |
| Candidate generation / ANN indexing | 1M–1B vectors | 0–1×24GB → 1×48GB | 32–256 vCPU, 128GB–2TB | 1–32TB | 1–12h → 1–7 days | Build \+ write index; many-small-files possible |
| Online inference (prod recs) | 1k–1M req/s | 0–1×24GB → 2×48GB | 32–512 vCPU, 128GB–4TB | 200GB–4TB | continuous | Tail latency sensitive; heavy cache/kv-store reads |
| A/B eval \+ reporting | daily/weekly | 0 GPUs → 1×24GB | 32–512 vCPU, 128GB–4TB | 0.5–8TB | 1–12h → days | Read-heavy; aggregations; writes dashboards/reports |

### **Parallelism expectations**

* Ingest/features: partition by time/user/site → big CPU clusters.  
* Training: DDP for deep models; embedding models often smaller GPU needs but big CPU/I/O.  
* Inference: batch refresh is parallel; online serving scales horizontally with caching.

### **Failure tolerance**

* Batch pipelines must be idempotent by `date_partition`, `dataset_version`, `model_version`.  
* Backfills must be resumable (they’re expensive and common).  
* Online inference needs redundancy; spot/preemptible is not for front-door traffic.

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined later via telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **Data processing**: Spark/Ray/Dask-like; parquet/arrow; lakehouse patterns  
* **Recommenders**: two-tower retrieval, ranking models, sequence models  
* **ANN search**: vector indexes (HNSW/IVF) and refresh pipelines  
* **Serving**: low-latency APIs \+ caches (Redis-like), feature stores  
* **Experimentation**: offline eval \+ online A/B infrastructure, feature flags

### **CUDA/driver constraints**

* GPU used for training and sometimes for online inference; stable CUDA 12.x baseline works.  
* Pin versions for reproducibility.

### **Container vs VM requirements**

* Containers default.  
* Some enterprises want dedicated nodes/projects.

### **Storage expectations**

* Local NVMe for caching and intermediate parquet shards.  
* Durable object storage for raw events, feature datasets, embedding dumps, models.  
* Lots of “many small files” issues → compaction jobs become necessary.

### **Networking needs**

* Private connectivity to event pipelines and databases.  
* Online inference may need multi-region with low latency.

**Output artifact:** reference runtime image set

* `recs-ingest-sessionize`  
* `recs-features`  
* `recs-train-retrieval-cu12`  
* `recs-train-rank-cu12`  
* `recs-batch-infer`  
* `recs-ann-build`  
* `recs-serve`  
* `compat`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* User behavior data can be PII/regulated (depends on region).  
* Vendor/brand data and pricing can be sensitive.

### **Common requirements**

* Region lock / residency  
* Encryption at rest \+ TLS in transit  
* RBAC by team/project  
* Retention and deletion policies (user deletion requests)

### **Logging rules**

* Don’t log raw user identifiers by default; use hashed IDs.  
* Store metadata: dataset versions, run IDs, timings, error codes.

**Output artifact:** security checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* Freshness (how fast new items/events affect recs)  
* Online latency p95/p99  
* Experiment reliability (A/B correctness)  
* Cost control (backfills, embeddings, ANN rebuilds)  
* Reproducibility (same data snapshot \= same offline metrics)

### **SLO starting points (priors)**

* Online rec latency: customer-specific p95 budgets  
* Freshness targets (hourly/daily) per SKU  
* Batch completion reliability: \>99% with resume/retry  
* Observability: queue time, feature freshness, model drift, cache hit rate

### **Support playbook themes**

* “Reds are stale” (pipeline lag, failed refresh)  
* “Latency spikes” (cache misses, hot keys)  
* “Metrics changed” (data leakage, schema drift)  
* “Index rebuild failed” (memory/disk limits)  
* “Costs exploded” (too-frequent refresh, no compaction)

### **Pricing sensitivity & billing preference**

* Batch pipelines: pay for CPU/GPU hours \+ storage  
* Online: reserved capacity \+ SLA tiers  
* Mid-market wants predictable monthly bills

**Output artifact:** SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Daily/hourly pipelines: ingest → features → train/refresh → deploy → monitor  
* Product teams drive A/B tests and rollouts  
* Data science teams iterate offline then ship

### **Auth patterns**

* Org/team RBAC, service accounts  
* Larger orgs: SSO later, audit logs now

### **Artifact flow**

* Inputs: clickstream/events, catalog, pricing, inventory  
* Outputs: feature datasets, embeddings, ANN indexes, models, reports, run manifests  
* Destinations: online serving systems \+ customer storage

### **Golden flows (end-to-end)**

1. **Daily refresh**  
   * Ingest events → sessionize → feature build → refresh embeddings \+ ANN → deploy → monitor  
2. **New model rollout**  
   * Train candidate → offline eval → shadow deploy → A/B test → promote/rollback  
3. **Backfill after schema change**  
   * Recompute features for time range → retrain → rebuild index → rerun regression suite → rollout

