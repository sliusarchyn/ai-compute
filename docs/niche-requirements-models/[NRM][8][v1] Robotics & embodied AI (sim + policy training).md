# **Robotics & embodied AI (sim \+ policy training)**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: environment setup, sim sanity, small rollouts, debugging sensors/rewards  
* **Simulation rollouts (data generation)**: run many episodes to collect trajectories  
* **Policy training (RL / imitation learning)**: PPO/SAC/BC, etc.  
* **Model training for perception**: vision models for robotic perception (seg/det/depth)  
* **Evaluation**: benchmark suites across seeds/envs, regression tests  
* **Real-world inference** (optional): run policy \+ perception in realtime (edge-like constraints)  
* **Dataset processing**: converting logs to training datasets, replay buffers

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (sim \+ policy sanity) | 1–50 episodes | 0–1×24GB → 1×48GB | 16–32 vCPU, 64–256GB | 0.5–2TB | 30–120 min → 6–24h | Lots of logs; frequent restarts; small checkpoints |
| Rollouts / data generation (sim) | 10k–10M steps/job | 0 GPUs → 1×24GB | 32–256 vCPU, 128GB–1TB | 1–8TB | 2–12h → 1–7 days | Write-heavy (trajectories, video frames); CPU-bound often |
| RL policy training (classic) | 1e7–1e10 steps | 1×24GB → 4×80GB | 32–256 vCPU, 128GB–2TB | 2–16TB | 6–48h → 1–4 weeks | Mixed: steady reads from replay \+ writes to checkpoints; lots of metrics |
| Imitation learning / BC | 10k–10M trajs | 1×24GB → 2×48GB | 16–128 vCPU, 64GB–1TB | 1–8TB | 2–24h → 1–7 days | Dataset streaming; periodic checkpoints; eval rollouts |
| Perception training (vision) | 100k–100M frames | 1×48GB → 8×80GB | 64–512 vCPU, 256GB–2TB | 2–32TB | 1–7 days → 2–8 weeks | High throughput reads; large checkpoints; augmentation heavy |
| Evaluation / regression | 100–100k episodes | 0–1×24GB → 1×48GB | 16–128 vCPU, 64GB–512GB | 0.5–4TB | 1–12h → 1–7 days | Bursty reads/writes; produces reports; many seeds |
| Realtime inference (optional) | 10–1k robots/agents | 0–1×24GB → 1×48GB | 8–64 vCPU, 32–256GB | 100–500GB | continuous | Tail latency sensitive; steady small reads; logs to durable store |

### **Parallelism expectations**

* **Rollouts** scale with **many CPU workers**; GPUs are optional unless using heavy visual rendering or large models.  
* **RL training** often uses:  
  * multiple env workers \+ one learner (single node)  
  * distributed setups for large scale (multi-node)  
* **Perception** is more like standard CV: DDP on multi-GPU nodes is common.

### **Failure tolerance**

* Rollouts: must be restartable by seed/env/episode; spot/preemptible is often acceptable.  
* Training: resume-from-checkpoint is required; RL can be sensitive to interruptions (but workable with frequent checkpoints).  
* Determinism matters for debugging: need reproducible seeds, pinned simulator versions.

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined later from interviews \+ telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **Simulators**: physics engines and simulators (MuJoCo/Isaac/Gazebo-like)  
* **RL libraries**: PPO/SAC style frameworks; custom codebases are common  
* **Rendering**: GPU rendering for photoreal sims; headless rendering support  
* **Distributed**: queues, parameter servers/actors, or Ray-like patterns (often used)

### **CUDA/driver constraints**

* If simulator uses GPU (rendering/Isaac), CUDA and driver versions matter a lot.  
* Provide stable base images and pin simulator dependencies tightly.

### **Container vs VM requirements**

* Containers are default, but robotics stacks may require:  
  * privileged access (GPU rendering, some sim dependencies)  
  * special kernel settings (less common in cloud)  
* Some teams prefer VMs for “workstation-like” dev environments.

### **Storage expectations**

* Local NVMe for:  
  * replay buffers  
  * trajectory datasets  
  * cached assets and simulator resources  
* Durable object storage for:  
  * run logs, checkpoints, dataset exports

### **Networking needs**

* Often heavy internal traffic for distributed RL (actors → learner).  
* For MVP, single-node “many-core \+ GPU” nodes cover most users.  
* For scale: low-latency networking becomes important.

**Output artifact:** reference runtime image set (base images \+ pinned versions)

* `robotics-sim` (sim engine \+ headless rendering)  
* `robotics-rollouts` (worker image)  
* `robotics-rl-train-cu12` (learner \+ GPU)  
* `robotics-perception-train-cu12`  
* `robotics-eval`  
* `compat` (fallback)

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Usually less regulated than medical/finance, but still IP-sensitive:  
  * proprietary robot designs, policies, sim assets  
  * real-world logs can include video of workplaces/people (privacy concerns)

### **Common requirements**

* Isolation for IP-heavy projects (dedicated nodes sometimes)  
* Encryption at rest \+ in transit  
* Access control for datasets/logs/model artifacts  
* Retention controls for video logs (privacy)

### **Logging rules**

* Don’t store raw camera feeds by default unless explicitly configured.  
* Store metadata: episode metrics, rewards, seeds, environment versions, errors.

**Output artifact:** security checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* **Throughput**: steps/sec or episodes/day  
* **Stability**: long RL runs without mysterious stalls  
* **Reproducibility**: same seeds \+ sim version produce same behavior  
* **Observability**: they want rich metrics (reward curves, success rate, FPS, GPU util)  
* **Cost**: RL can burn CPU/GPU for weeks

### **SLO starting points (priors)**

* Dev environments start ready: p90 \< 10–20 min  
* Long runs: \>99% stage reliability \+ automatic recovery/retry  
* Storage durability: checkpoints and logs never disappear  
* Clear progress reporting (steps/sec, episodes/sec, eval results)

### **Support playbook themes**

* “Sim is slow” (CPU bound, rendering config, asset loading)  
* “Training diverged” (hyperparams, nondeterminism, env bugs)  
* “Run stuck” (deadlocks in distributed actors, resource starvation)  
* “Disk full” (replay buffers and video logs explode)  
* “Different results” (version drift in sim or libraries)

### **Pricing sensitivity & billing preference**

* Researchers like cheap spot for rollouts \+ some reserved for learner stability  
* Teams often want “bundle pricing” per experiment (CPU-heavy \+ some GPU)  
* Enterprise wants reservations \+ predictable billing.

**Output artifact:** per-niche SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Iterative dev on a workstation → run experiments on a cluster  
* Batch sweeps across seeds/hyperparams  
* Pipeline: rollouts → train → eval → promote policy

### **Auth patterns**

* Org/project RBAC, API keys  
* Team sharing of environments, assets, checkpoints

### **Artifact flow**

* Inputs: simulator assets, env configs, seeds, datasets/replays  
* Outputs: policies/checkpoints, metrics, videos, eval reports, run manifests  
* Destinations: customer bucket \+ optional managed experiment tracking

### **Golden flows (end-to-end)**

1. **RL training loop**  
   * Spin sim workers → generate rollouts → learner updates policy → periodic eval → checkpoint \+ export metrics  
2. **Rollout farm \+ offline training**  
   * Massive rollout generation on CPU-heavy nodes → store replay → train policy offline on GPU node → evaluate \+ promote  
3. **Perception \+ policy co-training**  
   * Train perception model on multi-GPU node → integrate into sim → run policy training \+ regression suite

