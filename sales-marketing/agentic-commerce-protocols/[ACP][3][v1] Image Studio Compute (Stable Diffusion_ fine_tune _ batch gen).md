# Image Studio Compute (Stable Diffusion: fine-tune + batch gen)

## What you sell (productized SKUs)

Keep it studio-friendly and simple:

### Core SKUs

* **StudioGen-1GPU**: single-GPU generation workstation/pod (most studios)
* **StudioGen-2GPU**: faster throughput, parallel batches
* **StudioTune-1GPU**: LoRA fine-tune workspace (single GPU)
* **StudioTune-2GPU**: faster LoRA, larger batches / higher-res workflows

### Add-ons

* **NVMe scratch** (datasets + caches + outputs)
* **Object storage bucket** (outputs delivery)
* **Extra public IP**
* **Support tier** (standard vs premium)
* **Outbound restriction mode** (open/restricted/blocked)

> Important: studios are price-sensitive, so ACP must support **hourly + monthly**, and **short commitments**.

---

## Parties

* Buyer agent: studio’s procurement bot/script/human
* Seller agent: your provisioning + billing + ops (human fallback)

---

## Core objects

* `offer`, `rfq`, `quote`, `order`
* `service_instance` (VM/node)
* `access` (SSH + optional HTTPS endpoint)
* `meter` (GPU hours, storage, egress)
* `invoice` / `payment`
* `change_order` (add GPUs/NVMe, extend time)
* `support_ticket` / `incident`
* `termination` / `wipe_evidence`

---

# Message envelope (required)

```json
{
  "acp_version": "0.1",
  "message_id": "uuid",
  "ts": "2026-01-15T10:00:00Z",
  "type": "command|event|query",
  "name": "offer.list|rfq.create|quote.issue|order.accept|fulfillment.update|meter.report|invoice.issue|...",
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
    "use_case": "image_studio_sd",
    "region": "us-east",
    "billing_models": ["hourly", "monthly"],
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
        "offer_id": "offer_studiogen_1gpu",
        "title": "StudioGen-1GPU (Batch Generation)",
        "region": ["us-east", "eu-central"],
        "resources": { "gpu": 1, "vcpus": 24, "ram_gb": 96, "nvme_gb": 1000 },
        "billing": {
          "models": [
            { "model": "hourly", "price_usd_per_hour": 1.75, "min_hours": 2 },
            { "model": "monthly", "price_usd_per_month": 2300, "min_term_days": 30 }
          ]
        },
        "sla": { "provision_p90_hours": 8, "support": "best-effort|business-hours" },
        "security": { "single_tenant": true, "ssh_keys_only": true, "wipe_between_customers": true }
      },
      {
        "offer_id": "offer_studiotune_1gpu",
        "title": "StudioTune-1GPU (LoRA Fine-tune)",
        "resources": { "gpu": 1, "vcpus": 32, "ram_gb": 128, "nvme_gb": 2000 },
        "billing": {
          "models": [
            { "model": "hourly", "price_usd_per_hour": 2.15, "min_hours": 4 },
            { "model": "monthly", "price_usd_per_month": 2900, "min_term_days": 30 }
          ]
        }
      }
    ]
  }
}
```

---

# 2) RFQ — Studio-specific requirements capture

Studios differ from enterprise niches:

* they need **fast provision**, **hourly**, **easy scaling**, **big NVMe**, and **simple output delivery**
* they often don’t want heavy compliance language, but they do want **IP protection** and **wipes**

## `rfq.create` (buyer → seller)

```json
{
  "name": "rfq.create",
  "type": "command",
  "payload": {
    "rfq_id": "rfq_sd_001",
    "use_case": "image_studio_sd",
    "workload": {
      "job_types": ["batch_generation", "lora_finetune"],
      "model_family": "sdxl|sd15|flux|custom",
      "generation": {
        "images_per_day": 25000,
        "resolution": "1024|1536|2048",
        "peak_concurrency": 4
      },
      "finetune": {
        "runs_per_month": 8,
        "dataset_images": 5000,
        "typical_run_hours": 2,
        "peak_run_hours": 6
      }
    },
    "requirements": {
      "tenancy": { "single_tenant": true },
      "region": "us-east",
      "network": {
        "inbound": [
          { "port": 22, "proto": "tcp", "source": "buyer_ip_ranges" }
        ],
        "outbound": { "mode": "open|restricted", "allowlist": ["civitai.com:443", "huggingface.co:443"] }
      },
      "storage": {
        "scratch_nvme_gb": 4000,
        "output_delivery": {
          "mode": "s3|gcs|sftp|http",
          "target_uri": "s3://buyer-bucket/outputs/"
        },
        "retention_days": 14
      },
      "environment": {
        "stack": "BYO",
        "seller_baseline": ["ubuntu_lts", "docker", "nvidia_container_toolkit"],
        "ui": "optional_webui" 
      },
      "support": { "tier": "standard|premium", "onboarding_hours": 2 }
    },
    "commercial": {
      "billing_model": "hourly",
      "expected_hours_per_week": 120,
      "budget_usd_per_month": 2500
    }
  }
}
```

---

# 3) Quote — pricing + assumptions

For studios, be very explicit about:

* hourly minimums
* egress billing
* storage retention
* “outputs belong to buyer”
* wipe policy

## `quote.issue` (seller → buyer)

```json
{
  "name": "quote.issue",
  "type": "event",
  "payload": {
    "quote_id": "q_sd_001",
    "rfq_id": "rfq_sd_001",
    "valid_until": "2026-01-18T00:00:00Z",
    "pricing": {
      "model": "hourly",
      "line_items": [
        { "sku": "StudioGen-1GPU", "qty": 1, "price_usd_per_hour": 1.75, "min_hours": 2 },
        { "sku": "NVMe-AddOn-2TB", "qty": 1, "price_usd_per_month": 250 },
        { "sku": "Support-Standard", "qty": 1, "price_usd_per_month": 200 }
      ],
      "estimates": {
        "hours_per_month": 480,
        "estimated_total_usd": 840
      }
    },
    "assumptions": [
      "Buyer deploys their SD stack (BYO) via SSH",
      "Outputs delivered to buyer storage target (recommended)",
      "Seller provides baseline OS + GPU runtime; not responsible for model quality"
    ],
    "sla": {
      "provision_target": "p90<=8h",
      "support_response": "best-effort (standard) / p90<=4h (premium)"
    },
    "terms": {
      "single_tenant": true,
      "wipe_on_termination": true,
      "ip_policy": "buyer_owns_outputs",
      "logging_policy": "no_prompt_or_image_content_by_default",
      "retention_days": 14
    }
  }
}
```

---

# 4) Order acceptance

## `order.accept` (buyer → seller)

```json
{
  "name": "order.accept",
  "type": "command",
  "payload": {
    "order_id": "ord_sd_001",
    "quote_id": "q_sd_001",
    "billing": { "method": "card|invoice", "net_days": 0 },
    "contacts": {
      "technical": { "name": "…", "email": "…" },
      "billing": { "name": "…", "email": "…" }
    },
    "ssh_keys": [
      { "label": "studio-admin", "public_key": "ssh-ed25519 AAAA..." }
    ],
    "network_inputs": {
      "buyer_ip_ranges": ["203.0.113.0/24"]
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
    "order_id": "ord_sd_001",
    "status": "CONFIRMED",
    "next": ["seller_provisioning"]
  }
}
```

---

# 5) Fulfillment (instance + access)

## `fulfillment.update` (seller → buyer)

```json
{
  "name": "fulfillment.update",
  "type": "event",
  "payload": {
    "order_id": "ord_sd_001",
    "service_instance": {
      "instance_id": "inst_sd_01",
      "role": "studio_generation",
      "kind": "vm|bare_metal",
      "region": "us-east",
      "resources": { "gpu": 1, "vcpus": 24, "ram_gb": 96, "nvme_gb": 3000 },
      "status": "PROVISIONING|READY|FAILED",
      "access": {
        "ssh": { "host": "203.0.113.30", "port": 22, "user": "ubuntu" }
      },
      "network_policy": {
        "inbound_sources": ["buyer_ip_ranges"],
        "outbound_mode": "open"
      },
      "security": { "single_tenant": true, "wipe_on_termination": true }
    },
    "ready_definition": [
      "SSH access works with provided keys",
      "GPU runtime OK (nvidia-smi healthy)",
      "Docker + GPU runtime ready"
    ]
  }
}
```

---

# 6) Metering + billing (hourly-friendly)

Studios care about:

* **GPU-hours**
* **egress** (image outputs)
* **storage** (outputs retention)

## `meter.report` (seller → buyer)

```json
{
  "name": "meter.report",
  "type": "event",
  "payload": {
    "period": { "from": "2026-01-15T00:00:00Z", "to": "2026-01-15T23:59:59Z" },
    "instance_id": "inst_sd_01",
    "usage": {
      "instance_hours": 18,
      "gpu_hours": 18,
      "bandwidth_gb_egress": 120,
      "storage_gb_day_avg": 850
    }
  }
}
```

## `invoice.issue` (seller → buyer)

(hourly can be daily/weekly invoice or prepaid balance—ACP doesn’t care)

```json
{
  "name": "invoice.issue",
  "type": "event",
  "payload": {
    "invoice_id": "inv_sd_2026_01_001",
    "order_id": "ord_sd_001",
    "amount_usd": 31.50,
    "lines": [
      { "sku": "StudioGen-1GPU", "qty_hours": 18, "rate_usd_per_hour": 1.75, "amount_usd": 31.50 }
    ]
  }
}
```

---

# 7) Change orders (scale for peak weeks)

Studios often need “burst” capacity.

## `change_order.create` (buyer → seller)

```json
{
  "name": "change_order.create",
  "type": "command",
  "payload": {
    "change_id": "chg_sd_001",
    "order_id": "ord_sd_001",
    "request": {
      "add": [{ "sku": "StudioGen-1GPU", "qty": 2 }],
      "add_nvme_gb": 2000,
      "support_tier": "premium"
    },
    "reason": "Campaign week burst rendering"
  }
}
```

## `change_order.quote` (seller → buyer)

```json
{
  "name": "change_order.quote",
  "type": "event",
  "payload": {
    "change_id": "chg_sd_001",
    "delta_pricing": {
      "hourly": [
        { "sku": "StudioGen-1GPU", "qty": 2, "price_usd_per_hour": 1.75 }
      ],
      "monthly_addons": [
        { "sku": "NVMe-AddOn-2TB", "price_usd_per_month": 250 },
        { "sku": "Support-Premium", "price_usd_per_month": 600 }
      ]
    },
    "eta_hours": 6
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
    "ticket_id": "t_sd_001",
    "severity": "S3",
    "instance_id": "inst_sd_01",
    "subject": "Slow generation after updating drivers",
    "details": "Need help validating CUDA/driver compatibility."
  }
}
```

## `incident.declare`

```json
{
  "name": "incident.declare",
  "type": "event",
  "payload": {
    "incident_id": "inc_sd_001",
    "severity": "S1",
    "impact": "Node down",
    "eta_update_minutes": 30
  }
}
```

---

# 9) Termination + wipe evidence

## `order.terminate`

```json
{
  "name": "order.terminate",
  "type": "command",
  "payload": {
    "order_id": "ord_sd_001",
    "requested_end": "2026-01-20",
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
    "order_id": "ord_sd_001",
    "terminated_at": "2026-01-20T23:00:00Z",
    "wipe": { "performed": true, "method": "reimage+discard", "evidence_id": "wipe_log_sd_987" }
  }
}
```

---

# Studio-specific RFQ fields you should standardize

To close deals fast, always ask for:

1. **Job mix**: % batch gen vs % fine-tune
2. **Resolution & volume**: images/day, peak bursts
3. **Model family**: SDXL/SD1.5/other (affects VRAM needs)
4. **Data & outputs**: dataset size, output delivery target (bucket/sftp)
5. **Billing preference**: hourly vs monthly, minimum commitment
6. **Network**: inbound IP allowlist, outbound open/restricted
7. **Support**: “we just need a box” vs “help us make it fast”
8. **IP protection**: wipe policy, access control expectations

---

# Minimal ACP subset (implement first)

For studios you can ship ACP with just:

1. `offer.list` / `offer.list.result`
2. `rfq.create`
3. `quote.issue`
4. `order.accept` / `order.confirmed`
5. `fulfillment.update` (→ READY)
6. `meter.report` + `invoice.issue` (hourly)
