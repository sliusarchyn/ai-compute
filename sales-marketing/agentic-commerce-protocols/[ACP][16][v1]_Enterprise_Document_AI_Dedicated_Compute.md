# Enterprise Document AI Dedicated Compute (Concierge / BYO Stack)

## Scope (what ACP covers)

**Commerce + fulfillment for “Document Processing Pods”**:

* Dedicated VM / bare metal with SSH
* Optional GPU
* Storage/network add-ons
* Support tier
* Security/tenancy commitments
* Billing (monthly or hourly)
* Change orders (scale up/down, add NVMe, add IPs)
* Support tickets + incidents

ACP does **not** mandate how workloads run. It only ensures the customer can reliably buy and operate capacity for Doc AI.

---

## Key objects

### Parties

* `buyer_org` — customer company
* `buyer_agent` — bot/user acting for buyer
* `seller_org` — you
* `seller_agent` — your system/ops + human fallback

### Commercial objects

* `offer` — your SKU description + constraints + pricing model
* `rfq` — request for quote (includes niche requirements)
* `quote` — priced, time-bounded proposal (with terms)
* `order` — accepted quote
* `contract` — MSA/SOW/DPA references + service terms
* `invoice` — billed amounts
* `payment` — payment status / method

### Fulfillment objects

* `service_instance` — “Doc Pod” assigned to customer
* `access` — SSH endpoint(s), keys, ports policy
* `meter` — usage counters (hours, bandwidth, storage)
* `change_order` — add/remove resources or upgrade tier
* `support_ticket` / `incident`

---

## Minimal “Doc Pod” SKU model (what you sell)

Keep SKUs purpose-built (not arbitrary sliders):

* **DocPod-CPU-S**: OCR/rendering/parsing (CPU + NVMe)
* **DocPod-CPU-M**: heavier batches + faster disk
* **DocPod-GPU-1**: extraction/embeddings/assistant endpoint (1 GPU)
* **Add-ons**: extra NVMe, extra public IP, private networking (later), support tier

---

# ACP envelope (common fields)

All ACP messages share this envelope.

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

# 1) Discovery (Offer Catalog)

## `offer.list` (buyer → seller)

```json
{
  "name": "offer.list",
  "type": "query",
  "payload": {
    "region": "eu-central",
    "constraints": { "single_tenant": true, "gpu_optional": true }
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
        "offer_id": "offer_docpod_cpu_s",
        "title": "DocPod-CPU-S (OCR/Parsing)",
        "region": ["eu-central", "us-east"],
        "resources": { "vcpus": 32, "ram_gb": 128, "nvme_gb": 1000, "gpu": 0 },
        "billing": { "model": "monthly", "price_usd": 1200, "min_term_days": 30 },
        "sla": { "provision_p90_hours": 24, "support": "business-hours" },
        "security": {
          "single_tenant": true,
          "ssh_keys_only": true,
          "wipe_between_customers": true,
          "data_residency": true
        },
        "limits": { "max_public_ips": 2 }
      }
    ]
  }
}
```

---

# 2) RFQ (Request for Quote) — niche-specific payload

This is where Document AI specifics belong. For your concierge MVP, RFQ is how you learn what they need **without building a platform yet**.

## `rfq.create` (buyer → seller)

```json
{
  "name": "rfq.create",
  "type": "command",
  "payload": {
    "rfq_id": "rfq_001",
    "use_case": "enterprise_document_ai",
    "workload": {
      "doc_types": ["invoices", "contracts"],
      "formats": ["pdf", "png", "jpg"],
      "volume": { "pages_per_day": 200000, "peak_multiplier": 2.0 },
      "latency_mode": "batch",
      "pipeline_owner": "buyer", 
      "stack": "BYO"
    },
    "requirements": {
      "residency": "eu-central",
      "single_tenant": true,
      "network": {
        "inbound": [{ "port": 22, "proto": "tcp", "source": "buyer_ip_ranges" }],
        "outbound": "restricted|open",
        "integration": ["s3", "sftp"]
      },
      "storage": {
        "scratch_nvme_gb": 4000,
        "artifact_retention_days": 30
      },
      "compliance": {
        "pii": true,
        "logging": "no_content",
        "audit_required": true,
        "dpa_required": true
      },
      "support": { "tier": "standard|premium", "onboarding_hours": 6 }
    },
    "commercial": {
      "billing_model": "monthly",
      "term_days": 30,
      "budget_usd_per_month": 3000
    }
  }
}
```

---

# 3) Quote (pricing + commitments + assumptions)

## `quote.issue` (seller → buyer)

```json
{
  "name": "quote.issue",
  "type": "event",
  "payload": {
    "quote_id": "q_001",
    "rfq_id": "rfq_001",
    "valid_until": "2026-01-22T00:00:00Z",
    "line_items": [
      { "sku": "DocPod-CPU-M", "qty": 1, "price_usd_per_month": 1800 },
      { "sku": "NVMe-AddOn-2TB", "qty": 1, "price_usd_per_month": 300 },
      { "sku": "Support-Standard", "qty": 1, "price_usd_per_month": 400 }
    ],
    "total_usd_per_month": 2500,
    "assumptions": [
      "BYO stack installed by buyer; seller provides baseline OS + Docker",
      "Inbound only SSH from buyer IP ranges",
      "Artifacts stored in buyer bucket (recommended)"
    ],
    "sla": {
      "provision_target": "p90<=24h",
      "support_response": "p90<=8 business hours"
    },
    "terms": {
      "single_tenant": true,
      "data_residency": "eu-central",
      "wipe_on_termination": true,
      "log_policy": "no_content",
      "allowed_use": "document_ai_only"
    },
    "contract_refs": {
      "msa": "url-or-id",
      "dpa": "url-or-id",
      "sow": "url-or-id"
    }
  }
}
```

---

# 4) Order + Contract acceptance

## `order.accept` (buyer → seller)

```json
{
  "name": "order.accept",
  "type": "command",
  "payload": {
    "order_id": "ord_001",
    "quote_id": "q_001",
    "billing": { "method": "invoice|card", "net_days": 0 },
    "contacts": {
      "technical": { "name": "…", "email": "…" },
      "billing": { "name": "…", "email": "…" }
    },
    "ssh_keys": [
      { "label": "buyer-admin", "public_key": "ssh-ed25519 AAAA..." }
    ]
  }
}
```

## `order.confirmed` (seller → buyer)

```json
{
  "name": "order.confirmed",
  "type": "event",
  "payload": {
    "order_id": "ord_001",
    "status": "CONFIRMED",
    "next": [
      "seller_provisioning",
      "buyer_onboarding_call"
    ]
  }
}
```

---

# 5) Fulfillment (provisioning → access → ready)

## `fulfillment.update` (seller → buyer)

```json
{
  "name": "fulfillment.update",
  "type": "event",
  "payload": {
    "order_id": "ord_001",
    "service_instance": {
      "instance_id": "inst_001",
      "kind": "vm|bare_metal",
      "region": "eu-central",
      "resources": { "vcpus": 64, "ram_gb": 256, "nvme_gb": 4000, "gpu": 0 },
      "status": "PROVISIONING|READY|FAILED",
      "access": {
        "ssh": {
          "host": "203.0.113.10",
          "port": 22,
          "user": "ubuntu",
          "allowed_sources": ["buyer_ip_ranges"]
        }
      },
      "security": {
        "single_tenant": true,
        "disk_wipe_policy": "wipe_on_termination",
        "logging_policy": "no_content"
      }
    }
  }
}
```

**Definition of READY (important):**

* OS booted, SSH works with provided keys
* baseline packages installed (as per offer)
* firewall rules applied
* monitoring agent installed (internal)

---

# 6) Usage + Billing (simple MVP)

Even if you don’t have full metering yet, ACP defines it now.

## `meter.report` (seller → buyer, periodic)

```json
{
  "name": "meter.report",
  "type": "event",
  "payload": {
    "period": { "from": "2026-02-01", "to": "2026-02-29" },
    "instance_id": "inst_001",
    "usage": {
      "instance_hours": 672,
      "storage_gb_month": 4000,
      "bandwidth_gb": 1200
    },
    "notes": ["Bandwidth includes egress only"]
  }
}
```

## `invoice.issue` (seller → buyer)

```json
{
  "name": "invoice.issue",
  "type": "event",
  "payload": {
    "invoice_id": "inv_2026_02_001",
    "order_id": "ord_001",
    "amount_usd": 2500,
    "due_date": "2026-03-05",
    "lines": [
      { "sku": "DocPod-CPU-M", "amount_usd": 1800 },
      { "sku": "NVMe-AddOn-2TB", "amount_usd": 300 },
      { "sku": "Support-Standard", "amount_usd": 400 }
    ]
  }
}
```

---

# 7) Change orders (scale up/down)

Document AI demand spikes; change orders must be first-class.

## `change_order.create` (buyer → seller)

```json
{
  "name": "change_order.create",
  "type": "command",
  "payload": {
    "change_id": "chg_001",
    "instance_id": "inst_001",
    "request": {
      "add_nvme_gb": 2000,
      "upgrade_support": "premium"
    },
    "reason": "monthly backfill peak"
  }
}
```

## `change_order.quote` (seller → buyer)

```json
{
  "name": "change_order.quote",
  "type": "event",
  "payload": {
    "change_id": "chg_001",
    "delta_usd_per_month": 550,
    "eta_hours": 12,
    "requires_reboot": true
  }
}
```

---

# 8) Support + incidents (commerce-grade expectations)

## `support.ticket.create` (buyer → seller)

```json
{
  "name": "support.ticket.create",
  "type": "command",
  "payload": {
    "ticket_id": "t_001",
    "severity": "S2",
    "instance_id": "inst_001",
    "subject": "Disk filling faster than expected",
    "details": "Need guidance on scratch vs artifact storage"
  }
}
```

## `incident.declare` (seller → buyer)

```json
{
  "name": "incident.declare",
  "type": "event",
  "payload": {
    "incident_id": "inc_001",
    "severity": "S1",
    "impact": "Instance unreachable",
    "eta_update_minutes": 30
  }
}
```

---

# 9) Termination / wipe / offboarding

## `order.terminate` (buyer → seller)

```json
{
  "name": "order.terminate",
  "type": "command",
  "payload": {
    "order_id": "ord_001",
    "requested_end": "2026-03-31",
    "data_export_confirmed": true
  }
}
```

## `fulfillment.terminated` (seller → buyer)

```json
{
  "name": "fulfillment.terminated",
  "type": "event",
  "payload": {
    "instance_id": "inst_001",
    "terminated_at": "2026-03-31T23:00:00Z",
    "wipe": { "performed": true, "method": "reimage+discard", "evidence_id": "wipe_log_abc" }
  }
}
```

---

# What makes this ACP “Document AI specific” (and useful for sales)

Your RFQ/Quote should always capture these **Doc AI knobs**:

1. **Volume**: pages/day + peak multiplier
2. **Formats**: scanned vs digital PDFs (drives CPU needs)
3. **Pipeline ownership**: BYO stack (MVP) vs managed later
4. **Residency + PII**: region lock, logging constraints, DPA requirement
5. **Integration**: where docs come from and outputs go (S3/SFTP/DB)
6. **Support**: onboarding hours + response expectations
7. **Success criteria**: throughput target + stability target (even if you don’t guarantee accuracy)

That’s how you sell “hardware as a service” credibly in this niche.

---

# MVP implementation advice (so ACP is real, not theoretical)

For your first 2–5 customers, ACP can be:

* a **JSON form** + **email/Slack** transport, where your ops team is the seller agent
* later, the exact same messages can be served via API/UI

**Minimum messages to implement first (ACP v0 core):**

1. `offer.list` / `offer.list.result`
2. `rfq.create`
3. `quote.issue`
4. `order.accept` / `order.confirmed`
5. `fulfillment.update` (PROVISIONING → READY)
6. `invoice.issue`

Everything else can be phased in.
