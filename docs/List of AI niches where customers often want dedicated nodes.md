# List of AI niches where customers often want dedicated nodes

Usually because of **data locality, compliance, predictable performance, or custom stacks**.

## [**1\) LLM fine-tuning for private corp data**](https://docs.google.com/document/u/0/d/1qfZzFjIZJxzSQlVHY78T0_MULZRjdfaVOT59sO6MHMY/edit)

* **Why dedicated:** sensitive documents, IP, compliance, predictable throughput.  
* **Pattern:** heavy VRAM \+ fast NVMe for datasets/checkpoints.  
* **Nodes:** 1×80GB for LoRA/QLoRA; 4–8×80GB for full/large tuning; separate inference nodes.

## [**2\) “RAG factories” (embedding \+ indexing at scale)**](https://docs.google.com/document/u/0/d/1rNELlUrIuFINgiENgr50ICV9u0TeRfnLdcMliBofrP8/edit)

* **Why dedicated:** huge corpora, long-running indexing jobs, big local storage.  
* **Pattern:** GPU for embeddings \+ **storage-heavy** nodes for vector DB / indexing.  
* **Nodes:** 1×24–48GB \+ 4–16TB NVMe; sometimes CPU-only \+ lots of RAM.

## [**3\) Stable Diffusion / image generation studios**](https://docs.google.com/document/u/0/d/1cvo7p4t64UyDT4KE9IVNVAs00woZW_XUU_v5Rk1Hy2k/edit)

* **Why dedicated:** creators want consistent speed \+ custom models \+ long batch jobs.  
* **Pattern:** many short jobs, occasional training (DreamBooth/LoRA).  
* **Nodes:** 1×24GB is common; 48GB for bigger batches; separate “render farm” pools.

## [**4\) Video generation / video understanding (ad tech, media, VFX)**](https://docs.google.com/document/u/0/d/1_FJxzE3RUMktbX_24PJREBwdVjd9UC5ALAJbGMYXxbc/edit)

* **Why dedicated:** huge I/O, big VRAM, long runs, expensive failures.  
* **Pattern:** multi-GPU training, high disk bandwidth, lots of CPU for decoding.  
* **Nodes:** 2–8 GPUs \+ very fast NVMe \+ high CPU; often 25–100GbE.

## [**5\) Speech AI (ASR/TTS/voice cloning *legit use*)**](https://docs.google.com/document/u/0/d/13YaGzPeN4Db_mYFU36rY8wQ7DEMg9PZwVELuC3Sb4Mo/edit)

* **Why dedicated:** latency \+ batching control \+ custom CUDA/audio dependencies.  
* **Pattern:** inference with GPU sharing; periodic fine-tuning.  
* **Nodes:** 1×24–48GB inference nodes; occasional 80GB for training.

## [**6\) Medical imaging (radiology/pathology)**](https://docs.google.com/document/u/0/d/1RobOjoPCrdWQPpRPb1NeOoOuMcNPa-tH3oGCtavvc5k/edit)

* **Why dedicated:** regulatory \+ patient data \+ auditability.  
* **Pattern:** CV segmentation/classification \+ very large images (WSI pathology).  
* **Nodes:** GPU \+ **high RAM** \+ storage; often isolated tenancy.

## [**7\) Bioinformatics & protein/genomics modeling**](https://docs.google.com/document/u/0/d/1fqHld9mY9dbaSzPP1K-J67QFOgZay5AnEbO0jnCxclo/edit)

* **Why dedicated:** big models, big datasets, long experiments; institutions demand isolation.  
* **Pattern:** training \+ inference; sometimes multi-node.  
* **Nodes:** 80GB GPUs \+ high RAM; storage-heavy for datasets.

## [**8\) Robotics & embodied AI (sim \+ policy training)**](https://docs.google.com/document/u/0/d/1xc-j6g2R7zaRLXEFOpUNRWYtCgWCANSOFCwPB4Nlrws/edit)

* **Why dedicated:** custom drivers/sim stacks, GPU \+ CPU mix, long-running sims.  
* **Pattern:** simulation is CPU-heavy; training is GPU-heavy.  
* **Nodes:** CPU-dense nodes \+ a few GPUs; sometimes multi-GPU for RL.

## [**9\) Autonomous driving / multi-sensor perception**](https://docs.google.com/document/u/0/d/1YR4Iic320REDGCbilraZUqFNv2y7B0mS6GMj91snxHo/edit)

* **Why dedicated:** massive video/lidar datasets \+ high bandwidth \+ strict reproducibility.  
* **Pattern:** multi-GPU training, heavy preprocessing, distributed pipelines.  
* **Nodes:** 4–8 GPUs, big NVMe, 100GbE common in serious setups.

## [**10\) Finance / quant research with ML**](https://docs.google.com/document/u/0/d/1LbVfIcc4ccpa5wtD--Gz9wmGj5kTG5j0aF-6eyUhqiI/edit)

* **Why dedicated:** proprietary signals \+ compliance \+ low-latency inference options.  
* **Pattern:** training bursts \+ steady inference.  
* **Nodes:** 1–4 GPUs \+ strong CPU \+ fast storage; isolated networking.

## [**11\) Cybersecurity ML (anomaly, malware classification, log AI)**](https://docs.google.com/document/u/0/d/1HercOURcoQpjVZM_4pZ8XKMnIloDbwiH6B5AAwmI4gE/edit)

* **Why dedicated:** sensitive telemetry \+ high ingest \+ custom pipelines.  
* **Pattern:** GPU for model work, CPU/RAM for streaming/log processing.  
* **Nodes:** mixed clusters (CPU-heavy \+ some GPUs), big RAM, fast disk.

## [**12\) Industrial inspection (factory vision)**](https://docs.google.com/document/u/0/d/1EUAi_Kg0tN0vyNZ1XyV6r5JrbBYWsbZWSIU2SK6IkSk/edit)

* **Why dedicated:** edge-like latency needs, deterministic throughput, sometimes on-prem preference.  
* **Pattern:** inference 24/7, occasional retraining on new defects.  
* **Nodes:** 1×24–48GB inference nodes; can benefit from GPU partitioning/sharing.

## [**13\) Geospatial / satellite analytics**](https://docs.google.com/document/u/0/d/1a4KYTbcpYWvNuxnGnehpJ3foeB2aX6GZe6w9PGKrT8s/edit)

* **Why dedicated:** gigantic raster data \+ heavy preprocessing.  
* **Pattern:** CV on large tiles \+ storage bandwidth constraints.  
* **Nodes:** GPU \+ lots of RAM \+ large NVMe; sometimes multi-node data pipelines.

## [**14\) Gaming / NPC intelligence / real-time agents**](https://docs.google.com/document/u/0/d/1T7Yp__tkLvgAW65sMEU7-86PGsan6uFANDzMrU3DtVw/edit)

* **Why dedicated:** latency targets and predictable performance for live services.  
* **Pattern:** many small inference calls; needs good concurrency control.  
* **Nodes:** inference-optimized GPUs (24–48GB) with strong networking.

## [**15\) E-commerce personalization / recommender training**](https://docs.google.com/document/u/0/d/1OSyF90-Yol3yYmBK-qXWjLtZ3P3SitevZkvgU84YBfg/edit)

* **Why dedicated:** huge private clickstream \+ heavy offline training.  
* **Pattern:** GPU training bursts \+ CPU feature pipelines.  
* **Nodes:** training pools (multi-GPU) \+ CPU/RAM feature nodes \+ storage.

## [**16\) Enterprise document AI (OCR \+ extraction \+ assistants)**](https://docs.google.com/document/u/0/d/1t1xAbkQ7s9fLhYuUjZMgaNa_wD25iqHukPAF-u1dDNc/edit)

* **Why dedicated:** compliance \+ private docs \+ stable throughput.  
* **Pattern:** mix of CV \+ LLM.  
* **Nodes:** 1×24–48GB for production; 80GB for heavier tuning.

## [**17\) Code models / internal copilots (company codebase)**](https://docs.google.com/document/u/0/d/19hdSwC_ztNXBaZs2dBFUUleCtm6K07sNVvEUWivpFoQ/edit)

* **Why dedicated:** source code is sensitive; governance demands isolation.  
* **Pattern:** RAG \+ fine-tuning \+ inference.  
* **Nodes:** 48–80GB nodes \+ storage-heavy indexing nodes.

## **18\) Scientific computing / climate / physics surrogates**

* **Why dedicated:** HPC-style workloads, multi-node training, high interconnect needs.  
* **Pattern:** distributed training \+ big datasets.  
* **Nodes:** 4–8 GPU nodes, fast networking, strong CPU/RAM ratios.

---

