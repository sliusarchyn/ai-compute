# Video generation / video understanding (ad tech, media, VFX)

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: pipeline sanity (prompt templates, settings, codec profiles, asset staging)  
* **Interactive video generation**: short clips, fast iteration loops  
* **Batch video generation**: many variants for campaigns/libraries  
* **Video-to-video / editing-like**: restyle, extend, multi-pass workflows  
* **Upscaling / frame interpolation / super-resolution**  
* **Post-processing**: transcode, packaging, compositing steps  
* **Video understanding**: captioning, tagging, shot/scene detection, embeddings, moderation  
* **Training/tuning**: LoRA/adapters for video models; multimodal fine-tunes  
* **Eval/regression**: fixed prompt suite \+ quality/latency checks

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (pipeline sanity) | 1–50 short clips | 1×24GB → 1×48GB | 16–32 vCPU, 64–256GB | 0.5–2TB | 10–60 min → 2–8h | Moderate reads, lots of intermediate frame writes, repeated re-runs |
| Interactive video generation | 1–20 clips/session (2–8s) | 1×48GB → 1×80GB | 16–64 vCPU, 64–512GB | 1–4TB | 2–10 min/clip → 10–60 min/clip | Bursty asset reads; heavy scratch writes; final encode output |
| Batch video generation (variants) | 50–10,000 clips/job | 1×80GB → 8×80GB | 32–256 vCPU, 128GB–1TB | 2–16TB | 2–12h → 1–7 days | Continuous writes \+ uploads; throughput and queueing dominate |
| Video-to-video / editing-like (multi-pass) | 1–200 clips/job | 1×80GB → 4×80GB | 32–128 vCPU, 128–512GB | 2–16TB | 10–60 min/clip → 1–6h/clip | Multi-stage intermediates; restart cost high without staging |
| Upscale / interpolation / SR | 10–2,000 clips/job | 1×24GB → 1×48GB | 16–128 vCPU, 64–512GB | 2–32TB | 30 min–6h → 6–72h | Extremely disk I/O heavy (decode → frames → encode) |
| Post-processing (transcode/packaging) | 100–50,000 assets/day | 0 GPUs → 1×24GB | 32–256 vCPU, 64GB–1TB | 1–16TB | seconds–minutes/asset → minutes–hours/asset | Streaming I/O; codecs \+ CPU often the bottleneck |
| Video understanding (caption/tag/embeddings) | 1k–10M videos/day | 1×24GB → 4×48GB | 32–256 vCPU, 64GB–1TB | 1–16TB | 0.2×–2× realtime → 2×–10× realtime | Decode-heavy reads; small metadata writes; batching critical |
| Training/tuning (video models) | 10k–10M clips/frames | 4×80GB → 32×80GB | 128–1024 vCPU, 1–8TB (cluster) | 8–200TB (cluster) | 2–7 days → 2–8 weeks | Constant dataset streaming \+ big checkpoints; needs fast storage |

### **Parallelism expectations**

* **Inference/processing**: mostly parallel across clips/jobs (queue \+ worker pools), not NCCL-heavy.  
* **Training**: DDP common; multi-node shows up at the high end; interconnect quality starts to matter.

### **Failure tolerance**

* Needs to be restartable at multiple layers:  
  * **job-level idempotency** (job\_id, variant\_id)  
  * **stage-level resume** (generate → upscale → encode → upload)  
  * **segment/frame-range resume** for long clips  
* **Spot/preemptible** is fine for batch if you persist stage outputs and can resume from last completed stage/segment.

**Output artifact:** a “workload profile” with p50/p90 resource \+ time numbers (table above), later refined from interviews \+ telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* Core: PyTorch-based generation \+ understanding models; diffusion-style video stacks; optional TensorRT/Triton for speed.  
* Video I/O: FFmpeg (decode/encode); codec profiles matter (H.264/H.265/ProRes/AV1 depending on clients).  
* Orchestration: multi-stage pipelines (generate → post → upload), queues, retries, backpressure.  
* Understanding: batched inference plus decoding workers; often split **CPU decode** and **GPU model** workers.

### **CUDA/driver constraints**

* Provide a stable default **CUDA 12.x** image for modern GPUs.  
* Provide a **compat** image for older pins.  
* Pin versions (image digest \+ model \+ codec toolchain) for reproducibility.

### **Container vs VM requirements**

* **Containers** are default for workers and pipelines.  
* **VM/workstation** requests show up when artists need a workstation-like environment or special plugins.  
* Practical disk layout: boot \+ **scratch NVMe** \+ durable outputs (or object storage).

### **Storage expectations**

* Local NVMe scratch is essential (frames/intermediates explode in size).  
* Durable object storage for source assets, finals, manifests, checkpoints.  
* Shared storage can help stage handoffs, but can bottleneck if undersized.

### **Networking needs**

* Baseline: **10–25GbE** per node is a sane start for heavy ingest/output.  
* For serious distributed training: **100GbE-class** networking matters.  
* Private connectivity is common (internal asset stores, media pipelines).

**Output artifact:** “reference runtime image set” (base images \+ pinned versions)

* `video-gen-cu12`  
* `video-v2v-cu12`  
* `video-upscale-cu12`  
* `video-understand-cu12`  
* `video-encode` (FFmpeg/codec CPU stack)  
* `compat-cu11`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Unreleased creative assets, client IP, ad campaigns, film/TV footage.  
* Strong contractual controls on access.

### **Common requirements**

* Region lock / residency for asset processing.  
* Strong isolation (often dedicated nodes for premium clients).  
* Encryption at rest \+ TLS in transit.  
* Strict RBAC \+ audit trails (who rendered/downloaded/viewed).  
* Retention rules: auto-delete intermediates; controlled retention for finals.

### **Logging rules**

* No raw frames/prompts in logs by default.  
* Store metadata only: asset IDs/hashes, pipeline version, model version, timings, costs, error codes.  
* Customer-controlled export to their storage/SIEM.

**Output artifact:** security/compliance checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* Predictable throughput (clips/day, fps throughput).  
* Short queue times for interactive work.  
* Pipeline reliability (stage failures visible \+ resumable).  
* Cost visibility.  
* Reproducibility (version-pinned pipelines).

### **SLO starting points (priors)**

* Interactive job start ready: **p90 \< 10–20 min** (including staging/warmup).  
* Batch stage completion reliability: **\>99% per stage** (retries \+ resume).  
* Final outputs durability: “final renders never disappear.”  
* Observability: clear per-stage progress \+ bottleneck hints (decode vs model vs disk).

### **Support playbook themes**

* Slow jobs (decode bottleneck, storage saturated, cache misses, noisy neighbor).  
* Quality drift (version drift; missing pinning).  
* Stuck pipelines (retry loops, backpressure, dead letters).  
* Storage explosion (intermediates; missing retention).  
* Encoding failures (codec config, corrupted inputs).

### **Cost sensitivity & billing preference**

* Interactive teams pay for reserved capacity/low queue time.  
* Batch prefers cheaper compute; accepts spot if resumable.  
* Training pays for high-end clusters \+ guaranteed windows.

**Output artifact:** per-niche SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Ad tech: template-driven variant generation (many outputs, fast iteration).  
* Media/VFX: shot-based workflows, approvals, render-queue mental model.  
* Understanding: continuous pipelines for metadata/search/analytics.

### **Auth patterns**

* Org/team RBAC, per-project permissions.  
* Service accounts for automation.  
* Audit logs often required.

### **Artifact flow**

* Inputs: source videos, reference images, prompts/configs, templates.  
* Intermediates: frames, caches, multi-pass outputs (should auto-expire unless requested).  
* Outputs: final encoded videos \+ thumbnails \+ manifests (pipeline version, model hash, settings).  
* Destinations: customer bucket/storage, optional managed library.

### **Golden flows (end-to-end)**

1. **Ad variants batch production**  
   Upload template \+ prompt list \+ brand assets → queue thousands of jobs → staged outputs → auto-upload finals \+ manifest → notify completion  
2. **VFX shot iteration (interactive)**  
   Start workspace/workstation → pull shot assets → iterative video-to-video passes → review outputs → export finals with audit trail  
3. **Video understanding pipeline**  
   Ingest from storage → decode \+ batched inference → write metadata/index → monitor freshness/latency → export to downstream systems

