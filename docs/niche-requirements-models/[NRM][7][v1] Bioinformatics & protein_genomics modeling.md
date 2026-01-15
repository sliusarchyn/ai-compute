# **Bioinformatics & protein/genomics modeling**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: run a small subset, validate preprocessing, sanity-check metrics  
* **Protein inference**:  
  * **Embedding/annotation** (sequence → vectors, function tags)  
  * **Structure prediction** (sequence → structure-like outputs / confidence maps)  
* **Genomics inference**:  
  * **Batch analysis** (sample-level runs, cohort-level runs)  
  * **Variant/feature pipelines** (alignment \+ calling \+ QC, or ML-based scoring)  
* **Indexing/build steps**: build reference indexes, k-mer tables, feature stores  
* **Training / tuning**:  
  * PEFT tuning on private lab data (common)  
  * full training (rarer, very expensive)  
* **Eval/regression**: benchmark suites, reproducibility checks, drift checks  
* **Production**:  
  * batch “runs” triggered by lab workflows  
  * occasional online inference for interactive analysis (smaller share)

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (pipeline sanity) | 1–10 samples or small FASTA set | 0–1×24GB → 1×48GB | 16–32 vCPU, 64–256GB | 0.5–2TB | 30–120 min → 6–24h | Lots of preprocessing; repeated runs; moderate artifacts |
| Protein embeddings / annotation (batch) | 10k–100M sequences | 1×24GB → 4×48GB | 16–128 vCPU, 64–512GB | 1–8TB | 1–12h → 1–7 days | Read-heavy; write embeddings (GB–TB); batching critical |
| Protein structure inference (batch) | 10–100k proteins | 1×48GB → 4×80GB | 32–256 vCPU, 128GB–1TB | 2–16TB | minutes–hours/protein → hours–days/protein (hard cases) | Heavy scratch; big intermediates; output bundles \+ logs |
| Genomics batch pipelines (sample processing) | 10–10k samples/day | 0–1×24GB → 1×48GB | 32–256 vCPU, 128GB–1TB | 2–16TB | 1–6h/sample → 6–48h/sample | Massive sequential reads/writes; compression; intermediate files huge |
| Cohort-level analysis / aggregation | 100–100k samples | 0 GPUs → 1×24GB | 64–512 vCPU, 256GB–2TB | 2–32TB | 6–48h → days–weeks | Very I/O \+ RAM heavy; joins/aggregations; heavy metadata outputs |
| PEFT tuning (protein/genomics models) | curated private datasets | 1×80GB → 4×80GB | 32–128 vCPU, 256GB–1TB | 2–16TB | 6–48h → 2–10 days | Streaming \+ checkpoints; artifact versions \+ eval outputs |
| Full training (rare, research) | very large corpora | 8×80GB → 64×80GB (multi-node) | 256–2048 vCPU, 1–16TB (cluster) | 20–500TB (cluster) | 1–4 weeks → months | Constant streaming; huge checkpoints; network \+ storage dominate |

### **Parallelism expectations**

* **Classical bio pipelines** (sample processing) scale by **parallelizing samples**, not NCCL.  
* **Protein/genomics deep learning**:  
  * embeddings/inference scale via **batched GPU workers**  
  * training scales via **DDP**; multi-node shows up in serious research/orgs  
* Mixed workloads are common: **CPU-heavy preprocessing \+ GPU-heavy inference**.

### **Failure tolerance**

* Must be **restartable/idempotent**:  
  * by `sample_id`, `run_id`, `reference_version`, `pipeline_version`  
  * stage-level resume (preprocess → infer → aggregate → export)  
* Training must support **resume-from-checkpoint**; preemption is acceptable only if resume is robust and checkpoints are frequent enough.

**Output artifact:** a workload profile with p50/p90 resource \+ time numbers (table above), refined later with interviews \+ telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **ML**: PyTorch-based stacks (sometimes JAX), mixed precision, accelerator frameworks  
* **Bio file formats & tooling** (common expectations):  
  * FASTA/FASTQ, BAM/CRAM, VCF, GFF/GTF  
  * compression \+ indexing (bgzip/tabix-style patterns)  
* **Pipelines/orchestration**:  
  * workflow engines (Nextflow/Snakemake/WDL/CWL-style)  
  * queues \+ workers for embedding/inference farms  
* **Structure / sequence tooling**:  
  * sequence tokenization \+ chunking  
  * structure inference dependencies (often heavy native libs)

### **CUDA/driver constraints**

* Provide a stable **CUDA 12.x** baseline for modern GPUs.  
* Keep a **compat** image for pinned stacks.  
* Reproducibility matters: pin **image digest \+ model versions \+ reference datasets**.

### **Container vs VM requirements**

* **Containers** are the default (pipelines, reproducible runs).  
* **VMs** show up when:  
  * strict isolation is required (human genomic data, institutional policies)  
  * they need special filesystem tuning for large I/O  
* HPC-ish users may expect **shared filesystems**; you can emulate with object storage \+ caching early.

### **Storage expectations**

* Local NVMe is often critical for:  
  * decompression/indexing caches  
  * intermediate shards  
  * temp scratch for structure workloads  
* Durable object storage for:  
  * raw inputs, long-term artifacts, run manifests, model versions  
* “Bring your own bucket” is common for governance.

### **Networking needs**

* Many customers want **private connectivity** (VPN/peering) to lab storage.  
* Multi-node training (if offered later) wants higher bandwidth and low jitter; for MVP focus on single-node \+ scale-out workers.

**Output artifact:** “reference runtime image set” (base images \+ pinned versions)

* `bio-preproc` (CPU-heavy tools \+ format support)  
* `bio-embed-cu12` (GPU embedding/inference)  
* `bio-structure-cu12` (structure inference stack)  
* `bio-train-cu12` (training/tuning)  
* `workflow-runner` (Nextflow/Snakemake-style runner)  
* `compat-cu11` (fallback)

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* **Human genomics** often counts as highly sensitive personal data (even if “de-identified”).  
* **Protein/proprietary assays** can be core IP for a lab/company.

### **Common requirements you’ll hear**

* **Data residency / region lock**  
* **Isolation**: dedicated nodes for regulated datasets; dedicated network segments  
* **Encryption**: at rest \+ in transit, key management expectations  
* **Audit trails**:  
  * who accessed datasets  
  * who ran which pipeline version  
  * who downloaded results/models  
* **Retention rules**:  
  * strict control of intermediates (often huge and sensitive)  
  * defined retention windows for raw \+ derived data

### **Logging rules**

* Don’t log raw sequences or patient identifiers by default.  
* Store metadata only (run IDs, hashes of manifests, sizes, timings, error codes), with controlled access.

**Output artifact:** security/compliance checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* **Throughput** and predictable completion time (labs plan around runs)  
* **I/O performance** (this is the \#1 hidden killer: decompression \+ indexing \+ intermediates)  
* **Reproducibility**:  
  * same inputs \+ same reference version \+ same pipeline version → same outputs  
* **Cost visibility**:  
  * large batch runs can burn budget quickly  
* **Reliability**:  
  * no lost artifacts  
  * resumable pipelines with clear progress

### **SLO starting points (priors)**

* Job start/provisioning: p90 \< 10–20 min (depends on image \+ data staging)  
* Batch run completion reliability: \>99% with stage retries \+ resume  
* Artifact durability: “final outputs never disappear”  
* Observability: clear per-stage progress (preprocess vs infer vs write/aggregate)

### **Support playbook themes**

* “It’s slow” (I/O bottleneck, decompression, bad caching, undersized scratch)  
* “Disk filled up” (intermediates explode; retention missing)  
* “Different results than last time” (version drift; reference mismatch)  
* “Pipeline failed mid-way” (need stage-level resume \+ idempotency)  
* “Data access denied” (private networking/IAM complexity)

### **Cost sensitivity & billing preference**

* Research teams: want cheap batch, accept spot/preemptible if resumable  
* Regulated/enterprise: pay premium for dedicated \+ audited \+ predictable  
* Common billing split: compute (GPU/CPU) \+ storage (GB-month) \+ data egress

**Output artifact:** per-niche SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* **Pipeline-first**: run a workflow (Nextflow/Snakemake style) that pulls from a bucket, processes, and pushes artifacts back  
* **Embedding farms**: enqueue batches of sequences → GPU workers write embeddings → downstream indexing/analysis  
* **Lab triggers**: “new samples arrived” → run pipeline → notify \+ store results

### **Auth patterns**

* Org/project RBAC \+ service accounts  
* Strict audit for regulated datasets  
* Enterprise: SSO later, but audit logs and least privilege early

### **Artifact flow**

* Inputs: raw reads / sequences / references / manifests  
* Intermediates: huge scratch products (should be auto-expired by default)  
* Outputs: embeddings, annotations, reports, aggregate tables, model artifacts, run manifests  
* Destinations: customer bucket/storage \+ optional managed registry for models

### **Golden flows (end-to-end)**

1. **Embedding \+ indexing factory**  
   * Upload sequences → shard/batch → GPU embedding workers → write embedding shards → build index/feature store → export \+ report  
2. **Genomics pipeline run (private data)**  
   * Trigger on new samples → stage inputs → preprocess \+ analysis → produce VCF/metrics \+ run manifest → push outputs to private storage → audit log  
3. **Private tuning \+ evaluation**  
   * Curate dataset → run PEFT tuning → eval suite \+ regression checks → version model \+ promote to batch inference workers

