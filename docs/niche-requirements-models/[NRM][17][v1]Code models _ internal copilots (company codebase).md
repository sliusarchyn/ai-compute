# **Code models / internal copilots (company codebase)**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: prompt templates, tool-call flows, eval suites on small repos  
* **Repo ingestion**: clone/mirror repos, parse languages, build code graphs  
* **Indexing for RAG**: chunking, embeddings, symbol indexing, dependency maps  
* **Copilot inference (interactive)**: autocomplete, chat, code review suggestions (latency-sensitive)  
* **Batch code analysis**: security scans, refactor suggestions, doc generation  
* **Fine-tuning / adapters**:  
  * instruction tuning on internal style/guides  
  * RAG-only approaches (common) vs light fine-tunes (less common)  
* **Eval/regression**: unit-style coding tasks, patch correctness, latency/cost tests  
* **Governance ops**: access control, audit trails, model version rollouts

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (flow \+ eval) | 1–20 repos | 0–1×24GB → 1×48GB | 8–32 vCPU, 32–256GB | 200GB–2TB | 2–12h → 1–7 days | Many small reads; lots of logs; repeated eval runs |
| Repo ingestion \+ parsing | 10–10,000 repos | 0 GPUs → 1×24GB | 16–256 vCPU, 64GB–2TB | 0.5–8TB | 1–12h → days | Many small files; CPU-bound parsing; writes indexes |
| Embedding \+ RAG indexing | 1M–10B tokens/code chunks | 1×24GB → 4×48GB | 16–256 vCPU, 64GB–2TB | 1–16TB | 1–24h → days–weeks | Read-heavy; writes vectors \+ metadata; many small ops |
| Symbol/graph indexing | large monorepos | 0 GPUs → 1×24GB | 32–512 vCPU, 128GB–4TB | 1–16TB | 6–48h → days | CPU/RAM heavy; writes graph DB/index |
| Copilot chat (interactive) | 100–100k concurrent | 1×24GB → 8×80GB (scale-out) | 16–256 vCPU, 64GB–2TB | 200GB–4TB | continuous | Tail latency sensitive; retrieval-heavy; caching helps |
| Autocomplete (IDE) | 1k–1M req/s | 1×24GB → 8×80GB (scale-out) | 16–256 vCPU, 64GB–2TB | 200GB–4TB | continuous | Very latency-sensitive; small prompts; heavy cache |
| Batch code review / analysis | 10–100k PRs/day | 1×24GB → 4×48GB | 16–256 vCPU, 64GB–2TB | 0.5–8TB | minutes → days | Reads diffs; writes comments; can be queue-based |
| Fine-tuning/adapters (optional) | internal Q\&A/style data | 1×80GB → 4×80GB | 32–256 vCPU, 128GB–2TB | 1–16TB | 6–48h → 2–14 days | Dataset reads; checkpoints; eval suites |

### **Parallelism expectations**

* Indexing: shard by repo/branch; embarrassingly parallel.  
* Interactive: scale with replicas \+ batching; autocomplete wants micro-batching and warm pools.  
* Batch analysis: queue-based fan-out.

### **Failure tolerance**

* Indexing must be idempotent by `repo@commit` and can be resumed.  
* Interactive services need redundancy; rollouts must be safe (canary).  
* Fine-tuning must resume-from-checkpoint.

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined later via telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **RAG stack**: embeddings \+ vector DB \+ metadata store; optional re-rankers  
* **Code parsing**: tree-sitter/LSIF-like symbol extraction; language servers  
* **Serving**: low-latency LLM serving (vLLM/TensorRT-LLM-like), streaming responses  
* **IDE integration**: VS Code/JetBrains plugins, auth \+ telemetry  
* **Eval harness**: repo-based unit tasks, patch validation, regression dashboards

### **CUDA/driver constraints**

* Needed for model serving/embedding/rerankers; stable CUDA 12.x baseline.  
* Compat images for pinned stacks; model version pinning matters.

### **Container vs VM requirements**

* Containers default.  
* Some enterprises require VMs/dedicated tenancy.

### **Storage expectations**

* Local NVMe for:  
  * cloning monorepos, indexes, caches  
  * model weights for fast startup  
* Durable storage for:  
  * index snapshots, evaluation artifacts, audit logs

### **Networking needs**

* Private connectivity to Git hosts (GitHub Enterprise/GitLab), internal package registries.  
* Outbound restrictions common.  
* Multi-region may be needed for developer latency.

**Output artifact:** reference runtime image set

* `code-ingest-parse`  
* `code-embed-index-cu12`  
* `code-symbol-index`  
* `code-copilot-serve-cu12`  
* `code-autocomplete-serve-cu12`  
* `code-batch-review`  
* `code-finetune-cu12`  
* `compat`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Company code is core IP; may include secrets.  
* Strict restrictions on where code can go and who can access it.

### **Common requirements**

* Region lock / on-prem-like isolation  
* Dedicated projects/nodes for strict customers  
* Encryption at rest \+ TLS in transit  
* RBAC (repo/team-based), SSO integration later  
* Audit logs for access to code context and completions

### **Logging rules**

* Don’t store raw code snippets in logs by default.  
* Store metadata: repo IDs, commit hashes, prompt token counts, latency, error codes.  
* If storing prompts for QA, it must be opt-in with redaction and retention controls.

**Output artifact:** security/compliance checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* Latency (autocomplete especially)  
* Answer quality and grounding (no hallucinated APIs)  
* Data leakage prevention (no cross-tenant leakage)  
* Cost per dev per month  
* Safe rollout (new model shouldn’t break workflows)

### **SLO starting points (priors)**

* Autocomplete p95: very low (customer-specific), minimal cold starts  
* Chat p95 latency targets per customer  
* Index freshness: commits reflected within X minutes/hours  
* Reliability: \>99% service availability (tiered)  
* Observability: cache hit rate, retrieval hit rate, refusal rate, tokens/sec

### **Support playbook themes**

* “It’s slow” (cold starts, cache misses, overload)  
* “Wrong suggestions” (bad retrieval, outdated index, model drift)  
* “It leaked private code” (governance incident)  
* “Index is stale” (webhooks broken, parsing failures)  
* “Costs spiked” (no routing, large contexts, no caching)

### **Pricing sensitivity & billing preference**

* Enterprises want predictable per-seat pricing.  
* Compute heavy users want usage-based plus caps.  
* Dedicated tenancy is a premium tier.

**Output artifact:** SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Connect repos → build indexes → install IDE plugin → use chat/autocomplete → collect feedback → iterate prompts/models  
* Nightly index refresh \+ webhook-driven incremental updates  
* Regression suite before model upgrades

### **Auth patterns**

* Org/team RBAC \+ service accounts  
* SSO (SAML/OIDC) later, but audit logs early

### **Artifact flow**

* Inputs: repo snapshots, code symbols, docs, tickets, coding guidelines  
* Outputs: embeddings/indexes, completions, suggested patches, eval reports, run manifests  
* Destinations: IDE clients, PR comments, internal dashboards

### **Golden flows (end-to-end)**

1. **Repo indexing \+ chat**  
   * Mirror repo → parse symbols → embed chunks → build index → chat with citations → enforce RBAC  
2. **Autocomplete at scale**  
   * IDE sends small context → fast model route → return suggestion in latency budget → cache \+ telemetry  
3. **Safe model rollout**  
   * Run eval suite → canary to subset of devs → monitor metrics \+ feedback → promote/rollback

