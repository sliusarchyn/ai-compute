# **Scientific computing / climate / physics surrogates**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: small grid runs, unit/regression tests, pipeline sanity  
* **HPC simulation runs**: CFD, weather/climate models, PDE solvers, Monte Carlo  
* **Pre/post-processing**: mesh generation, regridding, feature extraction, visualization  
* **Surrogate model training**: neural operators, U-Nets, transformers on gridded data  
* **Surrogate inference**: fast emulation for ensembles / uncertainty quantification  
* **Ensemble sweeps**: many runs across params/seeds (common)  
* **Data assimilation** (optional): combine observations \+ models (often compute-heavy)  
* **Eval/regression**: physics constraints, conservation errors, skill scores

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (small runs) | 1–50 cases | 0–1×24GB → 1×48GB | 16–64 vCPU, 64–512GB | 0.5–4TB | 2–12h → 1–7 days | Lots of restarts; small checkpoints; debug logs |
| HPC simulation (single node) | medium grids | 0–1×24GB → 2×48GB | 32–256 vCPU, 128GB–2TB | 1–8TB | 6–48h → 1–2 weeks | Writes checkpoints; periodic outputs; I/O bursts |
| HPC simulation (multi-node) | large grids | 4×80GB → 64×80GB (cluster) | 256–4096 vCPU, 1–32TB | 10–500TB (cluster) | 1–2 weeks → months | Heavy interconnect (MPI); large checkpoint files; parallel I/O |
| Pre/post-processing | TB–PB datasets | 0 GPUs → 1×24GB | 64–1024 vCPU, 256GB–8TB | 2–64TB | 6–48h → weeks | Very I/O heavy; regridding; formats conversion |
| Surrogate training (neural operator/CV) | 10TB–PB data | 1×80GB → 16×80GB | 64–1024 vCPU, 256GB–8TB | 4–128TB | 2–14 days → 1–6 months | High-throughput reads; big checkpoints; DDP/FSDP |
| Surrogate inference (ensembles) | 1k–1M runs | 1×24GB → 8×80GB | 16–512 vCPU, 64GB–2TB | 1–32TB | seconds–minutes/run → hours–days | Read model \+ inputs; write outputs; embarrassingly parallel |
| Ensemble sweeps (sim or surrogate) | 100–1M jobs | 0–1×24GB → 8×80GB | 32–1024 vCPU, 128GB–8TB | 2–64TB | 6h–days → weeks | Scheduler-heavy; artifact explosion; metadata tracking |
| Data assimilation (optional) | large state vectors | 1×48GB → 16×80GB | 64–1024 vCPU, 256GB–8TB | 2–64TB | 1–7 days → weeks | Mix of dense linear algebra \+ I/O; checkpointing |

### **Parallelism expectations**

* **Classical HPC**: MPI/OpenMP, multi-node scaling; network \+ parallel FS matter.  
* **Surrogates**: DDP/FSDP (deep learning) \+ data parallel ingestion.  
* **Ensembles**: embarrassingly parallel job farms; scheduler and metadata become bottlenecks.

### **Failure tolerance**

* Long MPI jobs need robust checkpoint/restart; failures are expensive.  
* Ensembles should be idempotent per `case_id`/`seed` and tolerant to partial completion.  
* Spot/preemptible is fine for ensemble farms, risky for tightly coupled multi-node MPI unless you have strong checkpointing.

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined later via telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **HPC**: MPI, OpenMP, domain solvers; job schedulers  
* **DL surrogates**: PyTorch/JAX; distributed training frameworks  
* **Data formats**: NetCDF/HDF5/Zarr; regridding tools; domain-specific libs  
* **Visualization**: large-scale postproc; sometimes GPU visualization

### **CUDA/driver constraints**

* GPU needs depend on surrogate training/inference; stable CUDA 12.x baseline.  
* HPC apps may require specific compiler/MPI stacks—version pinning matters.

### **Container vs VM requirements**

* HPC users often prefer:  
  * containers with specialized MPI support  
  * or VMs with tuned kernel/driver stacks  
* Some apps require high-performance interconnect integration; in MVP you can focus on single-node and small clusters.

### **Storage expectations**

* Local NVMe for scratch and fast staging.  
* For big workloads, a shared parallel FS-like behavior is expected; early on you can use object storage \+ caching but it won’t satisfy top-end HPC.  
* Checkpoint storage must be fast and durable.

### **Networking needs**

* Multi-node MPI wants low-latency, high-bandwidth networking (RDMA-class).  
* For MVP: provide strong single-node boxes and clear limitations on multi-node.

**Output artifact:** reference runtime image set

* `hpc-base-mpi` (compiler \+ MPI)  
* `hpc-solver` (solver-specific images)  
* `surrogate-train-cu12`  
* `surrogate-infer-cu12`  
* `postproc-regrid`  
* `ensemble-runner`  
* `compat`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Often not personal, but can be:  
  * proprietary industrial data (CFD, materials)  
  * government-sensitive climate/defense work  
* IP protection is key.

### **Common requirements**

* Region lock / residency  
* Isolation (dedicated projects/nodes for sensitive customers)  
* Encryption at rest \+ TLS in transit  
* Reproducibility lineage for published results

### **Logging rules**

* Don’t log raw datasets; store metadata: case IDs, versions, parameters, timings.  
* Allow export of provenance manifests for papers/audits.

**Output artifact:** security checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* Time-to-solution (queue time \+ runtime)  
* Scaling efficiency (multi-node) for those who need it  
* I/O reliability (checkpoint storms kill runs)  
* Reproducibility (exact solver versions, compiler flags, seeds)  
* Cost visibility for ensembles

### **SLO starting points (priors)**

* Provision/start p90 \< 10–20 min (dev)  
* Batch completion reliability \>99% for ensemble jobs  
* Checkpoint durability: never lose checkpoints once written  
* Observability: per-job throughput, I/O wait, network utilization

### **Support playbook themes**

* “MPI scaling is bad” (network/interconnect limitations)  
* “I/O bottleneck” (checkpoint storms, parallel I/O misconfig)  
* “Different results” (version drift, nondeterminism)  
* “Job died after 5 days” (need better checkpoint cadence)  
* “Storage costs exploded” (retention for checkpoints)

### **Pricing sensitivity & billing preference**

* HPC users prefer reservations/allocations.  
* Ensembles can be spot-friendly with cheap compute tiers.  
* Storage and egress can dominate budgets; need clear pricing.

**Output artifact:** SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Prepare case configs → run solver → postprocess → publish outputs  
* Train surrogates on simulation archives → validate → run ensembles fast  
* Continuous regression for solver \+ surrogate versions

### **Auth patterns**

* Org/project RBAC; service accounts for automation  
* Audit logs for sensitive work

### **Artifact flow**

* Inputs: meshes/grids, boundary conditions, observations, solver configs  
* Outputs: fields, trajectories, checkpoints, surrogate models, evaluation reports, provenance manifests  
* Destinations: customer buckets, HPC archives, publications repos

### **Golden flows (end-to-end)**

1. **Solver run \+ postproc**  
   * Stage inputs → run solver → checkpoint → postprocess/regrid → export results \+ provenance  
2. **Surrogate training \+ deployment**  
   * Curate dataset → train distributed → evaluate skill metrics → version model → deploy for inference  
3. **Ensemble forecasting**  
   * Generate parameter grid → run surrogate inference farm → aggregate uncertainty metrics → publish dashboards

