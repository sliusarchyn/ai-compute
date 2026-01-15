# Top niches

### 16) [Enterprise Document AI (OCR → extraction → assistants)](https://docs.google.com/document/u/0/d/1t1xAbkQ7s9fLhYuUjZMgaNa_wD25iqHukPAF-u1dDNc/edit)

Why this is a great wedge:

* **Huge steady demand** (invoices, contracts, KYC, HR) \+ clear ROI → easier sales.  
* A lot of the pipeline is **CPU \+ storage** (OCR, parsing), so you can start with fewer expensive GPUs and still ship value.  
* Strong “platform lock-in”: schemas, eval suites, audit trails.  
   Market signal: IDP/Document AI spending and growth are consistently reported as rising in 2024–2025.

What you sell (node-first):

* **Batch doc pipeline workers** (CPU-heavy) \+ optional **GPU extraction/assistant endpoint** per tenant.  
* “Compliance-friendly” basics: encryption, RBAC, audit logs, retention controls.

### 17) [Internal Code Copilots (private company codebase)](https://docs.google.com/document/u/0/d/19hdSwC_ztNXBaZs2dBFUUleCtm6K07sNVvEUWivpFoQ/edit)

Why it fits:

* Customers will pay for **privacy \+ access control** \+ **repo-grounded answers** (and many can’t send code to external SaaS).  
* Technically it’s tractable on small infra: **indexing \+ embeddings \+ a few inference GPUs**.  
* You can implement an OpenAI-compatible API layer on top of high-throughput serving (vLLM-style) quickly.

What you sell:

* Dedicated “**Copilot pods**” (GPU) \+ “**indexer workers**” (CPU) per org.  
* Tight RBAC: repo/team-based permissions; audit trails.

### 1) [Private LLM fine-tuning (LoRA/QLoRA) for SMB/enterprise](https://docs.google.com/document/u/0/d/1qfZzFjIZJxzSQlVHY78T0_MULZRjdfaVOT59sO6MHMY/edit)

Why it fits:

* Clear dedicated-node story (“your data never leaves your box”).  
* Most customers start with **PEFT** (LoRA/QLoRA) rather than full training.  
* Usually **single-node** is enough for first customers (7B–70B adapters), which matches small cloud reality.

What you sell:

* “Fine-tune workspace” \+ “evaluation harness” \+ “deploy endpoint” as one flow.

### 3) [Stable Diffusion / image studios (fine-tune \+ batch generation)](https://docs.google.com/document/u/0/d/1cvo7p4t64UyDT4KE9IVNVAs00woZW_XUU_v5Rk1Hy2k/edit)

* Big prosumer/agency market, and lots of jobs can run on **single GPUs** (multi-GPU is mostly a speed optimization).

* Downside: more price-sensitive, more churn, lots of support noise.

