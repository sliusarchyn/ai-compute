# **Finance / quant research with ML**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: notebook experiments, small backtests, feature sanity checks  
* **Data ingest \+ feature engineering**: build factors/features from market \+ alt data  
* **Batch training**: train forecasting / classification models on large time series  
* **Hyperparameter sweeps**: many parallel runs (common)  
* **Backtesting / simulation**: replay strategies over long histories (CPU-heavy)  
* **Online inference**: low-latency scoring (signals), sometimes near-realtime  
* **Risk / portfolio optimization**: constrained optimization, scenario stress tests  
* **Research eval/regression**: reproducibility, leakage checks, drift monitoring

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (research notebooks) | 1–10 datasets / small windows | 0 GPUs → 1×24GB | 16–32 vCPU, 64–256GB | 0.5–2TB | 30 min–4h → 1–3 days | Many small reads; frequent reruns; lots of logs |
| Data ingest \+ feature engineering | 10GB–10TB/day | 0 GPUs → 1×24GB | 32–256 vCPU, 128GB–2TB | 1–16TB | 1–12h → 1–7 days | Heavy read/write; joins; parquet shards; caching |
| Batch training (tabular/time series) | 1M–10B rows | 0–1×24GB → 4×48GB | 32–256 vCPU, 128GB–2TB | 1–16TB | 2–12h → 1–14 days | Streaming reads; periodic checkpoints; metrics logs |
| Batch training (deep sequence models) | large sequences | 1×48GB → 8×80GB | 64–512 vCPU, 256GB–4TB | 2–32TB | 1–7 days → 2–8 weeks | High throughput reads; big checkpoints; DDP scaling |
| Hyperparameter sweeps | 50–50k trials | 1×24GB → 16×48GB (parallel) | 32–512 vCPU, 128GB–4TB | 1–32TB | 2–24h → days–weeks | Many small runs; heavy scheduling; artifact explosion |
| Backtesting / simulation | years–decades history | 0 GPUs → 1×24GB | 64–1024 vCPU, 256GB–8TB | 1–32TB | 1–12h → days–weeks | Read-heavy; lots of intermediate outputs; CPU-bound |
| Online inference (signals) | 1k–1M req/s (varies) | 0–1×24GB → 2×48GB | 16–256 vCPU, 64GB–1TB | 200GB–2TB | continuous | Tail latency sensitive; small reads; strict uptime |
| Risk / optimization | 1k–1M portfolios/scenarios | 0 GPUs → 1×24GB | 64–512 vCPU, 256GB–4TB | 0.5–8TB | 1–6h → 6–72h | CPU \+ RAM heavy; matrix ops; writes reports |

### **Parallelism expectations**

* Feature engineering \+ backtests scale by **data partitioning** (CPU clusters).  
* Training:  
  * tabular often CPU-first (GPU optional)  
  * deep models scale with multi-GPU DDP  
* Sweeps are the major “scheduler stress test”: lots of short/medium jobs.

### **Failure tolerance**

* Research workloads must be reproducible:  
  * fixed data snapshots, versioned features  
  * deterministic seeds  
* Backtests must be restartable by partition/time range.  
* Spot/preemptible:  
  * good for sweeps and feature builds if resumable  
  * risky for latency-sensitive online inference

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined later via telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **Python stacks**: pandas/polars, numpy, sklearn, xgboost/lightgbm, PyTorch/JAX for deep models  
* **Data lake**: parquet/arrow; object storage; feature stores  
* **Distributed compute**: Spark/Ray/Dask-like patterns (common for features/backtests)  
* **Experiment tracking**: MLflow/W\&B-like tooling; reproducible envs  
* **Low-latency serving**: gRPC/REST; model caches; sometimes C++/Rust components

### **CUDA/driver constraints**

* Only needed if GPU training/inference; stable CUDA 12.x baseline is fine.  
* Pin versions for reproducible research results.

### **Container vs VM requirements**

* Containers are default.  
* Some regulated orgs prefer VMs or dedicated tenancy.

### **Storage expectations**

* Local NVMe for caching datasets and intermediate parquet shards.  
* Durable object storage for:  
  * raw data, feature datasets, models, backtest artifacts  
* Strong versioning of datasets/features is a must.

### **Networking needs**

* Private connectivity for data sources (exchanges, vendors).  
* Outbound restrictions and allowlists are common.  
* Online inference may need colocated low-latency networking (depends on strategy).

**Output artifact:** reference runtime image set

* `quant-features`  
* `quant-train-cpu`  
* `quant-train-cu12`  
* `quant-backtest`  
* `quant-serve`  
* `compat`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Proprietary trading signals and positions are core IP.  
* Market data licenses may impose strict usage/storage rules.

### **Common requirements**

* Isolation (often dedicated projects; sometimes dedicated nodes)  
* Encryption at rest \+ TLS in transit  
* Strict RBAC \+ audit logs  
* Data retention rules aligned with vendor licenses

### **Logging rules**

* Don’t log raw positions/strategies by default.  
* Store metadata: run IDs, dataset versions, timings, error codes, aggregate metrics.

**Output artifact:** security/compliance checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* Reproducibility and lineage (data snapshot \+ code \+ env)  
* Fast iteration (queue times kill research velocity)  
* Backtest correctness (no data leakage)  
* Cost control (sweeps can explode)  
* For some: low-latency serving reliability

### **SLO starting points (priors)**

* Notebook/workspace start: p90 \< 10–20 min  
* Batch completion reliability: \>99% with resume/retry  
* Artifact durability: results/models never disappear  
* Observability: job queue time, throughput, cost per run, dataset version clarity

### **Support playbook themes**

* “Backtest differs from yesterday” (data drift, version mismatch)  
* “It’s slow” (I/O bottleneck, bad partitioning)  
* “Sweep is chaotic” (scheduler limits, artifact bloat)  
* “Model is leaking” (train/test contamination)  
* “Serving latency spiked” (cache misses, noisy neighbor)

### **Pricing sensitivity & billing preference**

* Researchers like on-demand \+ spot for sweeps  
* Desks like reservations for predictable monthly spend  
* Serving may be a separate SKU (SLA \+ reserved capacity)

**Output artifact:** SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Notebooks for research → batch runs for training/sweeps → backtests → reports → deploy signals  
* Feature pipelines scheduled daily/hourly  
* Continuous regression: nightly benchmark/backtest gates

### **Auth patterns**

* Org/team RBAC \+ service accounts  
* Enterprise SSO later, but audit logs early

### **Artifact flow**

* Inputs: market data \+ alt data \+ feature configs  
* Outputs: feature datasets, models, backtests, reports, run manifests  
* Destinations: customer storage \+ optional managed registry

### **Golden flows (end-to-end)**

1. **Feature build → train → backtest**  
   * Snapshot data → build features → train models → run backtest suite → publish report \+ artifacts  
2. **Hyperparameter sweep**  
   * Define search space → schedule thousands of trials → track best → export model \+ reproducibility bundle  
3. **Deploy signal scoring**  
   * Promote model → deploy low-latency scorer → monitor drift/latency → roll back on regression

