# **Geospatial / satellite analytics**

## **1\) Workload anatomy (the “physics”)**

### **Typical job types you must support**

* **Dev/Test**: small AOIs (areas of interest), validate reprojection/tiling, quick model runs  
* **Data ingest**: pull imagery (satellite/drone), normalize formats, build catalogs  
* **Preprocessing**: reprojection, orthorectification, tiling/chipping, cloud masking, mosaics  
* **Batch inference**: detection/segmentation/classification over large AOIs/time ranges  
* **Change detection / time series**: compare across dates; seasonal normalization  
* **Indexing & vectorization**: build tile pyramids, vector outputs, spatial indexes  
* **Training / tuning**: fine-tune models per geography/sensor; domain adaptation  
* **Online inference (optional)**: interactive AOI queries, low-latency tile inference  
* **Eval/regression**: accuracy vs ground truth, drift by sensor/season, performance regression

### **Workload profile (p50 / p90)**

| Job type | Typical scale | GPU config (p50 → p90) | CPU/RAM (p50 → p90) | Local NVMe | Runtime (p50 → p90) | I/O pattern |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| Dev/Test (small AOI) | 1–1000 km² or 10–10k tiles | 0–1×24GB → 1×48GB | 16–64 vCPU, 64–512GB | 0.5–4TB | 1–6h → 1–3 days | Many small tile reads/writes; frequent reruns |
| Data ingest (imagery pull) | 10GB–10TB/day | 0 GPUs → 1×24GB | 32–256 vCPU, 128GB–2TB | 1–16TB | continuous / daily batches | Heavy downloads \+ writes; catalog updates; checksum/metadata |
| Preprocessing (reproj/tiling/mosaics) | 100GB–100TB/job | 0–1×24GB → 1×48GB | 64–512 vCPU, 256GB–4TB | 4–64TB | 6–48h → days–weeks | Extremely I/O heavy; large intermediate rasters; CPU-bound often |
| Batch inference (seg/det/class) | 10k–10B tiles | 1×24GB → 8×80GB | 32–512 vCPU, 128GB–2TB | 2–64TB | 0.1×–1× realtime → 1×–10× realtime | Read tiles → write masks/vectors; chunked outputs; batching critical |
| Change detection (time series) | 2–100 timepoints/AOI | 1×24GB → 4×48GB | 32–256 vCPU, 128GB–2TB | 2–32TB | 6–48h → days | Reads multiple timepoints; writes deltas; heavy joins/aggregation |
| Tile pyramids \+ indexing | national/global pyramids | 0 GPUs → 1×24GB | 64–512 vCPU, 256GB–4TB | 4–64TB | 12h–7 days → weeks | Write-heavy; many small files; benefits from locality |
| Training/tuning (geo vision) | 100k–1B chips | 1×48GB → 8×80GB | 64–512 vCPU, 256GB–4TB | 2–64TB | 2–14 days → 1–3 months | High-throughput reads; augmentation; big checkpoints |
| Online inference (interactive AOI) | 10–10k req/min | 0–1×24GB → 2×48GB | 16–128 vCPU, 64–512GB | 0.5–8TB | continuous | Tail latency sensitive; tile cache is key; small outputs |

### **Parallelism expectations**

* Preprocessing scales by **spatial partitioning** (tile grids) and CPU worker pools.  
* Inference scales by **tile batching** across many GPUs; often embarrassingly parallel.  
* Training scales via DDP; multi-node appears at scale.

### **Failure tolerance**

* Must be restartable/idempotent:  
  * by `aoi_id`, `tile_id`, `time_range`, `sensor_id`, `product_version`  
* Spot/preemptible is great for preprocessing/inference if you can resume by tile ranges.  
* Reproducibility requires pinned pipelines \+ deterministic reprojection/tiling parameters.

**Output artifact:** workload profile with p50/p90 resource \+ time numbers (table above), refined later via telemetry.

---

## **2\) Stack \+ environment constraints**

### **Required frameworks / tooling (typical)**

* **Geo raster tooling**: reprojection, mosaics, cloud masks, DEM usage  
* **Formats**: GeoTIFF/COG, JPEG2000, NetCDF/HDF (varies by sensor)  
* **Catalog/index**: STAC-like catalogs, spatial indexing, tile servers  
* **ML**: PyTorch for segmentation/detection; export to ONNX/TensorRT for serving if needed

### **CUDA/driver constraints**

* GPU needed for ML inference/training; stable CUDA 12.x baseline is fine.  
* Compat images for pinned stacks.  
* Pin GDAL/proj toolchain versions for reproducibility (geo libs are sensitive).

### **Container vs VM requirements**

* Containers are default.  
* VMs sometimes requested for specialized geo stacks or licensing constraints.

### **Storage expectations**

* Local NVMe for caching tiles and large intermediate rasters.  
* Durable object storage for:  
  * raw imagery  
  * processed products (COGs, tile pyramids)  
  * vector outputs  
  * models \+ manifests  
* Many-small-files problem is common (tile pyramids) → you need good object storage performance \+ batching.

### **Networking needs**

* High egress/ingress when pulling imagery from providers and pushing products.  
* Private networking sometimes required for gov/defense customers.

**Output artifact:** reference runtime image set

* `geo-ingest-catalog`  
* `geo-preproc-gdal`  
* `geo-infer-cu12`  
* `geo-change-cu12`  
* `geo-tiler-index`  
* `geo-train-cu12`  
* `geo-serve`  
* `compat`

---

## **3\) Data governance \+ risk**

### **Data sensitivity**

* Depends on customer:  
  * commercial land-use: moderate  
  * defense/critical infrastructure: high  
* Location intelligence can be sensitive even without personal data.

### **Common requirements**

* Region lock / residency  
* Isolation (dedicated projects/nodes for sensitive customers)  
* Encryption at rest \+ TLS in transit  
* Strict RBAC \+ audit logs  
* Retention and deletion policies

### **Logging rules**

* Don’t log raw tiles by default.  
* Store metadata: AOI ids, tile ranges, pipeline versions, timings, error codes.

**Output artifact:** security checklist \+ tenancy requirements.

---

## **4\) Operational expectations (what they’ll blame you for)**

### **What matters most**

* Throughput (km²/hour, tiles/sec)  
* I/O performance (preprocessing and pyramids are storage-bound)  
* Accuracy consistency across sensors/seasons  
* Reproducibility (same AOI \+ same pipeline \= same product)  
* Cost control (large AOIs become expensive fast)

### **SLO starting points (priors)**

* Job start/provisioning: p90 \< 10–20 min  
* Batch completion reliability: \>99% with resume/retry  
* Artifact durability: products never disappear  
* Observability: per-stage throughput \+ cache hit rate \+ error maps

### **Support playbook themes**

* “It’s slow” (GDAL/proj CPU bottleneck, storage saturation)  
* “Outputs shifted” (reprojection parameter/version drift)  
* “Tile pyramid explosion” (many small files; object storage limits)  
* “Accuracy dropped” (sensor drift, seasonal effects)  
* “Provider API issues” (ingest failures)

### **Pricing sensitivity & billing preference**

* Many want per-AOI/per-km² pricing, plus storage tiers  
* Spot acceptable for preprocessing/inference  
* Gov/enterprise prefer reservations and dedicated capacity

**Output artifact:** SLOs \+ support playbook \+ pricing notes.

---

## **5\) User workflow & integrations**

### **How they launch work today**

* Pick AOI \+ time range \+ sensor → run preprocessing → run inference → export products  
* Scheduled monitoring (weekly/monthly change detection)  
* Analysts use tile viewers \+ vector overlays

### **Auth patterns**

* Org/project RBAC, per-project permissions  
* Service accounts for automation  
* Audit logs for sensitive customers

### **Artifact flow**

* Inputs: raw imagery \+ metadata \+ DEMs \+ AOIs  
* Intermediates: mosaics, reprojection outputs, chip datasets  
* Outputs: COGs, masks, vectors, tiles, reports, run manifests  
* Destinations: customer bucket, map tile server, GIS tools

### **Golden flows (end-to-end)**

1. **AOI product generation**  
   * Define AOI/time/sensor → preprocess (COG \+ tiles) → infer → vectorize → export \+ manifest  
2. **Change detection monitoring**  
   * Schedule AOI snapshots → compare dates → generate change layers → publish dashboard  
3. **Fine-tune for a new region/sensor**  
   * Curate labeled chips → train/tune → regression suite across seasons → deploy to batch inference farm

