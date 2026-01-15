# **Speech AI (ASR/TTS/voice cloning — legit use)**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: sample clips, prompt/SSML checks, pipeline sanity  
* **Batch ASR**: transcribe files (meetings, podcasts, call archives)  
* **Streaming ASR**: realtime transcription (calls, live captions)  
* **Diarization \+ timestamps**: “who spoke when”, word/segment alignment  
* **Batch TTS**: generate many audio assets (ads, IVR prompts, videos)  
* **Realtime TTS**: low-latency speech synthesis (agents, assistants)  
* **Voice conversion** (optional): convert voice style (only with explicit consent)  
* **Voice cloning / speaker adaptation training**: build a new voice (consent \+ governance)  
* **Eval/regression**: WER/MOS tracking, latency tests, stress tests

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (ASR/TTS sanity) | 10–200 clips | 0 GPUs → 1×24GB | 8–16 vCPU, 16–64GB | 50–200GB | 10–60 min → 2–6h | Small audio reads; lots of logs; quick iterations |
| Batch ASR (offline) | 10–100k hours audio/day | 1×24GB → 4×48GB | 16–128 vCPU, 64–512GB | 0.5–4TB | 0.1×–1× realtime → 1×–5× realtime | Decode-heavy reads; small text writes; batching critical |
| Streaming ASR (realtime) | 10–10k concurrent streams | 1×24GB → 4×48GB | 16–128 vCPU, 64–512GB | 200GB–1TB | continuous | Many small packets; tail latency sensitive; needs stable batching/backpressure |
| Diarization \+ alignment | 100–10k hours/day | 0–1×24GB → 2×48GB | 16–128 vCPU, 64–512GB | 0.5–2TB | 0.5×–2× realtime → 2×–10× realtime | More CPU/RAM; intermediate embeddings; metadata outputs |
| Batch TTS (asset generation) | 100–5M utterances/job | 1×24GB → 2×48GB | 8–64 vCPU, 32–256GB | 0.5–4TB | 10 min–6h → 6h–3 days | Many small audio writes; optional post-processing; uploads in batches |
| Realtime TTS (prod) | 10–5k concurrent | 1×24GB → 2×48GB | 8–64 vCPU, 32–256GB | 100–500GB | continuous | Strict p95 latency; short requests; caching \+ warm pools |
| Voice cloning / speaker adaptation (training) | 5–120 min voice data (consented) | 1×48GB → 2×80GB | 16–64 vCPU, 64–512GB | 0.5–4TB | 2–12h → 1–7 days | Dataset reads; periodic checkpoints; artifacts (models) medium-size |
| Larger fine-tuning (multi-speaker / domain) | 100–10k hours | 4×80GB → 16×80GB | 128–512 vCPU, 0.5–4TB | 8–100TB (cluster) | 2–14 days → 2–8 weeks | Continuous streaming \+ big checkpoints; needs robust resume |

**Parallelism expectations**

* **ASR**: scale via **batching \+ many worker replicas**; GPU helps, but **CPU decode** can be the bottleneck.  
* **Streaming**: concurrency \+ tail latency dominate; careful micro-batching and backpressure.  
* **Training**: single-node often enough for speaker adaptation; multi-node only for large corp/domain training.

**Failure tolerance**

* Batch pipelines must be **idempotent** (file\_id / segment\_id) and resumable.  
* Streaming: tolerate node loss via **session handoff** (or quick reconnect) and partial transcript persistence.  
* Spot/preemptible is fine for batch jobs if progress is checkpointed; risky for realtime.

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined later via interviews \+ telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **ASR inference**: PyTorch/ONNX/TensorRT-style deployments; streaming wrappers; VAD.  
* **Diarization**: speaker embeddings \+ clustering; alignment/timestamps tooling.  
* **TTS**: neural TTS runtime, vocoder, optional SSML parsing.  
* **Serving**: gRPC/WebSocket for streaming ASR; HTTP/gRPC for TTS; queue workers for batch.  
* **Optimization**: model quantization, batching, GPU sharing (MIG/time-slicing) for inference density.

### **CUDA/driver constraints**

* Provide a stable default **CUDA 12.x** image for modern GPUs.  
* Keep a **compat** image for pinned/legacy stacks.  
* Version pinning matters for reproducibility (model \+ runtime \+ tokenizer \+ audio libs).

### **Container vs VM requirements**

* **Containers** are default.  
* VMs show up when customers need strict isolation or custom audio drivers, but most can be containerized.

### **Storage expectations**

* **Local NVMe cache** for:  
  * audio shards, feature caches, intermediate embeddings  
  * hot model files (fast startup)  
* **Object storage** for:  
  * audio archives, transcripts, alignments, model artifacts  
* For streaming, storage is less about throughput and more about **durable session logs**.

### **Networking needs**

* Streaming ASR needs:  
  * stable inbound (WebRTC/SIP/WebSocket/gRPC gateways)  
  * predictable p95 latency  
* Many enterprise customers want **private networking** (VPC/VPN/peering) and egress control.

**Output artifact:** “reference runtime image set” (base images \+ pinned versions)

* `asr-batch-cu12`  
* `asr-streaming-cu12` (with gateway dependencies)  
* `diarization-align`  
* `tts-batch-cu12`  
* `tts-realtime-cu12`  
* `speech-train-cu12` (speaker adaptation)  
* `compat-cu11`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Voice audio is usually **personal data** (PII); call center audio may be regulated by policy/law.  
* For “legit voice cloning”, you need **explicit consent \+ governance**.

### **Common requirements**

* **Consent \+ provenance**: store proof of consent; track dataset origin.  
* **Region lock** / residency.  
* **Isolation**: dedicated nodes for some customers; at minimum dedicated project \+ encrypted volumes.  
* **Encryption**: at rest \+ in transit.  
* **Audit logs**: who uploaded audio, who ran cloning, who accessed models/outputs.

### **Logging rules**

* Don’t store raw audio or full transcripts in logs by default.  
* Store metadata only: job IDs, durations, error codes, aggregate metrics.  
* Allow customer-controlled export to their SIEM/storage.

**Output artifact:** security/compliance checklist \+ tenancy requirements (consent, auditing, retention, isolation, residency).

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* **Latency** (streaming ASR partials; realtime TTS first audio chunk)  
* **Tail latency** (p95/p99) more than averages  
* **Accuracy/quality metrics**:  
  * ASR: WER, domain-specific term accuracy  
  * TTS: MOS-style user rating, pronunciation consistency  
* **Throughput \+ cost** for batch pipelines  
* **Reliability**: no dropped streams, resumable batch, durable outputs

### **SLO starting points (priors)**

* Streaming ASR: stable partial updates; p95 latency targets per customer  
* Realtime TTS: fast “time-to-first-audio”; consistent p95  
* Batch: predictable realtime factor (RTF) and high completion rate

### **Support playbook themes**

* “Latency spiked” (batching/backpressure, gateway saturation)  
* “Accuracy got worse” (domain drift, wrong model version)  
* “Pronunciation issues” (lexicons/SSML/config)  
* “Costs exploded” (no batching, too high sample rates, no caching)  
* “Streaming disconnects” (network/gateway resilience)

### **Pricing sensitivity & billing preference**

* Batch: per-hour compute \+ per-minute audio processed \+ storage  
* Streaming/realtime: reserved capacity or concurrency tiers (predictable bills)  
* Enterprises often want reservations \+ invoices.

**Output artifact:** per-niche SLOs \+ support playbook \+ pricing sensitivity notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Batch ASR: upload to bucket → run workers → store transcripts \+ timestamps  
* Streaming ASR: telephony/WebRTC gateway → ASR streams → downstream NLP/CRM  
* TTS: text/SSML inputs → audio outputs → media pipelines  
* Voice cloning (consented): upload voice samples → train/adapt → gated deployment

### **Auth patterns**

* Org/project RBAC, API keys, service accounts  
* Enterprise: SSO later, audit logs now

### **Artifact flow**

* Inputs: audio files/streams, text/SSML, lexicons, domain vocabulary  
* Outputs: transcripts, word timestamps, diarization segments, audio files, model artifacts  
* Destinations: customer bucket/registry \+ optional managed storage

### **Golden flows (end-to-end)**

1. **Call center streaming ASR**  
   * Stream audio → partial transcripts \+ diarization → store finalized transcript \+ timestamps → push to CRM/analytics  
2. **Batch transcription pipeline**  
   * Drop audio in bucket → batch ASR \+ alignment → write JSON/CSV outputs → quality report (WER proxies, term hits)  
3. **Consented voice creation for TTS**  
   * Upload consented samples → train/adapt voice → review quality → deploy behind RBAC \+ audit → generate assets or realtime TTS

