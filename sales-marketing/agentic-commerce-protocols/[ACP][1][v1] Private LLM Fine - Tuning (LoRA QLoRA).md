# Private LLM Fine-Tuning (LoRA/QLoRA)

## What you sell (productized, not custom)

Start with **2–3 bundles** that match LoRA/QLoRA reality:

* **TunePod-1GPU**: single GPU workspace (most LoRA/QLoRA starts here)
* **TunePod-2GPU**: faster runs / bigger batch / some 30B-ish adapters
* **TunePod-4GPU**: serious throughput, bigger evals, heavier pipelines

Optional add-ons:

* **Eval Worker (CPU)** (or extra GPU-hours) for batch eval
* **Artifact storage** (object storage bucket)
* **Outbound restrictions** (for privacy)
* **Support tier** (onboarding + debugging OOM/throughput)

---

## Parties

* Buyer agent: customer procurement bot/script/human
* Seller agent: your provisioning + billing system + ops human fallback

---

## Core objects

* `offer` — SKU bundles + constraints
* `rfq` — requirements capture (model size, dataset size, privacy, runtime)
* `quote` — price + assumptions + terms (DPA, residency, wipe)
* `order` — acceptance
* `service_instance` — provisioned workspace(s)
* `access` — SSH and optional endpoints
* `meter` — instance hours, GPU hours, storage, egress
* `invoice` / `payment`
* `change_order` — scale, add GPU, extend term
* `support_ticket` / `incident`
* `termination` / `wipe_evidence`

---

# Message Envelope (required)

```json
{
  "acp_version": "0.1",
  "message_id": "uuid",
  "ts": "2026-01-15T10:00:00Z",
  "type": "command|event|query",
  "name": "offer.list|rfq.create|quote.issue|order.accept|fulfillment.update|invoice.issue|...",
  "buyer_org_id": "org_buyer_123",
  "seller_org_id": "org_seller_001",
  "actor": { "type": "buyer_agent|seller_agent|human", "id": "agt_..." },
  "idempotency_key": "optional-string",
  "payload": {}
}
```

---

# 1) Offer Discovery (Catalog)

## `offer.list` (buyer → seller)

```json
{
  "name": "offer.list",
  "type": "query",
  "payload": {
    "use_case": "private_llm_finetuning",
    "region": "eu-central",
    "constraints": { "single_tenant": true, "gpu_required": true }
  }
}
```

## `offer.list.result` (seller → buyer)

```json
{
  "name": "offer.list.result",
  "type": "event",
  "payload": {
    "offers": [
      {
        "offer_id": "offer_tunepod_1gpu",
        "title": "TunePod-1GPU (LoRA/QLoRA Workspace)",
        "region": ["eu-central", "us-east"],
        "resources": { "gpu": 1, "vcpus": 32, "ram_gb": 128, "nvme_gb": 2000 },
        "billing": { "model": "monthly", "price_usd": 3200, "min_term_days": 30 },
        "sla": { "provision_p90_hours": 24, "support": "business-hours" },
        "security": {
          "single_tenant": true,
          "ssh_keys_only": true,
          "wipe_between_customers": true,
          "no_telemetry": true
        },
        "network": {
          "private_network": "optional",
          "outbound_modes": ["open", "restricted", "blocked"]
        }
      },
      {
        "offer_id": "offer_tunepod_2gpu",
        "title": "TunePod-2GPU (Faster LoRA/QLoRA)",
        "resources": { "gpu": 2, "vcpus": 48, "ram_gb": 256, "nvme_gb": 4000 },
        "billing": { "model": "monthly", "price_usd": 5800, "min_term_days": 30 }
      }
    ]
  }
}
```

---

# 2) RFQ — Fine-tuning specific requirements capture

This is where you encode what matters commercially and operationally:

* model size + base model source
* dataset size/type
* expected runtime
* privacy/network restrictions
* artifact expectations
* evaluation needs

## `rfq.create` (buyer → seller)

```json
{
  "name": "rfq.create",
  "type": "command",
  "payload": {
    "rfq_id": "rfq_tune_001",
    "use_case": "private_llm_finetuning",
    "workload": {
      "method": "lora|qlora",
      "base_model": {
        "name": "llama-3.x|mistral|qwen|custom",
        "size": "7B|13B|34B|70B",
        "source": "huggingface|buyer_registry|airgapped_media"
      },
      "dataset": {
        "type": "text|chat|code|docs",
        "size_gb": 80,
        "samples_million": 2.5,
        "contains_pii": true
      },
      "training": {
        "expected_runs_per_month": 6,
        "typical_run_hours": 8,
        "peak_run_hours": 20,
        "checkpointing": "required",
        "resume_from_checkpoint": true
      },
      "evaluation": {
        "required": true,
        "benchmarks": ["custom_eval", "ragas_optional"],
        "expected_eval_hours_per_run": 2
      }
    },
    "requirements": {
      "tenancy": { "single_tenant": true },
      "data_residency": "eu-central",
      "network": {
        "inbound": [
          { "port": 22, "proto": "tcp", "source": "buyer_ip_ranges" }
        ],
        "outbound": {
          "mode": "restricted",
          "allowlist": [
            "huggingface.co:443",
            "datasets.company.tld:443",
            "wandb.company.tld:443"
          ]
        }
      },
      "storage": {
        "scratch_nvme_gb": 3000,
        "artifact_bucket": "s3://buyer-bucket/finetune/",
        "retention_days": 30
      },
      "compliance": {
        "dpa_required": true,
        "logging_policy": "no_training_data_in_logs",
        "audit_required": true
      },
      "environment": {
        "stack": "BYO",
        "seller_baseline": ["ubuntu_lts", "docker", "nvidia_container_toolkit"],
        "cuda": "12.x",
        "frameworks": ["pytorch"]
      },
      "support": { "tier": "standard|premium", "onboarding_hours": 8 }
    },
    "commercial": {
      "billing_model": "monthly",
      "term_days": 30,
      "budget_usd_per_month": 6000
    }
  }
}
```

---

# 3) Quote — pricing + assumptions + governance terms

Fine-tuning quotes must spell out:

* what you guarantee (provision time, access, wipe)
* what you don’t (model quality)
* how data is handled
* outbound/network policy
* artifact ownership and retention

## `quote.issue` (seller → buyer)

```json
{
  "name": "quote.issue",
  "type": "event",
  "payload": {
    "quote_id": "q_tune_001",
    "rfq_id": "rfq_tune_001",
    "valid_until": "2026-01-22T00:00:00Z",
    "line_items": [
      { "sku": "TunePod-2GPU", "qty": 1, "price_usd_per_month": 5800 },
      { "sku": "Support-Standard", "qty": 1, "price_usd_per_month": 600 }
    ],
    "total_usd_per_month": 6400,
    "assumptions": [
      "Buyer supplies training code/containers and datasets (BYO stack)",
      "Artifacts stored to buyer bucket; local NVMe used as scratch",
      "Seller provides baseline OS + CUDA driver stack + Docker runtime",
      "No guarantee of model quality/accuracy; infra only"
    ],
    "sla": {
      "provision_target": "p90<=24h",
      "support_response": "p90<=8 business hours"
    },
    "terms": {
      "single_tenant": true,
      "data_residency": "eu-central",
      "wipe_on_termination": true,
      "outbound_policy": "restricted_allowlist",
      "log_policy": "no_training_data_in_logs",
      "audit_events": ["access", "admin_actions", "reimage", "network_policy_changes"]
    },
    "contract_refs": {
      "msa": "ref_msa_v1",
      "dpa": "ref_dpa_v1",
      "sow": "ref_sow_finetune_v1"
    }
  }
}
```

---

# 4) Order acceptance (and required provisioning inputs)

## `order.accept` (buyer → seller)

```json
{
  "name": "order.accept",
  "type": "command",
  "payload": {
    "order_id": "ord_tune_001",
    "quote_id": "q_tune_001",
    "billing": { "method": "invoice|card", "net_days": 0 },
    "contacts": {
      "technical": { "name": "…", "email": "…" },
      "security": { "name": "…", "email": "…" },
      "billing": { "name": "…", "email": "…" }
    },
    "ssh_keys": [
      { "label": "buyer-admin", "public_key": "ssh-ed25519 AAAA..." }
    ],
    "network_inputs": {
      "buyer_ip_ranges": ["203.0.113.0/24"],
      "outbound_allowlist": [
        "huggingface.co:443",
        "datasets.company.tld:443",
        "wandb.company.tld:443"
      ]
    }
  }
}
```

## `order.confirmed` (seller → buyer)

```json
{
  "name": "order.confirmed",
  "type": "event",
  "payload": {
    "order_id": "ord_tune_001",
    "status": "CONFIRMED",
    "next": ["seller_provisioning", "onboarding_call"]
  }
}
```

---

# 5) Fulfillment (provision workspace + access)

## `fulfillment.update` (seller → buyer)

```json
{
  "name": "fulfillment.update",
  "type": "event",
  "payload": {
    "order_id": "ord_tune_001",
    "service_instance": {
      "instance_id": "inst_tune_01",
      "role": "finetune_workspace",
      "kind": "vm|bare_metal",
      "region": "eu-central",
      "resources": { "gpu": 2, "vcpus": 48, "ram_gb": 256, "nvme_gb": 4000 },
      "status": "PROVISIONING|READY|FAILED",
      "access": {
        "ssh": { "host": "203.0.113.20", "port": 22, "user": "ubuntu" }
      },
      "network_policy": {
        "inbound_sources": ["buyer_ip_ranges"],
        "outbound_mode": "restricted",
        "outbound_allowlist": ["huggingface.co:443", "datasets.company.tld:443", "wandb.company.tld:443"]
      },
      "security": {
        "single_tenant": true,
        "no_telemetry": true,
        "logging_policy": "no_training_data_in_logs"
      }
    },
    "ready_definition": [
      "SSH access works with provided keys",
      "GPU driver stack OK (nvidia-smi healthy)",
      "Docker + GPU runtime ready",
      "Outbound allowlist enforced"
    ]
  }
}
```

---

# 6) Metering + billing (MVP)

Fine-tuning customers care about: GPU hours + storage + egress.

## `meter.report` (seller → buyer)

```json
{
  "name": "meter.report",
  "type": "event",
  "payload": {
    "period": { "from": "2026-02-01", "to": "2026-02-29" },
    "instance_id": "inst_tune_01",
    "usage": {
      "instance_hours": 672,
      "gpu_hours": 1344,
      "storage_gb_month": 4000,
      "bandwidth_gb_egress": 800
    },
    "notes": [
      "GPU hours = gpu_count * instance_hours",
      "Egress includes model pulls and artifact uploads"
    ]
  }
}
```

## `invoice.issue` (seller → buyer)

```json
{
  "name": "invoice.issue",
  "type": "event",
  "payload": {
    "invoice_id": "inv_2026_02_tune_001",
    "order_id": "ord_tune_001",
    "amount_usd": 6400,
    "due_date": "2026-03-05",
    "lines": [
      { "sku": "TunePod-2GPU", "qty": 1, "amount_usd": 5800 },
      { "sku": "Support-Standard", "qty": 1, "amount_usd": 600 }
    ]
  }
}
```

---

# 7) Change orders (scale, extend, add GPU, add storage)

## `change_order.create` (buyer → seller)

```json
{
  "name": "change_order.create",
  "type": "command",
  "payload": {
    "change_id": "chg_tune_001",
    "order_id": "ord_tune_001",
    "request": {
      "add_nvme_gb": 2000,
      "upgrade_gpu_count": 4,
      "extend_term_days": 30,
      "support_tier": "premium"
    },
    "reason": "More runs/month + larger base model eval"
  }
}
```

## `change_order.quote` (seller → buyer)

```json
{
  "name": "change_order.quote",
  "type": "event",
  "payload": {
    "change_id": "chg_tune_001",
    "delta_usd_per_month": 4200,
    "eta_hours": 24,
    "requires_reboot": true
  }
}
```

---

# 8) Support + incidents

## `support.ticket.create`

```json
{
  "name": "support.ticket.create",
  "type": "command",
  "payload": {
    "ticket_id": "t_tune_001",
    "severity": "S2",
    "instance_id": "inst_tune_01",
    "subject": "OOM during training",
    "details": "Need help tuning batch size, gradient checkpointing, and disk pressure."
  }
}
```

## `incident.declare`

```json
{
  "name": "incident.declare",
  "type": "event",
  "payload": {
    "incident_id": "inc_tune_001",
    "severity": "S1",
    "impact": "Instance unreachable",
    "eta_update_minutes": 30
  }
}
```

---

# 9) Termination + wipe evidence (privacy-critical)

## `order.terminate`

```json
{
  "name": "order.terminate",
  "type": "command",
  "payload": {
    "order_id": "ord_tune_001",
    "requested_end": "2026-03-31",
    "data_export_confirmed": true,
    "wipe_required": true
  }
}
```

## `fulfillment.terminated`

```json
{
  "name": "fulfillment.terminated",
  "type": "event",
  "payload": {
    "order_id": "ord_tune_001",
    "terminated_at": "2026-03-31T23:00:00Z",
    "wipe": { "performed": true, "method": "reimage+discard", "evidence_id": "wipe_log_123" }
  }
}
```

---

# Fine-tuning specific RFQ fields (standardize these)

So your sales process is fast and consistent:

1. **Method**: LoRA vs QLoRA
2. **Base model**: size + source (HF vs private registry vs airgapped)
3. **Dataset**: size, type, PII flags, residency constraints
4. **Run profile**: runs/month, typical hours, checkpoint frequency
5. **Evaluation**: required? how often? compute needs
6. **Outbound policy**: open vs restricted vs blocked
7. **Artifacts**: where to store, retention days
8. **Support**: onboarding hours + response expectations
9. **Success criteria**: infra SLOs (provision time, uptime), not model quality

---

# ACP v0 minimal subset (implement first)

If you want the smallest real ACP to run deals:

1. `offer.list` / `offer.list.result`
2. `rfq.create`
3. `quote.issue`
4. `order.accept` / `order.confirmed`
5. `fulfillment.update` (PROVISIONING → READY)
6. `invoice.issue`
