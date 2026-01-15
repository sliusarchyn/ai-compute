# Internal Code Copilots (Private Company Codebase)

## What ACP covers (commerce + fulfillment)

* Offer discovery (SKUs, regions, constraints)
* RFQ (requirements capture: repos, RBAC, network, privacy)
* Quote (pricing + assumptions + security terms)
* Order acceptance + payment/net terms
* Fulfillment (provision pods/nodes, deliver access endpoints)
* Metering + invoices
* Change orders (scale GPUs, add indexers, add repos)
* Support tickets + incidents
* Termination + wipe/offboarding

ACP does **not** define your copilot inference API or indexing pipeline mechanics. It defines how it’s **bought, provisioned, governed, and billed**.

---

## Parties

* **Buyer Agent**: customer procurement bot / script / person
* **Seller Agent**: your provisioning + billing system + human ops fallback

---

## Core objects

* `offer` — sellable SKU + constraints
* `rfq` — request for quote (codebase/cp requirements)
* `quote` — priced proposal, time-bounded, with terms
* `order` — accepted quote
* `service_instance` — provisioned resources (copilot GPU pod, indexer workers, optional vector store)
* `access` — endpoints, SSH, API base URL (if provided), allowed networks
* `meter` — usage counters
* `invoice` / `payment`
* `change_order`
* `support_ticket` / `incident`

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
    "use_case": "private_code_copilot",
    "region": "us-east",
    "constraints": {
      "single_tenant": true,
      "no_telemetry": true,
      "private_network_optional": true
    }
  }
}
```

## `offer.list.result` (seller → buyer)

Example SKUs for this niche:

* **Copilot GPU Pod** (inference)
* **Indexer Worker** (CPU)
* Optional **Vector Store** (dedicated per tenant, later managed)

```json
{
  "name": "offer.list.result",
  "type": "event",
  "payload": {
    "offers": [
      {
        "offer_id": "offer_copilot_gpu_1",
        "title": "CopilotPod-GPU-1 (Inference)",
        "region": ["us-east", "eu-central"],
        "resources": { "gpu": 1, "vcpus": 32, "ram_gb": 128, "nvme_gb": 2000 },
        "billing": { "model": "monthly", "price_usd": 3500, "min_term_days": 30 },
        "sla": { "provision_p90_hours": 24, "support": "business-hours" },
        "security": {
          "single_tenant": true,
          "ssh_keys_only": true,
          "wipe_between_customers": true,
          "no_telemetry": true
        },
        "network": {
          "inbound_default": [{ "port": 22, "proto": "tcp" }],
          "private_network": "optional",
          "outbound_modes": ["open", "restricted", "blocked"]
        }
      },
      {
        "offer_id": "offer_indexer_cpu_m",
        "title": "IndexerWorker-CPU-M (Repo Indexing)",
        "region": ["us-east", "eu-central"],
        "resources": { "gpu": 0, "vcpus": 64, "ram_gb": 256, "nvme_gb": 4000 },
        "billing": { "model": "monthly", "price_usd": 1800, "min_term_days": 30 }
      }
    ]
  }
}
```

---

# 2) RFQ (Request for Quote) — Code Copilot specific

This is where you capture the real constraints: repo sources, RBAC, outbound rules, audit expectations, and how “private” it must be.

## `rfq.create` (buyer → seller)

```json
{
  "name": "rfq.create",
  "type": "command",
  "payload": {
    "rfq_id": "rfq_copilot_001",
    "use_case": "private_code_copilot",
    "workload": {
      "users": { "active_devs": 80, "peak_concurrent": 25 },
      "repos": {
        "count": 220,
        "total_size_gb": 180,
        "languages": ["ts", "go", "java", "python"],
        "providers": ["github_enterprise", "gitlab_self_hosted"]
      },
      "traffic": {
        "mode": "interactive",
        "requests_per_min_peak": 600,
        "streaming": true
      }
    },
    "requirements": {
      "tenancy": { "single_tenant": true },
      "data_residency": "eu-central",
      "network": {
        "inbound": [
          { "port": 22, "proto": "tcp", "source": "buyer_ip_ranges" },
          { "port": 443, "proto": "tcp", "source": "buyer_vpn_ranges", "optional": true }
        ],
        "outbound": {
          "mode": "restricted",
          "allowlist": [
            "git.company.tld:443",
            "registry.company.tld:443",
            "sso.company.tld:443"
          ]
        },
        "private_network": true
      },
      "identity_access": {
        "auth_integration": "sso_saml|oidc|local",
        "rbac": "repo_team_scoped",
        "audit_required": true
      },
      "logging_policy": {
        "no_code_content_in_logs": true,
        "retain_days": 30
      },
      "deployment_model": {
        "stack": "BYO",
        "seller_baseline": ["ubuntu_lts", "docker", "nvidia_container_toolkit"],
        "api_compat": "openai_compatible_optional"
      },
      "support": { "tier": "standard|premium", "onboarding_hours": 10 }
    },
    "commercial": {
      "billing_model": "monthly",
      "term_days": 30,
      "budget_usd_per_month": 8000
    }
  }
}
```

---

# 3) Quote (pricing + assumptions + security terms)

## `quote.issue` (seller → buyer)

Key: quote must include **assumptions** and **explicit privacy/network commitments**.

```json
{
  "name": "quote.issue",
  "type": "event",
  "payload": {
    "quote_id": "q_copilot_001",
    "rfq_id": "rfq_copilot_001",
    "valid_until": "2026-01-22T00:00:00Z",
    "line_items": [
      { "sku": "CopilotPod-GPU-1", "qty": 2, "price_usd_per_month": 3500 },
      { "sku": "IndexerWorker-CPU-M", "qty": 1, "price_usd_per_month": 1800 },
      { "sku": "Support-Standard", "qty": 1, "price_usd_per_month": 600 }
    ],
    "total_usd_per_month": 9400,
    "assumptions": [
      "Customer provides model/runtime containers (BYO stack)",
      "Repo access via allowlisted outbound to customer Git endpoints",
      "No vendor telemetry; logs exclude code content by policy",
      "Artifacts (indexes) stored on tenant NVMe unless buyer requests external bucket"
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
      "log_policy": "no_code_content",
      "audit_events": ["access", "admin_actions", "repo_sync"]
    },
    "contract_refs": {
      "msa": "ref_msa_v1",
      "dpa": "ref_dpa_v1",
      "sow": "ref_sow_copilot_v1"
    }
  }
}
```

---

# 4) Order acceptance + provisioning inputs

## `order.accept` (buyer → seller)

```json
{
  "name": "order.accept",
  "type": "command",
  "payload": {
    "order_id": "ord_copilot_001",
    "quote_id": "q_copilot_001",
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
      "buyer_vpn_ranges": ["10.100.0.0/16"],
      "outbound_allowlist": ["git.company.tld:443", "registry.company.tld:443"]
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
    "order_id": "ord_copilot_001",
    "status": "CONFIRMED",
    "next": ["seller_provisioning", "onboarding_call"]
  }
}
```

---

# 5) Fulfillment (instances + access + readiness)

You’ll likely provision **2 logical components**:

* `copilot_pod` (GPU inference)
* `indexer_worker` (CPU indexing / repo sync)

## `fulfillment.update` (seller → buyer)

```json
{
  "name": "fulfillment.update",
  "type": "event",
  "payload": {
    "order_id": "ord_copilot_001",
    "service_instances": [
      {
        "instance_id": "inst_copilot_01",
        "role": "copilot_inference",
        "kind": "vm|bare_metal",
        "region": "eu-central",
        "resources": { "gpu": 1, "vcpus": 32, "ram_gb": 128, "nvme_gb": 2000 },
        "status": "PROVISIONING|READY|FAILED",
        "access": {
          "ssh": { "host": "203.0.113.10", "port": 22, "user": "ubuntu" },
          "api": { "base_url": "https://203.0.113.10:443", "optional": true }
        },
        "network_policy": {
          "inbound_sources": ["buyer_ip_ranges"],
          "outbound_mode": "restricted",
          "outbound_allowlist": ["git.company.tld:443", "registry.company.tld:443"]
        },
        "security": {
          "single_tenant": true,
          "no_telemetry": true,
          "logging_policy": "no_code_content"
        }
      },
      {
        "instance_id": "inst_indexer_01",
        "role": "repo_indexer",
        "kind": "vm|bare_metal",
        "resources": { "gpu": 0, "vcpus": 64, "ram_gb": 256, "nvme_gb": 4000 },
        "status": "PROVISIONING|READY|FAILED",
        "access": { "ssh": { "host": "203.0.113.11", "port": 22, "user": "ubuntu" } }
      }
    ]
  }
}
```

**READY definition (for this niche):**

* SSH works with provided keys
* baseline runtime installed (Ubuntu + Docker + GPU toolkit if needed)
* outbound allowlist enforced
* optional: a “connectivity check” to Git endpoints succeeds

---

# 6) Metering + invoices (MVP-friendly)

## `meter.report` (seller → buyer)

```json
{
  "name": "meter.report",
  "type": "event",
  "payload": {
    "period": { "from": "2026-02-01", "to": "2026-02-29" },
    "instances": [
      { "instance_id": "inst_copilot_01", "instance_hours": 672, "bandwidth_gb_egress": 420 },
      { "instance_id": "inst_indexer_01", "instance_hours": 672, "bandwidth_gb_egress": 180 }
    ],
    "storage": { "nvme_gb_month": 6000 },
    "notes": ["Egress includes model pulls + repo sync traffic (if external)"]
  }
}
```

## `invoice.issue` (seller → buyer)

```json
{
  "name": "invoice.issue",
  "type": "event",
  "payload": {
    "invoice_id": "inv_2026_02_copilot_001",
    "order_id": "ord_copilot_001",
    "amount_usd": 9400,
    "due_date": "2026-03-05",
    "lines": [
      { "sku": "CopilotPod-GPU-1", "qty": 2, "amount_usd": 7000 },
      { "sku": "IndexerWorker-CPU-M", "qty": 1, "amount_usd": 1800 },
      { "sku": "Support-Standard", "qty": 1, "amount_usd": 600 }
    ]
  }
}
```

---

# 7) Change orders (scale + add repos + upgrade support)

## `change_order.create` (buyer → seller)

```json
{
  "name": "change_order.create",
  "type": "command",
  "payload": {
    "change_id": "chg_copilot_001",
    "order_id": "ord_copilot_001",
    "request": {
      "add": [
        { "sku": "CopilotPod-GPU-1", "qty": 1 }
      ],
      "modify": [
        { "instance_id": "inst_indexer_01", "add_nvme_gb": 2000 }
      ],
      "support_tier": "premium"
    },
    "reason": "Onboarding 60 more devs + larger monorepo"
  }
}
```

## `change_order.quote` (seller → buyer)

```json
{
  "name": "change_order.quote",
  "type": "event",
  "payload": {
    "change_id": "chg_copilot_001",
    "delta_usd_per_month": 4200,
    "eta_hours": 24,
    "requires_reboot": false
  }
}
```

---

# 8) Support + incidents (important for “private copilot” trust)

## `support.ticket.create` (buyer → seller)

```json
{
  "name": "support.ticket.create",
  "type": "command",
  "payload": {
    "ticket_id": "t_copilot_001",
    "severity": "S2",
    "instance_id": "inst_copilot_01",
    "subject": "High latency during peak hours",
    "details": "Need guidance on batching / GPU saturation / scaling"
  }
}
```

## `incident.declare` (seller → buyer)

```json
{
  "name": "incident.declare",
  "type": "event",
  "payload": {
    "incident_id": "inc_copilot_001",
    "severity": "S1",
    "impact": "Inference pod unreachable",
    "eta_update_minutes": 30
  }
}
```

---

# 9) Termination + wipe (privacy-critical)

## `order.terminate` (buyer → seller)

```json
{
  "name": "order.terminate",
  "type": "command",
  "payload": {
    "order_id": "ord_copilot_001",
    "requested_end": "2026-03-31",
    "data_export_confirmed": true,
    "wipe_required": true
  }
}
```

## `fulfillment.terminated` (seller → buyer)

```json
{
  "name": "fulfillment.terminated",
  "type": "event",
  "payload": {
    "order_id": "ord_copilot_001",
    "terminated_at": "2026-03-31T23:00:00Z",
    "wipe": { "performed": true, "method": "reimage+discard", "evidence_id": "wipe_log_xyz" }
  }
}
```

---

# Copilot-specific “RFQ fields” you should standardize (so quotes are fast)

These are the knobs that matter commercially:

1. **Repo footprint**: count, size, monorepo vs many repos
2. **Git provider**: GitHub Enterprise / GitLab self-hosted / Bitbucket
3. **Auth**: SSO (SAML/OIDC) vs local
4. **Network**: outbound allowlist (Git + registry + SSO), private network requirement
5. **Privacy**: no telemetry, log policy “no code content”, audit requirements
6. **Concurrency**: peak concurrent users, streaming requirement
7. **Support expectation**: onboarding hours + response window
8. **Deployment model**: BYO stack now; “OpenAI-compatible endpoint” later

---

# ACP v0 minimal set (if you want the smallest implementable subset)

Implement just these first:

1. `offer.list` / `offer.list.result`
2. `rfq.create`
3. `quote.issue`
4. `order.accept` / `order.confirmed`
5. `fulfillment.update` (PROVISIONING → READY)
6. `invoice.issue`

Everything else can be phased in.
