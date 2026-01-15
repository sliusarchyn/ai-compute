# **Gaming / NPC intelligence / real-time agents**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: prompt/policy iteration, tool calls, small sims, debugging behaviors  
* **Realtime NPC inference**: dialogue, decision-making, planning, tool use (latency-sensitive)  
* **Batch generation**: NPC dialogue banks, quest text, localization drafts, lore content  
* **RAG/knowledge**: game lore retrieval, player state/context, memory systems  
* **Simulation at scale**: multi-agent playtests, stress tests, economy sims  
* **Training/tuning**:  
  * fine-tune smaller models for style/guardrails  
  * RL/IL for behavior policies (less common but valuable)  
* **Eval/regression**: safety/consistency tests, “golden” conversation suites, load tests  
* **Production ops**: scaling concurrent sessions, observability, abuse prevention

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (agent iteration) | 10–500 sessions | 0–1×24GB → 1×48GB | 8–32 vCPU, 32–256GB | 200GB–2TB | 1–6h → 1–3 days | Many small requests; logs/traces heavy; rapid restarts |
| Realtime NPC inference (prod) | 100–1M concurrent NPC turns (varies) | 1×24GB → 8×80GB (scale-out) | 16–256 vCPU, 64GB–1TB | 200GB–2TB | continuous | Tail latency sensitive; small requests; caching \+ warm pools |
| RAG retrieval \+ memory | 10k–100M docs/events | 0–1×24GB → 2×48GB | 16–256 vCPU, 64GB–2TB | 0.5–8TB | continuous | Vector DB reads/writes; small metadata; high QPS |
| Batch content generation | 1k–100M lines/assets | 1×24GB → 4×48GB | 16–128 vCPU, 64GB–512GB | 0.5–8TB | 1–12h → days–weeks | Write-heavy (text/assets); job queue; artifact management |
| Large-scale simulation/playtests | 1k–1M agents | 0–1×24GB → 4×48GB | 64–1024 vCPU, 256GB–8TB | 1–32TB | 6–48h → days–weeks | CPU-heavy; logs and metrics huge; snapshots |
| Fine-tuning (style/guardrails) | curated corpora | 1×48GB → 4×80GB | 32–256 vCPU, 128GB–2TB | 1–16TB | 6–48h → 2–10 days | Streaming reads; checkpoints; eval suites |
| RL/IL behavior training | episodes/trajectories | 1×24GB → 8×80GB | 64–512 vCPU, 256GB–4TB | 2–32TB | 2–14 days → weeks | Rollouts \+ learner; heavy logs; checkpointing |

### **Parallelism expectations**

* Realtime: scale-out inference replicas; concurrency \+ batching.  
* RAG: split retrieval/indexing from generation; horizontal scale.  
* Simulation: CPU scale across many instances; GPU optional depending on model/visual components.  
* Training: DDP for multi-GPU; many runs are single-node fine-tunes.

### **Failure tolerance**

* Realtime: must be resilient; degrade gracefully (fallback responses, timeouts).  
* Batch and simulation: idempotent jobs by `job_id`/`seed`; easy resume.  
* Training: resume-from-checkpoint mandatory.

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined via telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **LLM serving**: optimized inference runtime (vLLM/TensorRT-LLM-like), KV cache, batching  
* **Tool calling**: game APIs, function calling, state machines  
* **RAG**: embeddings \+ vector DB \+ memory store; event streams  
* **Observability**: traces per “turn”, conversation logs with redaction, latency histograms  
* **Game integration**: SDKs for Unity/Unreal/server-authoritative backends

### **CUDA/driver constraints**

* Stable CUDA 12.x for modern GPUs.  
* Compat images for pinned stacks.  
* Version pinning is crucial for consistent behavior.

### **Container vs VM requirements**

* Containers are default.  
* VMs may be requested for dedicated tenancy or special networking.

### **Storage expectations**

* Low-latency storage for:  
  * vector indexes  
  * session memory  
  * caches  
* Durable storage for:  
  * logs (with retention), content assets, fine-tuned models, evaluation suites

### **Networking needs**

* Low-latency networking for realtime inference.  
* Often global distribution (regions close to players).  
* Private networking between game servers and AI inference is common.

**Output artifact:** reference runtime image set

* `npc-llm-serve-cu12`  
* `npc-rag-index`  
* `npc-rag-query`  
* `npc-batch-gen`  
* `npc-sim`  
* `npc-train-cu12`  
* `compat`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Player chat can contain PII and safety risks.  
* Game content IP is sensitive.  
* Abuse and prompt injection are real.

### **Common requirements**

* Isolation per game studio/project  
* Encryption at rest \+ TLS in transit  
* RBAC for devs vs ops vs content writers  
* Retention controls for chat logs (privacy)  
* Safety guardrails and incident audit trails

### **Logging rules**

* Redact player identifiers by default.  
* Store only metadata and safety signals unless opt-in.  
* Provide audit logs for moderation decisions.

**Output artifact:** security checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* Tail latency (p95/p99) and consistency  
* Uptime and autoscaling behavior  
* Behavioral consistency (no sudden personality drift)  
* Cost per 1k turns / per session  
* Safety: toxicity, harassment, policy violations, jailbreak resistance

### **SLO starting points (priors)**

* Realtime inference: customer-specific p95/p99 latency budgets  
* Availability: high for production regions  
* Batch jobs: \>99% completion with retries  
* Observability: per-turn traces, token usage, cache hit rate, refusal rates

### **Support playbook themes**

* “Latency spikes” (cold starts, cache misses, batch config)  
* “NPC acts weird” (version drift, prompt changes)  
* “Safety incidents” (guardrail gaps, missing filters)  
* “Costs exploded” (no caching, too-large models, no routing)  
* “Game integration broken” (SDK/API schema changes)

### **Pricing sensitivity & billing preference**

* Realtime often priced by concurrency \+ tokens/turns; studios want predictable bills.  
* Batch generation priced per output volume.  
* Enterprise studios want reserved capacity and SLAs.

**Output artifact:** SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Build agent prompts/tools → run playtests → regression suites → staged rollout  
* Separate pipelines for batch content generation vs realtime inference  
* Ongoing safety tuning \+ metrics monitoring

### **Auth patterns**

* Org/project RBAC, service accounts  
* Audit logs; sometimes SSO for larger studios

### **Artifact flow**

* Inputs: game lore/docs, prompts, tool schemas, player state snapshots  
* Outputs: NPC responses, action decisions, content assets, traces, evaluation reports  
* Destinations: game servers, content pipelines, analytics dashboards

### **Golden flows (end-to-end)**

1. **Realtime NPC in production**  
   * Game server sends context → RAG fetch \+ memory → LLM responds → tool calls → return action \+ text → log/redact \+ metrics  
2. **Playtest regression suite**  
   * Run scripted player interactions across seeds → score behavior \+ safety → compare against baseline → promote/rollback  
3. **Batch content pipeline**  
   * Generate dialogue banks/quests → human review → localization → publish with versioning and provenance

