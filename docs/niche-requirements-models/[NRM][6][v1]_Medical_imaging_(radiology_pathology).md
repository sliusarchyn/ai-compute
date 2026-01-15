# **Medical imaging (radiology / pathology)**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: data import sanity (DICOM/WSI), preprocessing, small inference runs  
* **Batch inference (radiology)**: process studies from PACS (CT/MRI/X-ray) and write results back  
* **Batch inference (pathology WSI)**: tile/patch extraction → GPU inference → aggregation → overlays  
* **Online inference**: low-latency “on-demand” analysis for a single study/slide  
* **Preprocessing pipelines**: decompression, resampling, normalization, tiling, QC  
* **Training / tuning**: segmentation/classification (3D U-Net, detection), domain adaptation  
* **Eval/regression**: AUC/DICE, calibration, drift checks; dataset curation loops

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (import \+ small runs) | 10–200 studies or 1–20 slides | 0–1×24GB → 1×48GB | 16–32 vCPU, 64–256GB | 0.5–2TB | 30–120 min → 6–24h | Lots of parsing/decode; small outputs; heavy logs/QC |
| Radiology batch inference | 100–100k studies/day | 1×24GB → 4×48GB | 32–256 vCPU, 128GB–1TB | 1–8TB | \~0.1×–1× realtime → 1×–5× realtime | Read DICOM series; write masks/measurements \+ audit; bursty ingest |
| Radiology online inference | single study on demand | 1×24GB → 1×48GB | 16–64 vCPU, 64–256GB | 0.5–2TB | 5–60s → 1–10 min | Tail-latency sensitive; caching of models \+ preproc |
| Pathology WSI preprocessing (tiling/QC) | 1–5,000 slides/job | 0–1×24GB → 1×48GB | 32–256 vCPU, 128GB–1TB | 2–16TB | 1–6h → 1–5 days | Extremely I/O heavy; tile reads \+ huge scratch writes; CPU-bound often |
| Pathology WSI batch inference | 10–5,000 slides/day | 1×48GB → 4×80GB | 64–512 vCPU, 256GB–2TB | 4–32TB | 10–60 min/slide → 1–8h/slide | Read gigapixel WSI tiles; write heatmaps/overlays; big intermediate caches |
| Training/tuning (radiology) | 1k–1M studies | 1×80GB → 8×80GB | 64–512 vCPU, 256GB–2TB | 2–32TB | 1–7 days → 2–8 weeks | Streaming data \+ augmentation; periodic checkpoints (GBs–100s GBs) |
| Training/tuning (pathology WSI) | 1k–1M slides | 2×80GB → 16×80GB | 128–1024 vCPU, 0.5–8TB | 8–200TB (cluster) | 1–14 days → 1–12 weeks | Patch sampling \+ massive I/O; often two-stage (feature extract → train head) |

**Why the storage/I/O is brutal (pathology)**

* Whole-slide images (WSIs) are commonly **gigapixel** scale (e.g., \~1.6 gigapixels average reported in one study). ([PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC9577055/?utm_source=chatgpt.com))  
* Even with compression, single slides can still be **\~1–2 GB** and are “challenging to move around, save, load, and view.” ([NVIDIA Developer](https://developer.nvidia.com/blog/whole-slide-image-analysis-in-real-time-with-monai-and-rapids/?utm_source=chatgpt.com))  
  This is why pathology workloads often need **big NVMe scratch \+ high RAM**.

### **Parallelism expectations**

* **Radiology inference**: parallel across studies; some models are 3D and benefit from GPU batching, but CPU preprocessing can bottleneck.  
* **Pathology WSI**: parallel across slides and across tiles/patches; GPU does inference, CPU often does decoding/tiling.  
* **Training**: single-node DDP is common; multi-node appears at higher scale (especially pathology).

### **Failure tolerance**

* Pipelines must be **restartable/idempotent**:  
  * by **study/series UID** (DICOM) or **slide ID \+ tile range**  
  * stage-level resume (decode → preprocess → infer → postprocess → export)  
* **Resume-from-checkpoint** is expected for training (jobs are long and expensive).

**Output artifact:** a workload profile with p50/p90 resource \+ time numbers (table above), later replaced with real telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **Core ML**: PyTorch-based (medical often uses MONAI-style tooling in practice)  
* **Radiology I/O**: DICOM ingest (C-STORE / DICOMweb), decompression, resampling; integration with **PACS/RIS** workflows is a core requirement ([Radiological Society of North America](https://pubs.rsna.org/doi/full/10.1148/radiol.232653?utm_source=chatgpt.com))  
* **Pathology I/O**: WSI readers (tile/patch extraction), background removal/QC, pyramid levels  
* **Serving**: batch runners \+ optional online endpoint; results export (DICOM SR/SEG, overlays, reports)

### **CUDA/driver constraints**

* Prefer a stable **CUDA 12.x** baseline for modern GPUs.  
* Keep a **compat** image for pinned research stacks.

### **Container vs VM requirements**

* **Containers** are default for research \+ batch pipelines.  
* **VMs / dedicated tenancy** show up more often here than other niches due to compliance and hospital IT policies.

### **Storage expectations**

* Radiology: needs reliable access to PACS \+ intermediate caching.  
* Pathology: needs **very fast local NVMe** (tiling, intermediate heatmaps) and durable object storage for outputs.  
* Many customers want “bring your own bucket” for compliance and ownership.

### **Networking needs**

* Hospitals often require **private connectivity** (VPN / private link / peering).  
* Inbound is typically restricted; outbound may be allowlisted.

**Output artifact:** “reference runtime image set” (base images \+ pinned versions)

* `med-rad-infer-cu12` (radiology inference \+ DICOM tooling)  
* `med-wsi-preproc` (WSI tiling/QC)  
* `med-wsi-infer-cu12` (pathology inference)  
* `med-train-cu12` (training \+ augmentation)  
* `compat-cu11` (fallback)

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* This niche often involves **ePHI/PHI** and regulated environments.  
* Strong requirements for access control, auditing, integrity, and secure transmission are typical expectations under HIPAA security practices. ([HHS.gov](https://www.hhs.gov/sites/default/files/ocr/privacy/hipaa/administrative/securityrule/techsafeguards.pdf?utm_source=chatgpt.com))

### **Common requirements you’ll hear**

* **Region lock / residency**  
* **Isolation**: dedicated nodes (often), dedicated network, least-privilege RBAC  
* **Encryption**: at rest \+ in transit; key management expectations  
* **Audit trails**: who accessed studies/slides, who ran models, who exported results  
* **Retention rules**: strict retention and deletion policies for intermediates

### **Logging rules**

* Do not log image pixels or patient identifiers by default.  
* Store minimal metadata (hashed IDs, job timings, error codes), with controlled access.

**Output artifact:** security/compliance checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* **Reliability**: no dropped studies, no silent partial failures  
* **Latency/throughput**: predictable processing time per study/slide  
* **Workflow integration**: results must land where clinicians work (PACS/RIS/EHR workflows), using standards-based interoperability ([Radiological Society of North America](https://pubs.rsna.org/doi/full/10.1148/radiol.232653?utm_source=chatgpt.com))  
* **Reproducibility**: pinned model \+ pinned runtime \+ audit logs

### **SLO starting points (priors)**

* Batch pipelines: **\>99%** completion with resumable stage retries  
* Online inference: clear p95/p99 targets (customer-specific)  
* Data integrity: zero-loss guarantees for accepted inputs \+ durable outputs

### **Support playbook themes**

* “Import failed” (DICOM quirks, compression, missing series)  
* “It’s slow” (CPU preprocessing, I/O saturation, WSI tiling bottleneck)  
* “Results don’t show in PACS” (integration/export format issues)  
* “We need audit evidence” (access logs, run manifests, retention)

### **Regulatory note (product direction)**

If you’re building toward clinical decision support, AI can fall under **Software as a Medical Device (SaMD)** expectations; FDA highlights that adaptive AI/ML changes may need premarket review. ([U.S. Food and Drug Administration](https://www.fda.gov/medical-devices/software-medical-device-samd/artificial-intelligence-software-medical-device?utm_source=chatgpt.com))

**Output artifact:** per-niche SLOs \+ support playbook \+ pricing sensitivity notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Radiology: studies arrive in PACS → batch or near-real-time AI → results back into PACS viewer/report workflow  
* Pathology: slides scanned → WSI stored → tiling \+ inference → heatmaps/regions → pathologist review system

### **Auth patterns**

* Org/project RBAC \+ service accounts  
* Enterprise/hospitals: SSO, strict audit, least privilege

### **Artifact flow**

* Inputs: DICOM studies / WSI slides \+ metadata  
* Outputs: segmentations/measurements, overlays/heatmaps, structured reports, run manifests  
* Destinations: PACS-integrated outputs \+ customer storage (preferred)

### **Golden flows (end-to-end)**

1. **Radiology triage batch**  
   * Ingest from PACS → preprocess → infer → export results to PACS/reporting → audit log  
2. **Pathology WSI slide analysis**  
   * Pull WSI → tile/QC → infer patches → aggregate → heatmap overlay → export \+ retention policy  
3. **Research → validated deployment**  
   * Curate dataset → train/tune with full lineage → offline eval/regression → controlled rollout with pinned versions \+ audit

