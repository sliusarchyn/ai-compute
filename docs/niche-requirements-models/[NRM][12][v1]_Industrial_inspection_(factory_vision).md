# **Industrial inspection (factory vision)**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: camera calibration checks, small dataset labeling, quick model tests  
* **Data ingest \+ labeling prep**: collect images/video from lines, sync metadata, create labeling tasks  
* **Training / tuning**: defect detection (classification/detection/segmentation), domain adaptation per line/product  
* **Batch inference (offline QA)**: run models over stored batches for audit and analysis  
* **Online inference (on-line inspection)**: realtime inspection on production line (latency \+ jitter sensitive)  
* **Anomaly detection (unsupervised)**: “novel defect” detection, often per product/line  
* **Edge deployment packaging**: export/quantize, build runtimes, regression tests  
* **Eval/regression**: precision/recall, false reject/false accept, drift monitoring

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (calibration \+ small runs) | 1–50k images | 0–1×24GB → 1×48GB | 16–32 vCPU, 64–256GB | 0.5–2TB | 2–12h → 1–5 days | Many small image reads; frequent reruns; lots of debug artifacts |
| Data ingest \+ labeling prep | 10GB–10TB/day | 0 GPUs → 1×24GB | 32–256 vCPU, 128GB–2TB | 1–16TB | continuous / daily batches | Write-heavy; image/video \+ metadata; task exports |
| Training/tuning (per product/line) | 100k–100M images | 1×48GB → 8×80GB | 64–512 vCPU, 256GB–2TB | 2–32TB | 1–7 days → 2–8 weeks | High-throughput reads; augmentation; big checkpoints |
| Unsupervised anomaly training | 10k–10M images | 1×24GB → 2×48GB | 32–256 vCPU, 128GB–1TB | 1–16TB | 6–48h → 1–14 days | Heavy feature extraction; writes embeddings/models |
| Batch inference (offline QA) | 100k–1B images/job | 1×24GB → 8×80GB | 32–512 vCPU, 128GB–2TB | 2–64TB | 0.1×–1× realtime → 1×–5× realtime | Read-heavy; writes predictions \+ heatmaps \+ reports |
| Online inference (production line) | 10–10k FPS per line/site | 0–1×24GB → 2×48GB | 16–256 vCPU, 64GB–1TB | 200GB–4TB | continuous | Tail latency/jitter critical; steady streams; small outputs |
| Edge packaging \+ quantization | per model/version | 0–1×24GB → 1×48GB | 16–64 vCPU, 64–256GB | 200GB–2TB | 1–6h → 6–48h | Reads checkpoints; writes optimized engines; regression artifacts |
| Eval/regression suites | 10k–10M images | 0–1×24GB → 1×48GB | 16–128 vCPU, 64–512GB | 0.5–8TB | 2–12h → 1–7 days | Produces reports; repeat runs; heavy reads |

### **Parallelism expectations**

* Training and batch inference scale via DDP and worker pools.  
* Online inference scales by **lines/cameras**, often requiring predictable per-stream performance.  
* Pre/post-processing and image decode can be CPU-bound; plan for strong CPU \+ fast storage.

### **Failure tolerance**

* Online inference: must be highly available; graceful degradation (fallback to last good model) is common.  
* Batch jobs: must be idempotent by `batch_id` / `camera_id` / `timestamp_range`.  
* Spot/preemptible is fine for offline training/batch inference if resumable; not for production line inference.

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined later by telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **Vision ML**: PyTorch; export to ONNX/TensorRT for edge/low latency  
* **Data tooling**: image/video ingestion, metadata stores, labeling integration  
* **Serving**: low-latency inference services (often gRPC), camera stream ingestion  
* **Optimization**: quantization, batch vs streaming pipelines, GPU sharing where safe

### **CUDA/driver constraints**

* Stable CUDA 12.x baseline for modern GPUs.  
* Compat images for pinned stacks.  
* For edge, you may need specific runtimes; keep “build” images separate from “serve” images.

### **Container vs VM requirements**

* Containers are default for training/inference services.  
* Some customers want VMs or dedicated nodes for strict isolation and predictable latency.

### **Storage expectations**

* Local NVMe for hot datasets and caches; huge retention for image archives.  
* Durable object storage for raw data, labeled datasets, models, reports.  
* Lifecycle policies matter (raw images grow fast).

### **Networking needs**

* In-factory networks may be constrained; often requires private ingress/VPN.  
* Online inference may need local processing (edge), but your platform often handles training \+ centralized QA analytics.

**Output artifact:** reference runtime image set

* `vision-ingest-labelprep`  
* `vision-train-cu12`  
* `vision-anomaly-cu12`  
* `vision-batch-infer-cu12`  
* `vision-online-serve-cu12`  
* `vision-export-onnx-trt`  
* `compat`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Usually IP-sensitive (products, processes), not typically regulated like PHI.  
* Some images may contain people (privacy considerations).

### **Common requirements**

* Isolation for proprietary manufacturing data  
* Encryption at rest \+ TLS in transit  
* Access control per site/line/product  
* Retention and deletion policies (often strict for IP)

### **Logging rules**

* Don’t store raw frames in logs by default.  
* Store metadata: camera\_id, model\_version, timestamps, defect scores, error codes.

**Output artifact:** security checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* False rejects/false accepts (direct cost impact)  
* Latency \+ jitter for on-line inspection  
* Uptime and predictable performance  
* Model drift (lighting, camera changes, new materials)  
* Traceability: why was a part rejected?

### **SLO starting points (priors)**

* Online inference availability: customer-specific, but high  
* Latency p95 targets per line/site  
* Batch analytics completion reliability: \>99%  
* Artifact durability: reports and model versions never disappear

### **Support playbook themes**

* “Model suddenly worse” (lighting drift, camera moved)  
* “Latency spikes” (resource contention, decode bottlenecks)  
* “Too many false rejects” (thresholds, calibration)  
* “Data pipeline broke” (camera feed changes, metadata mismatch)  
* “Can’t explain decision” (need overlays/heatmaps \+ audit trails)

### **Pricing sensitivity & billing preference**

* Training \+ batch analytics often on-demand/reserved  
* Production inference often reserved (predictable latency)  
* Enterprises prefer predictable monthly cost \+ per-site pricing

**Output artifact:** SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Collect data → label → train → deploy to line → monitor drift → retrain  
* Nightly/weekly batch analytics for QA and audits  
* Incident investigations use stored images \+ overlays

### **Auth patterns**

* Org/team RBAC, per-site permissions  
* Service accounts for ingestion from factory networks

### **Artifact flow**

* Inputs: images/video streams \+ metadata \+ labels  
* Outputs: model versions, defect predictions, overlays/heatmaps, QA reports, run manifests  
* Destinations: customer storage \+ optional managed model registry

### **Golden flows (end-to-end)**

1. **Train → deploy on line**  
   * Ingest \+ label → train → evaluate → export engine → deploy → monitor drift \+ false reject/accept  
2. **Offline QA audit batch**  
   * Run inference over stored batches → generate reports \+ overlays → export to QA systems  
3. **Drift-driven retrain**  
   * Detect drift (lighting/material) → collect new labeled set → quick fine-tune → regression suite → rollout with rollback path

