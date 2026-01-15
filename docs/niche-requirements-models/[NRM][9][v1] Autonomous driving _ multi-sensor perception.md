# **Autonomous driving / multi-sensor perception**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: small scene subsets, pipeline sanity, sensor decode checks  
* **Dataset preprocessing**: decode/convert logs, synchronize sensors, generate labels, build shards  
* **Training (perception)**: detection/segmentation/tracking (camera \+ lidar \+ radar)  
* **Training (prediction/planning)**: trajectory prediction, behavior models (less common than perception)  
* **Simulation & replay**: closed-loop evaluation, scenario replay, sensor simulation  
* **Batch inference**: run models over large log corpora to generate predictions/metrics  
* **Online inference (edge-like)**: low-latency perception stack for realtime testing  
* **Eval/regression**: benchmark suites, scenario-based metrics, safety gating

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (pipeline sanity) | 1–50 scenes | 0–1×24GB → 1×48GB | 32–64 vCPU, 128–512GB | 1–4TB | 1–6h → 1–3 days | Heavy decode \+ sync; frequent reruns; medium artifacts |
| Preprocessing (decode \+ shard \+ sync) | 10–10,000 hours logs | 0 GPUs → 1×24GB | 64–512 vCPU, 256GB–2TB | 4–64TB | 6–48h → days–weeks | Massive sequential reads/writes; compression; shard creation |
| Training (camera-only perception) | 10k–10M frames | 1×48GB → 8×80GB | 64–512 vCPU, 256GB–2TB | 2–32TB | 2–7 days → 2–8 weeks | High-throughput reads; augmentation; big checkpoints |
| Training (multi-sensor: cam+lidar+radar) | 10k–10M synced frames | 2×80GB → 16×80GB (multi-node) | 128–1024 vCPU, 0.5–8TB | 8–200TB (cluster) | 1–4 weeks → months | Bandwidth \+ sync overhead; storage \+ network dominate |
| Simulation / replay (closed-loop) | 100–100k scenarios | 1×24GB → 4×48GB | 64–512 vCPU, 256GB–2TB | 2–32TB | 6–48h → 1–4 weeks | Heavy CPU \+ rendering; lots of logs/video; reproducibility critical |
| Batch inference over logs | 100–100k hours | 1×24GB → 8×80GB | 64–512 vCPU, 256GB–2TB | 4–64TB | \~0.2×–2× realtime → 2×–10× realtime | Read-heavy; write predictions/metrics; often bottlenecked by decode |
| Online inference (realtime) | 10–1k vehicles/agents | 1×24GB → 2×48GB | 16–128 vCPU, 64–512GB | 0.5–4TB | continuous | Tail latency sensitive; steady sensor stream; strict jitter requirements |
| Eval/regression suites | 100–50k scenes | 0–1×24GB → 1×48GB | 32–256 vCPU, 128GB–1TB | 1–16TB | 6–48h → days | Produces reports; repeat runs; heavy reads \+ moderate writes |

### **Parallelism expectations**

* **Preprocessing and batch inference** scale by **sharding logs** and running many workers.  
* **Training** scales via **DDP**; multi-node becomes common in serious multi-sensor setups.  
* **Simulation** is CPU-heavy; GPU may be needed for rendering or model-in-the-loop.

### **Failure tolerance**

* Must be restartable/idempotent:  
  * by log segment / scene ID / shard ID  
  * stage-level resume (decode → sync → train/infer → aggregate → export)  
* Training must support resume-from-checkpoint (runs are long/expensive).  
* Determinism matters: pinned versions, seeds, exact dataset manifests.

**Output artifact:** a workload profile with p50/p90 resource \+ time numbers (table above), refined later via interviews \+ telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **Perception ML**: PyTorch-based stacks; heavy custom code; mixed precision; large dataloaders  
* **Sensor tooling**: multi-sensor synchronization, calibration handling, time alignment  
* **Data pipelines**: shard builders, feature stores, dataset manifests, label ingestion  
* **Simulation/replay**: scenario engines, rendering pipelines, log replay tooling  
* **Distributed training**: DDP/FSDP/DeepSpeed appears in big runs

### **CUDA/driver constraints**

* Stable CUDA 12.x baseline for modern GPUs.  
* Compat images for pinned dependencies.  
* Version pinning is critical for reproducibility.

### **Container vs VM requirements**

* Containers are default.  
* VMs show up when:  
  * they need specialized drivers (simulation/render)  
  * stricter isolation or custom kernel settings are required  
* Multi-disk layouts are common: huge scratch \+ durable outputs.

### **Storage expectations**

* Local NVMe is essential due to:  
  * massive datasets/logs  
  * shard caches  
  * intermediate artifacts (predictions, videos)  
* Durable object storage for:  
  * raw logs, datasets, checkpoints, reports, manifests  
* Data locality is a major cost/latency factor.

### **Networking needs**

* Baseline: 25GbE is common in serious setups; higher for heavy ingest.  
* Distributed training at scale benefits from 100GbE-class networking.  
* Private connectivity often required (internal log stores).

**Output artifact:** reference runtime image set (base images \+ pinned versions)

* `av-preproc` (decode \+ sync \+ shard tools)  
* `av-train-perception-cu12`  
* `av-train-multisensor-cu12`  
* `av-sim-replay` (scenario \+ rendering)  
* `av-batch-infer-cu12`  
* `av-eval`  
* `compat-cu11`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Proprietary driving logs are core IP.  
* May include **faces, plates, location data** (privacy-sensitive).  
* Strong contractual controls are common.

### **Common requirements**

* Region lock / residency  
* Isolation (often dedicated nodes/projects)  
* Encryption at rest \+ TLS in transit  
* Strict RBAC \+ audit trails  
* Retention policies (raw logs vs derived artifacts; deletion controls)

### **Logging rules**

* Don’t store raw sensor frames in general logs by default.  
* Store metadata: shard IDs, timings, versions, aggregate metrics, error codes.  
* Provide controlled access paths for raw log review.

**Output artifact:** security/compliance checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* **I/O throughput** and data pipeline stability  
* **Training throughput** (samples/sec) and scaling efficiency  
* **Reproducibility**: same manifest \+ same code \= same metrics  
* **Reliability**: no dropped shards, no partial silent failures  
* **Cost visibility**: data \+ compute are huge

### **SLO starting points (priors)**

* Provision/start: p90 \< 10–20 min (depends on staging)  
* Batch jobs: \>99% completion with stage retries \+ resume  
* Training: checkpoint durability and resume correctness  
* Observability: clear bottleneck indicators (decode vs GPU vs disk)

### **Support playbook themes**

* “Preprocessing is slow” (decode bottleneck, compression, storage saturation)  
* “Training is slow” (dataloader issues, poor sharding, network/NCCL problems)  
* “Results changed” (version drift in dataset manifests or code)  
* “Disk exploded” (intermediate artifacts, videos, prediction dumps)  
* “Simulation fails” (driver/rendering issues, nondeterminism)

### **Pricing sensitivity & billing preference**

* Big teams prefer reservations/committed spend.  
* Many accept spot for preprocessing/batch inference if resumable.  
* Training often requires guaranteed capacity windows.

**Output artifact:** per-niche SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Pipeline-driven: ingest logs → preprocess/shard → train → infer → evaluate → promote  
* Continuous regression: nightly/weekly benchmark runs  
* Simulation loops for scenario testing

### **Auth patterns**

* Org/project RBAC \+ service accounts  
* Strict audit logs for data access

### **Artifact flow**

* Inputs: raw logs \+ calibration \+ labels \+ manifests  
* Intermediates: shards, caches, sim renders, predictions  
* Outputs: checkpoints, metrics dashboards, evaluation reports, run manifests  
* Destinations: customer storage \+ optional managed registry

### **Golden flows (end-to-end)**

1. **Log ingestion → shard build**  
   * Pull logs → decode/sync → build shards → validate → publish manifest  
2. **Multi-sensor training run**  
   * Load manifest → distributed training → checkpoint \+ eval → regression report → version \+ promote  
3. **Batch inference \+ scenario eval**  
   * Run inference over log corpora → aggregate metrics → scenario-based report → gate releases

