# **BE Implementation Estimates (3 MVP Phases)**

**Assumptions**

* 1 senior BE engineer, MVP-quality (restart-safe, idempotent).  
* OpenStack reachable; BE integrates via Keystone \+ Heat (1 Deployment \= 1 Heat Stack).  
* WS is **simple**: FE loads via HTTP, subscribes to deployment updates; on reconnect FE refetches once.  
* Crypto top-up in MVP-3 uses **hosted invoice/checkout provider \+ webhooks** (no chain monitoring).  
* Point scale: **1 tiny • 2 small • 3 medium • 5 large • 8 very large • 13 huge/risky**

---

## **MVP-1 — Core Cloud (Auth \+ Deployments \+ OpenStack \+ WS)**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Platform foundation (NestJS structure, DB/migrations, config, logging, errors, health) | 16 / 24 / 40 | 3 |
| Auth (email/pass \+ verification \+ reset) | 20 / 32 / 50 | 5 |
| Customer/account basics (customerId, minimal settings, sessions list/revoke) | 10 / 18 / 30 | 3 |
| Catalog (regions/instance types/images/storage profiles, seed \+ read APIs) | 12 / 20 / 35 | 3 |
| OpenStack adapter v1 (Keystone token cache/refresh \+ Heat create/delete/get/events/outputs \+ error mapping) | 50 / 80 / 130 | 8 |
| Heat templates library v1 \+ validation (Ubuntu/GPU mapping, consistent outputs) | 16 / 28 / 50 | 3 |
| Deployments domain (CRUD, status model, idempotency keys) | 24 / 40 / 70 | 5 |
| Reconciler/worker (poll Heat, persist outputs, delete during provision, retries, restart safety) | 70 / 110 / 180 | 13 |
| WebSocket v1 (auth \+ subscribe/unsubscribe \+ push deployment.updated/event) | 14 / 22 / 40 | 3 |
| Integration tests \+ dev harness (create → active → delete) | 20 / 35 / 60 | 3 |
| **MVP-1 Total** | **252 / 409 / 685** | **36** |

---

## **MVP-2 — Monetization \+ Platform Basics (Balance \+ Card \+ Usage \+ API \+ Support)**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Ledger \+ balance model (append-only, balance compute, auditability) | 35 / 60 / 100 | 8 |
| Card top-up integration (provider adapter \+ intents/checkout) | 25 / 45 / 80 | 5 |
| Webhooks \+ reconciliation (verify \+ dedup \+ status mapping \+ basic refunds policy) | 20 / 35 / 60 | 5 |
| Usage metering v1 (ACTIVE time tracking, pricing calc, periodic debits) | 50 / 85 / 140 | 8 |
| Deployment gating (deny create if insufficient balance/limits, activity log entry) | 10 / 18 / 30 | 3 |
| Invoices history v1 (invoice records \+ line items; PDF later) | 16 / 28 / 50 | 3 |
| API keys \+ scopes \+ IP whitelist (customer API access) | 20 / 35 / 70 | 5 |
| Support tickets (create/list/messages/close) | 18 / 30 / 55 | 3 |
| API docs (OpenAPI \+ Postman export endpoint/process) | 10 / 18 / 35 | 3 |
| Tests (billing \+ metering \+ API keys \+ tickets) | 22 / 36 / 60 | 2 |
| **MVP-2 Total** | **226 / 390 / 680** | **45** |

---

## **MVP-3 — Expansion (Crypto via Hosted Invoice \+ Auto Top-up \+ Optional Extras)**

### **MVP-3 Lite (Payments expansion only)**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Crypto top-up via provider (create hosted invoice/checkout \+ redirect URL) | 8 / 14 / 25 | 3 |
| Crypto webhooks (signature verify \+ dedup \+ status mapping \+ ledger credit) | 22 / 38 / 80 | 8 |
| Reconciliation job (pull status fallback for missed webhooks) | 8 / 14 / 25 | 3 |
| Edge cases (partial/overpay/expiry/manual review flags) | 10 / 20 / 45 | 3 |
| Auto top-up (low-limit, amount, safeguards) | 20 / 40 / 80 | 5 |
| Tests \+ sandbox verification | 18 / 30 / 50 | 2 |
| **MVP-3 Lite Total** | **86 / 156 / 305** | **24** |

### **MVP-3 Full (if you want closer parity with the “another platform” list)**

| Feature Block (optional add-ons) | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Bank transfer (multi-currency instructions \+ reconciliation flow) | 50 / 90 / 170 | 8 |
| Custom images pipeline (upload/import/register \+ permissions) | 60 / 120 / 220 | 13 |
| Windows Server support (image/provisioning differences) | 40 / 90 / 180 | 8 |
| Stats dashboards (monthly usage, expense distribution, activity log) | 30 / 55 / 110 | 5 |
| Security extras (2FA \+ referral) | 35 / 70 / 140 | 8 |
| WS scale-out readiness (Redis adapter / multi-instance routing) | 16 / 35 / 80 | 5 |

**MVP-3 Full Total (typical)** \= **MVP-3 Lite typical (156h)** \+ selected add-ons above.

---

## **Phase Totals (Typical)**

| Phase | Typical Hours | Points |
| ----- | ----- | ----- |
| **MVP-1** | **409h** | **36** |
| **MVP-2** | **390h** | **45** |
| **MVP-3 Lite** | **156h** | **24** |

**Key variability drivers**

* OpenStack reconciler edge cases and GPU provisioning behavior.  
* Strictness of billing/usage rules (reserved funds, auto-stop on low balance).  
* Whether MVP-3 includes Windows/custom images/bank transfers.

