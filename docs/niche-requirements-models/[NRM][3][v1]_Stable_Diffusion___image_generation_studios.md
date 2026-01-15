# Stable Diffusion / image generation studios Niche Requirements Model

## **1\) Workload anatomy (the “physics”)**

### **Job types you must support**

* **Interactive generation** (txt2img/img2img)  
* **Batch rendering** (campaigns/catalogs)  
* **Inpainting/outpainting \+ ControlNet-heavy** workflows  
* **Upscaling / post-processing**  
* **LoRA fine-tuning** (style/character/product)  
* **Heavier fine-tuning** (DreamBooth-ish / higher-res / longer runs)  
* **Eval/regression** (same prompt suite, compare versions)

### **Workload profile (p50 / p90 shape)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Interactive txt2img/img2img | 1–50 images/session | **1×24GB → 1×48GB** | 8–16 vCPU, **32–96GB** | 200GB–1TB | **5–20s/img → 20–90s/img** | Reads model(s) into VRAM; frequent small image writes; lots of UI logs |
| Interactive “heavy” (ControlNet, refiner, hi-res fix) | 1–30 images/session | **1×48GB → 1×80GB** | 16–32 vCPU, **64–128GB** | 500GB–2TB | **15–60s/img → 1–5 min/img** | More models loaded; higher VRAM spikes; higher scratch usage |
| Batch rendering (campaign) | 200–50,000 images/job | **1×24GB → 4×48GB** | 16–64 vCPU, **64–256GB** | 1–4TB | **30 min–6h → 6h–2 days** | Massive output writes; periodic uploads; stable throughput matters |
| Upscaling / post-processing | 50–10,000 images/job | **1×24GB → 1×48GB** | 8–32 vCPU, **32–128GB** | 500GB–4TB | **10–60 min → 6–24h** | Read+write heavy; can bottleneck on disk; sometimes CPU-bound |
| LoRA training (common studio) | 50–5,000 images dataset | **1×24GB → 1×48GB** | 16–32 vCPU, **64–256GB** | 0.5–2TB | **30 min–6h → 6–48h** | Dataset reads \+ caching; checkpoints/adapters written periodically; artifacts usually small–medium |
| Heavier fine-tuning (higher-res / longer) | larger datasets / higher res | **1×48GB → 2×80GB** | 32–64 vCPU, **128–512GB** | 1–4TB | **2–12h → 1–5 days** | More checkpointing; higher disk use; resume becomes critical |
| Eval/regression runs | 100–5,000 prompts | **1×24GB → 1×48GB** | 8–16 vCPU, **32–96GB** | 200GB–1TB | **10–60 min → 6–24h** | Repeatable prompt suite; needs strict version pinning; outputs \+ manifests |

### **Parallelism**

* Typical: **single GPU per “workspace”**, concurrency handled via **queue** or multiple workspaces.  
* Multi-GPU is mostly for **throughput** (more jobs parallel), not for NCCL-heavy distributed training.

### **Failure tolerance**

* Inference: failures are annoying but cheap → must preserve **workflow \+ prompt \+ seed** for reruns.  
* Training: must support **resume** (checkpoint/last state), otherwise users will hate spot/preemptible.  
* Spot/preemptible: acceptable for **batch renders** if you persist outputs continuously and can restart.

**Output artifact:** this workload table \+ (later) your real p50/p90 from telemetry.

---

## **2\) Stack \+ environment constraints**

### **Common stacks**

* **Workflow UIs**: ComfyUI-like node graphs, A1111-like UIs, custom pipelines.  
* **Core runtime**: PyTorch \+ diffusion pipelines (diffusers/custom), attention optimizations, model/LoRA loaders.  
* **Training**: script/tooling-driven LoRA training (people expect “one-click templates” and known-good configs).

### **CUDA/driver constraints (platform-friendly approach)**

* Provide **one main image** for modern GPUs (CUDA 12.x family).  
* Keep a **compat image** for older deps (so customer projects don’t break).

### **Container vs VM**

* **Containers** are the default for studios.  
* Some want “desktop-like” environments (RDP/VNC) — you can still do this container-first by offering:  
  * persistent workspace volume  
  * optional remote UI access in a controlled way

### **Storage expectations**

* NVMe is *critical* because studios:  
  * download lots of checkpoints/LoRAs  
  * cache heavily  
  * write tons of images  
* “Per-user workspace volume” becomes a product feature (model library \+ outputs).

### **Networking needs**

* Usually needs outbound internet for model downloads (or you provide mirrors/caching).  
* Inbound should be **private by default**, optional exposure for UI.

**Output artifact:** reference runtime image set (pinned)

* `sd-workspace-cu12` (UI \+ common deps)  
* `sd-infer-cu12` (lean inference)  
* `sd-train-lora-cu12` (training toolchain)  
* `sd-upscale-cu12`  
* `sd-compat-cu11` (fallback)

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Client product photos, unreleased creatives, brand assets, sometimes faces (still sensitive).  
* Prompts/workflows are also IP.

### **Common requirements**

* Strong tenant isolation (workspace isolation \+ access control)  
* Encryption at rest for workspace volumes \+ outputs  
* Team RBAC: artist vs admin vs billing

### **Logging rules**

* Don’t store raw prompts or images by default.  
* Store only metadata: job params (steps/resolution/model ids), durations, cost, errors.

**Output artifact:** security checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* **Time-to-first-image** (cold start pain)  
* **Cache effectiveness** (model downloads killing UX)  
* **Queue UX** (cancel/retry/priorities)  
* **Reproducibility** (same workflow should stay stable over time)

### **Starting SLO priors**

* Warm start (cached): **p90 \< 2–5 min** to “ready”  
* Cold start (uncached): **p90 \< 10–20 min**  
* Batch completion reliability: **\>99%** with resumable outputs  
* Output durability: “never lose images”

### **Support playbook themes**

* OOM (resolution/batch/too many addons)  
* Slowdowns (cache miss, throttled downloads, noisy neighbor)  
* “Output changed” (version drift) → need pinned image digests \+ model hashes  
* Storage full / missing outputs → retention \+ bucket export

### **Pricing sensitivity**

* On-demand for bursts  
* Monthly reserved “workspaces”  
* Spot for large batches if resumable

**Output artifact:** SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How studios work today**

* Browser UI \+ saved workflows/presets  
* Shared team model library (checkpoints/LoRAs/ControlNet assets)  
* Batch automation for campaigns

### **Auth patterns**

* Org \+ team RBAC  
* Shareable projects  
* API keys later for automation

### **Artifact flow**

* Inputs: checkpoints, LoRAs, reference images, workflow JSON  
* Outputs: images \+ thumbnails \+ run manifests (seed/model hash/workflow version)  
* Destination: studio bucket \+ optional managed library

### **Golden flows**

1. **Interactive workspace** → iterate → save workflow preset → export outputs  
2. **Batch campaign render** → queue → continuous upload \+ manifest → notify completion  
3. **Train LoRA** → store adapter in private library → apply in inference → version \+ share

---

If you want, next step is the same as we did before: **interview checklist \+ telemetry spec** for this niche (cold start vs warm start, cache hit rate, OOM causes, images/min, and batch job resume mechanics).

