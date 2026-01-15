# Research script

To do “deep” niche research *and* end up with a solid **Agent Communication Protocol (ACP) contract**, you need two parallel tracks:

1. **Niche requirements model** (what users *actually* do \+ constraints)  
2. **Protocol/interop model** (how agents must talk: runs, messages, artifacts, state, errors)

ACP (Agent Communication Protocol) itself is a **REST-based standard** meant for agent interoperability, supporting **multimodal messages, sync/async \+ streaming, discovery, stateful/stateless patterns, and long-running tasks**. ([Agent Communication Protocol](https://agentcommunicationprotocol.dev/introduction/welcome))

---

## **A. What you must learn for each niche (to size nodes \+ product \+ ACP needs)**

### **1\) Workload anatomy (the “physics”)**

Figure out the *real* workload shapes:

* **Job types:** dev/test, fine-tune, full training, batch inference, online inference  
* **Runtime distribution:** seconds/minutes/hours/days (p50/p90)  
* **Resource envelope:** VRAM, RAM, CPU cores, storage (NVMe), network  
* **I/O pattern:** dataset reads, checkpoint writes, artifact uploads (how many GB, how often)  
* **Parallelism:** single GPU vs multi-GPU (DDP), multi-node, need for NCCL, interconnect  
* **Failure tolerance:** can resume from checkpoint? how painful is a preemption?

**Output artifact:** a “workload profile” with p50/p90 resource \+ time numbers.

### **2\) Stack \+ environment constraints**

* Required frameworks (PyTorch/JAX/TensorRT/etc), CUDA versions, driver constraints  
* Container vs VM requirements (custom kernel? privileged? Windows?)  
* Storage expectations (local NVMe scratch vs object storage; dataset locality)  
* Networking needs (public ingress? private VPC? outbound restrictions?)

**Output artifact:** a “reference runtime image set” (base images \+ pinned versions).

### **3\) Data governance \+ risk**

* Data sensitivity (PII/PHI/IP), compliance (HIPAA-like, SOC2 expectations), audit needs  
* Data residency requirements (region lock)  
* Isolation needs (dedicated node, dedicated network, encrypted volumes)  
* Logging rules (what can/can’t be stored)

**Output artifact:** “security/compliance checklist” \+ tenancy requirements.

### **4\) Operational expectations (what they’ll blame you for)**

* Uptime/SLO needs (dev is tolerant, production less so)  
* Observability needs: metrics they care about (latency, throughput, GPU util, queue time)  
* Support expectations: onboarding help, “my job is stuck”, “OOM”, “slow GPU”  
* Cost sensitivity & billing model preference (on-demand vs reserved vs spot/preemptible)

**Output artifact:** per-niche SLOs \+ “support playbook” \+ pricing sensitivity notes.

### **5\) User workflow & integrations**

* How they launch work today: notebooks, CLI, CI/CD, MLOps tools, HuggingFace, Weights & Biases  
* Auth patterns: API keys, org accounts, RBAC, team sharing  
* Artifact flow: model registry, dataset sources, where outputs must land

**Output artifact:** 2–3 “golden flows” per niche (end-to-end).

---

## **B. What you must define to implement ACP cleanly (turn research into a protocol contract)**

ACP is designed to standardize **how agents communicate** (not to dictate your orchestration logic). ([IBM](https://www.ibm.com/think/topics/agent-communication-protocol))  
So for your platform, you translate each niche into **agent capabilities \+ message/run semantics**:

### **1\) Agent roles and boundaries**

Define *who* speaks ACP:

* Customer agent (their workflow agent)  
* Your platform agents (provisioning, job runner, data mover, billing/metering)  
* Optional: “niche agents” (Fine-tune Agent, RAG Indexing Agent, Video Render Agent)

### **2\) Capability model (discoverability)**

ACP has concepts like **agent manifest/discovery** so clients can learn “what this agent can do.” ([Agent Communication Protocol](https://agentcommunicationprotocol.dev/introduction/welcome))  
For each niche, define:

* Actions (e.g., `start_training`, `resume_from_checkpoint`, `export_model`, `run_eval`)  
* Required inputs (dataset refs, hyperparams, container image, GPU type)  
* Produced artifacts (checkpoints, model package, evaluation report)  
* Limits (max duration, supported GPUs, max dataset size)

### **3\) Run lifecycle \+ long-running tasks**

ACP explicitly supports **async-first**, long tasks, and streaming updates. ([Agent Communication Protocol](https://agentcommunicationprotocol.dev/introduction/welcome))  
You must define:

* States: queued → provisioning → running → checkpointing → completed/failed/canceled  
* Progress events: step %, tokens/sec, samples/sec, ETA (best-effort)  
* Idempotency \+ resume semantics (critical for training niches)

### **4\) Message \+ artifact semantics**

ACP supports “rich/multimodal messages” and artifact generation patterns. ([Agent Communication Protocol](https://agentcommunicationprotocol.dev/introduction/welcome))  
Decide:

* What is a **message** vs an **artifact** (logs vs model files vs evaluation PDFs)  
* Metadata you require (job id, cost center, dataset hash, git commit, lineage)  
* How you stream logs/metrics (and how clients subscribe)

### **5\) State and sessions**

ACP supports **stateful agents** and even “distributed sessions” patterns. ([Agent Communication Protocol](https://agentcommunicationprotocol.dev/introduction/welcome))  
Define:

* What state is persisted (run config, auth context, dataset pointers)  
* Session lifetime rules (important for notebooks vs CI pipelines)

### **6\) Errors, trust, security**

ACP includes an error model and production-grade concerns. ([Agent Communication Protocol](https://agentcommunicationprotocol.dev/introduction/welcome))  
You must define:

* Error taxonomy (OOM, dataset not found, image pull fail, quota exceeded, spot preemption)  
* AuthN/AuthZ model (org/project RBAC, per-run permissions)  
* Audit trail requirements (especially regulated niches)

---

## **C. A concrete “research → ACP” checklist you can reuse per niche**

For each niche, fill these 12 items:

1. Top 3 user personas \+ their “job-to-be-done”  
2. Workload types (dev/train/infer) \+ p50/p90 durations  
3. GPU/CPU/RAM/NVMe/network p50/p90  
4. Dataset sources \+ artifact destinations  
5. Framework/CUDA constraints \+ base images  
6. Failure/resume needs \+ checkpoint policy  
7. Observability needs (what metrics matter)  
8. Security/compliance requirements (isolation, residency, logging)  
9. Integrations (HF/W\&B/MLflow/CI) \+ auth patterns  
10. Pricing sensitivity \+ preferred billing model  
11. Required **agent capabilities** (verbs) and schemas (inputs/outputs)  
12. Run lifecycle states \+ streaming events \+ error taxonomy

When you have those, your ACP spec becomes almost mechanical: **manifest capabilities \+ run lifecycle \+ message/artifact conventions**.

If you want, pick **one niche** (e.g., “LLM fine-tuning for private corp data” or “RAG factories”), and I’ll show you what the filled template looks like and what the ACP “capabilities \+ run events” contract would be for it.

