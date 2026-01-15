# LLM fine-tuning for private corp data

**Niche Requirements Model v1** for **LLM fine-tuning on private corporate data**.

---

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: “Does my dataset \+ prompt format work?” small runs, fast iteration.  
* **Fine-tune ([PEFT](https://docs.google.com/document/d/1c8JyyS4OWWJ2yY2Sa3SbR-zewPQ1cJcNjrv0p4off10/edit?tab=t.0): [LoRA](https://docs.google.com/document/d/1c8JyyS4OWWJ2yY2Sa3SbR-zewPQ1cJcNjrv0p4off10/edit?tab=t.0)/[QLoRA](https://docs.google.com/document/d/1c8JyyS4OWWJ2yY2Sa3SbR-zewPQ1cJcNjrv0p4off10/edit?tab=t.0))**: most common for private corp data.  
* **Preference tuning** ([DPO/ORPO/KTO-ish](https://docs.google.com/document/d/1c8JyyS4OWWJ2yY2Sa3SbR-zewPQ1cJcNjrv0p4off10/edit?tab=t.0)): common when they want style/behavior changes.  
* **Full training / full fine-tune**: rarer (expensive, heavy compliance teams, or model owners).  
* **Batch inference / eval**: offline scoring, regression tests, benchmark suites.  
* **Online inference**: serving the tuned model (often separate infra from training).

### **Workload profile (p50 / p90 “shape”)**

These assume **7B–70B** class models and common corp datasets (docs, tickets, policies, code, chats).

| Job type | Typical model sizes | GPU config (p50 → p90) | CPU/RAM (rule-of-thumb) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (format/overfit) | 7B–13B | **1×48GB → 1×80GB** | 8–16 vCPU, **64–128GB RAM** | 200–500GB | **15–60 min → 2–6h** | Small dataset reads; frequent log streaming; minimal checkpoints |
| PEFT LoRA/QLoRA SFT | 7B–13B | **1×80GB → 2×80GB** | 16–32 vCPU, **128–256GB RAM** | 0.5–2TB | **2–8h → 12–36h** | Dataset reads steady; checkpoint adapter every 30–120min; artifacts small (0.1–3GB) |
| PEFT LoRA/QLoRA SFT | 34B | **2×80GB → 4×80GB** | 32–64 vCPU, **256–512GB RAM** | 1–4TB | **8–24h → 2–4 days** | More frequent checkpointing; eval passes add bursty reads |
| Preference tuning (DPO/ORPO) | 7B–34B | **1–2×80GB → 4×80GB** | \+25–50% CPU/RAM vs SFT | \+0.5–2TB | **4–16h → 1–4 days** | Often heavier eval/compare loops; more artifacts (reports) |
| Full fine-tune | 7B–13B | **4×80GB → 8×80GB** | 64–128 vCPU, **512GB–1.5TB RAM** | 2–8TB | **1–3 days → 1–2 weeks** | **Huge checkpoints** if saving full weights (10s–100s GB) |
| Batch inference / eval | 7B–70B | **1×48/80GB → 2×80GB** | 8–32 vCPU, 64–256GB RAM | 0.5–2TB | **10 min–6h → 1–2 days** | Large read bursts; outputs (JSONL) can be GBs |
| Online inference (prod) | 7B–70B | **1×24/48/80GB** | 8–32 vCPU, 32–256GB RAM | 100–500GB | continuous | Steady request traffic; needs predictable latency |

**Parallelism expectations**

* **Single GPU** dominates dev/test and many LoRA tunes.  
* **Multi-GPU single node (DDP)** is common for 34B+ and faster turnaround.  
* **Multi-node** is *rare* in SMB corp fine-tuning; becomes relevant for enterprise/full fine-tune.  
  If you do multi-node: you’ll need **NCCL-friendly networking** and careful topology.

**Network needs (by mode)**

* Single-node training: **10–25GbE/node** is usually fine (biggest bottleneck is storage).  
* Multi-node training: realistically **100GbE \+ RDMA** (or you’ll get support nightmares).

**Failure tolerance**

* Corporate teams expect **resume-from-checkpoint** to “just work.”  
* Preemption:  
  * acceptable for **dev/test** and sometimes **training** if auto-resume \+ frequent checkpoints  
  * *painful* for full fine-tune if checkpointing is huge/slow

**Output artifact for this section**

* A “workload profile” like the table above, but you’ll later replace the priors with real p50/p90 from telemetry \+ interviews.

---

## **2\) Stack \+ environment constraints**

### **Most common training stack (what you should assume)**

* **PyTorch \+ Hugging Face**: `transformers`, `datasets`, `accelerate`  
* PEFT: `peft`, `bitsandbytes` (QLoRA)  
* Preference tuning: `trl` (+ their own reward/ranking logic)  
* Scaling: `torchrun` (DDP), sometimes `deepspeed` or `FSDP`  
* Inference/eval: often `vllm` or `text-generation-inference` (optional early)

### **CUDA/driver constraints (practical platform rule)**

* Support **CUDA 12.x** as default for new GPUs.  
* Keep a **fallback CUDA 11.8** image for older stacks/custom code.  
* Pin images so customers get reproducibility.

### **Container vs VM**

* **Containers are the default** for fine-tuning.  
* VMs show up when they need:  
  * stricter isolation/compliance  
  * custom kernel modules / unusual drivers  
  * corporate IT wants “VM mental model”  
* Avoid requiring privileged containers unless you truly need IB/RDMA features.

### **Storage expectations**

* **Local NVMe scratch** for:  
  * dataset caching (tokenized shards)  
  * checkpoints (fast writes)  
* **Object storage** for:  
  * long-term artifact retention  
  * sharing across runs/teams  
* Customers like “bring your own bucket” (S3/GCS) for compliance.

### **Networking needs**

* Common asks:  
  * **no public inbound** (training jobs shouldn’t be internet-exposed)  
  * **restricted outbound** (allowlist domains, or “no internet” mode)  
  * private connectivity options later: VPN / VPC peering

**Output artifact**

* “Reference runtime image set” (base images you provide \+ pinned libs). Example set:  
  1. `train-hf-cu12` (PyTorch \+ Transformers \+ Accelerate \+ PEFT \+ TRL)  
  2. `train-deepspeed-cu12` (adds DeepSpeed/FSDP tooling)  
  3. `infer-vllm-cu12` (serving \+ eval)  
  4. `cu11-compat` (older CUDA fallback)

---

## **3\) Data governance \+ risk**

### **What “private corp data” usually means**

* IP-sensitive docs, tickets, internal chats, code, customer conversations  
* Often includes **PII** (names, emails, phone numbers) even if not “medical”

### **Common requirements**

* **Region lock** (data residency)  
* **Dedicated tenancy**:  
  * dedicated node (no other tenants on same host)  
  * dedicated network segment (at least per-tenant VLAN/VPC)  
* **Encryption**:  
  * at rest (volumes \+ object storage)  
  * in transit (TLS everywhere)  
* **Audit**:  
  * who launched runs, accessed artifacts, downloaded models

### **Logging rules (super important)**

Customers often want:

* logs retained **short** (e.g., 7–30 days) or export-only to their SIEM  
* **no dataset contents** in logs  
* redact prompts/outputs by default (or configurable)

**Output artifact**

* Security/compliance checklist \+ tenancy requirements:  
  * data residency, isolation level, encryption, audit logs, retention, access controls, export controls.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What they care about most**

* **Queue time** (time to get GPUs)  
* **Start time** (image pull \+ environment ready)  
* **Throughput** (tokens/sec, steps/sec)  
* **OOM frequency** (bad defaults → lots of support)  
* **Run reliability** (resume works, artifacts not lost)

### **Baseline SLO targets (good starting point)**

* Dev/Test pods:  
  * start within **5–10 min p90**  
* Training runs:  
  * provisioning within **10–20 min p90**  
  * checkpoint upload success **\>99%**  
* Inference (if you offer it):  
  * basic availability target **99.5%+** (higher later)

### **Support expectations**

You’ll get these tickets constantly:

* “OOM, what do I change?”  
* “Slow run on this GPU vs last time”  
* “Stuck downloading dataset / pulling image”  
* “Training diverged / loss exploded”  
  So you need:  
* sane defaults (batch size, grad accum)  
* clear run diagnostics  
* quick “recommended config” templates

### **Cost sensitivity**

* Dev/test: wants cheap, may accept preemptible/spot  
* Training: wants predictable completion; may pay premium for “non-preemptible”  
* Enterprise: likes reservations / committed spend \+ invoices

**Output artifact**

* Per-niche SLOs \+ support playbook \+ pricing sensitivity notes.

---

## **5\) User workflow & integrations**

### **How they launch work today (most common)**

* Notebooks (Jupyter/Lab) for iteration  
* CLI scripts (HF \+ accelerate \+ torchrun)  
* CI pipelines for eval/regression  
* W\&B for experiment tracking (or MLflow in enterprises)  
* Hugging Face model tooling (but not always public hub)

### **Auth patterns**

* Org/project accounts  
* API keys  
* RBAC: “can run jobs” vs “can access artifacts/models”  
* Team sharing of datasets/models inside org

### **Artifact flow**

* Dataset: upload, or connect to their bucket  
* Outputs:  
  * LoRA adapters  
  * merged model weights (optional)  
  * eval reports (JSON/HTML)  
  * training logs/metrics  
* Destination: their bucket or your managed registry

**Golden flows (end-to-end)**

1. **Secure LoRA tune**  
   * Upload dataset → launch PEFT tune from template → stream metrics → export adapter → store in private registry/bucket  
2. **Bring-your-own-bucket enterprise flow**  
   * Connect S3/GCS → run training with no public internet → artifacts written only to their bucket → audit log maintained  
3. **Eval \+ promote**  
   * Run eval suite (batch inference) → compare vs baseline → tag model version → handoff to inference deployment (or download)

