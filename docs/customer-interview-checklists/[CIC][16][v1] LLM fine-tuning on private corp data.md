# **(LLM fine-tuning on private corp data)**

### **0\) Context \+ persona (5 min)**

* What’s the org size and team (1 person, small ML team, enterprise ML platform)?

* Who runs jobs today (ML engineer, data engineer, DevOps)?

* What’s the “success” outcome: better answers, style control, compliance, cost, latency?

### **1\) Workload anatomy (capture numbers) (15–25 min)**

For the last 30–90 days:

* What % of jobs are: dev/test, SFT (LoRA/QLoRA), preference tuning, full fine-tune, batch eval, online inference?

* Typical model sizes used (7B/13B/34B/70B/other)?

* Typical run durations: **p50/p90** for each job type (rough is ok).

* Typical GPU request: 1 / 2 / 4 / 8 GPUs? How often multi-node?

* What blocks you today: queue time, OOM, slow storage, slow networking, cost?

* What’s your current **checkpoint frequency**? (minutes) and **resume rate** (% of runs resumed at least once)

* Dataset size: raw GB and/or token count; typical seq length; number of samples.

* What artifacts do you produce: LoRA adapters only, merged model, eval reports, logs?

**Hard numbers to collect (write down):**

* dataset\_size\_gb, tokens\_estimate, seq\_len

* gpu\_count, gpu\_vram\_needed

* run\_time\_p50/p90 by job type

* checkpoint\_interval\_min, checkpoint\_size\_gb

* success\_rate %, failure causes distribution

### **2\) Stack \+ environment constraints (10–15 min)**

* Frameworks: Transformers/Accelerate/TRL/DeepSpeed/FSDP? vLLM/TGI?

* Precision \+ quantization: fp16/bf16/fp8? QLoRA (4-bit)?

* Must-have CUDA/driver versions (or “whatever works”)?

* Container-only OK? If VM required: why (security, drivers, corporate policy)?

* Need internet? If yes: which domains (HF, W\&B, package mirrors)? If no: strict egress rules?

### **3\) Data governance \+ risk (10–15 min)**

* What’s inside the data: IP, PII, regulated (PHI/financial)?

* Requirements:

  * region lock (which regions)?

  * **dedicated node** required or “dedicated project/network is enough”?

  * encryption at rest required? customer-managed keys?

  * audit logs: who accessed what, retention period

* Logging constraints: can you store prompts/outputs? (usually “no”)

* Data ingress/egress: upload, S3/GCS, VPN/peering?

### **4\) Operational expectations (10–15 min)**

* What’s acceptable queue time / start time for dev vs training?

* Do you accept preemptible/spot? Under what conditions?

* SLO priorities: start latency, completion reliability, throughput, artifact durability

* Support expectation: Slack/Discord? response time? do they need onboarding?

### **5\) Workflow & integrations (10–15 min)**

* Launch method: notebook, CLI, CI, MLOps tool

* Tracking: W\&B / MLflow / custom dashboards

* Model registry: HF private, internal registry, S3, artifact store

* Auth: orgs/projects, RBAC roles, API keys, SSO later

### **Interview output template (1 page)**

* **Top 3 job types** \+ % share

* **p50/p90 durations** per job type

* **GPU shape**: common GPU count \+ VRAM tier

* **Dataset size** \+ checkpoint cadence/size

* **Must-have constraints**: region, isolation, egress, retention

* **Integrations**: W\&B/MLflow/HF/S3/GCS

* **Top pain points** ranked

