# **Implementation Estimates Doc (BE \+ FE)**

## **Definitions**

* **Phases:** MVP-1, MVP-2, MVP-3 Lite (3-step MVP plan)  
* **Buffer:** **\+30% Buffer (QA, polishing, and unplanned work)**  
* **Point scale:** 1 tiny • 2 small • 3 medium • 5 large • 8 very large • 13 huge/risky  
* **FE stack:** React \+ TypeScript \+ TailwindCSS \+ shadcn/ui, simple WebSocket usage (no SSE)  
* **BE stack:** Node.js \+ TypeScript (NestJS recommended), PostgreSQL, WebSocket (simple subscribe), OpenStack integration via Keystone \+ Heat

---

## **Executive Summary (Totals)**

### **Backend (Customer BE)**

| Phase | Typical (h) | \+30% Buffer (h) | Points |
| ----- | ----- | ----- | ----- |
| **MVP-1** (Auth \+ Deployments \+ OpenStack \+ WS) | 409 | **532** | 36 |
| **MVP-2** (Balance \+ Card \+ Usage \+ API Keys \+ Support \+ Docs) | 390 | **507** | 45 |
| **MVP-3 Lite** (Crypto hosted invoice \+ webhooks \+ Auto top-up) | 156 | **203** | 24 |
| **Total** | **955** | **1,242** | — |

### **Frontend (Customer UI)**

| Phase | Typical (h) | \+30% Buffer (h) | Points |
| ----- | ----- | ----- | ----- |
| **FE MVP-1** (Auth \+ Deploy UI \+ WS status) | 244 | **318** | 41 |
| **FE MVP-2** (Billing UI \+ API Keys UI \+ Support UI) | 256 | **333** | 41 |
| **FE MVP-3 Lite** (Crypto top-up UI \+ Auto top-up UI) | 72 | **94** | 13 |
| **Total** | **572** | **745** | — |

### **Combined (BE \+ FE)**

| Scope | Typical (h) | \+30% Buffer (h) |
| ----- | ----- | ----- |
| **MVP-1** | 653 | **850** |
| **MVP-2** | 646 | **840** |
| **MVP-3 Lite** | 228 | **297** |
| **All phases** | **1,527** | **1,985** |

---

## **Backend Estimates (Detailed)**

## **MVP-1 — Core Cloud (BE)**

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

## **MVP-2 — Monetization \+ Platform Basics (BE)**

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

## **MVP-3 Lite — Crypto (Hosted Invoice) \+ Auto Top-up (BE)**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Crypto top-up via provider (create hosted invoice/checkout \+ redirect URL) | 8 / 14 / 25 | 3 |
| Crypto webhooks (signature verify \+ dedup \+ status mapping \+ ledger credit) | 22 / 38 / 80 | 8 |
| Reconciliation job (pull status fallback for missed webhooks) | 8 / 14 / 25 | 3 |
| Edge cases (partial/overpay/expiry/manual review flags) | 10 / 20 / 45 | 3 |
| Auto top-up (low-limit, amount, safeguards) | 20 / 40 / 80 | 5 |
| Tests \+ sandbox verification | 18 / 30 / 50 | 2 |
| **MVP-3 Lite Total** | **86 / 156 / 305** | **24** |

---

## **Frontend Estimates (Detailed)**

## **FE MVP-1 — Core Cloud UI**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| App scaffold (routing/layout, tailwind+shadcn setup, env, build) | 10 / 16 / 28 | 3 |
| Design system basics (theme, common components, forms, toasts) | 10 / 18 / 30 | 3 |
| API client \+ typing (wrapper \+ error handling) | 12 / 20 / 35 | 3 |
| Auth pages (sign up, sign in, email verify, reset password) | 18 / 30 / 50 | 5 |
| Session handling UX (logout, verify banners, guarded routes) | 8 / 14 / 24 | 3 |
| Deployments list page (table/cards, simple pagination) | 12 / 20 / 35 | 3 |
| Create deployment wizard (region → type → image → storage → ssh key) | 24 / 40 / 70 | 8 |
| Deployment details page (spec, status, IPs, SSH instructions, delete) | 14 / 26 / 45 | 5 |
| WebSocket integration v1 (connect, subscribe/unsubscribe, update UI state) | 10 / 18 / 35 | 3 |
| Empty/error/loading states | 8 / 14 / 24 | 2 |
| QA \+ fixes (responsive, edge cases) | 16 / 28 / 45 | 3 |
| **FE MVP-1 Total** | **142 / 244 / 421** | **41** |

---

## **FE MVP-2 — Billing \+ API Keys \+ Support UI**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Billing overview (balance card, quick actions) | 10 / 18 / 30 | 3 |
| Card top-up UI (amount → redirect/confirm states) | 14 / 26 / 45 | 5 |
| Ledger/top-ups/invoices lists | 18 / 32 / 55 | 5 |
| Invoice details (line items, download link) | 8 / 14 / 24 | 3 |
| Usage view v1 (monthly usage summary, simple charts/tables) | 14 / 26 / 45 | 5 |
| API keys UI (list/create/show-once secret/delete) | 14 / 24 / 45 | 5 |
| Scopes picker UI | 8 / 14 / 24 | 3 |
| IP whitelist UI (CIDR v4/v6 validation) | 10 / 18 / 30 | 3 |
| Support tickets UI (list/create/details/messages) | 18 / 32 / 55 | 5 |
| Docs links page (Postman/OpenAPI) | 4 / 6 / 12 | 1 |
| QA \+ fixes | 16 / 28 / 45 | 3 |
| **FE MVP-2 Total** | **134 / 256 / 410** | **41** |

---

## **FE MVP-3 Lite — Crypto \+ Auto Top-up UI**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Crypto top-up UI (select asset/chain → create intent → redirect \+ status) | 12 / 22 / 40 | 5 |
| Top-up status UX (pending/confirmed/failed) | 8 / 14 / 24 | 3 |
| Auto top-up settings UI (low-limit, amount, enable/disable) | 10 / 18 / 30 | 3 |
| QA \+ fixes | 10 / 18 / 30 | 2 |
| **FE MVP-3 Lite Total** | **40 / 72 / 124** | **13** |

