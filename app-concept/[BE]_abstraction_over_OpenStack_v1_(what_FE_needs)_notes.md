## **BE abstraction over OpenStack v1 (what FE needs) — notes**

### **Main idea**

* **FE talks only to your BE**  
* **BE owns “product entities”** (customer, balance, invoices, deployments, API keys, tickets)  
* **OpenStack is hidden behind BE adapters**  
* **1 Deployment \= 1 Heat Stack** (single boundary for VM \+ port \+ SG \+ floating IP \+ volumes)

---

## **Core BE domains (entities)**

### **Identity / Account**

* `User`  
* `Customer` (Customer ID)  
* `Session` (active sessions management)  
* `EmailVerificationToken`  
* `TwoFactor` (TOTP secret, recovery codes)  
* `BillingProfile` (company/individual, address, VAT, etc.)  
* `Referral` (code \+ rewards)

### **Deploy / Compute**

* `Region` (Montreal / Oslo / Texas)  
* `InstanceType` (CPU-only, H100/A100 NVLINK/PCIe variants; maps to OpenStack flavors/traits)  
* `Image` (Ubuntu 22.04/20.04, OblivusAI image, Windows 2022, Custom)  
* `StoragePlan` (local vs network; size limits)  
* `Deployment` (your “VM”)  
  * `desired_state` PRESENT/ABSENT  
  * `status` REQUESTED/PROVISIONING/ACTIVE/ERROR/DELETING/DELETED  
  * OpenStack refs: `stack_id`, `stack_name`, `server_id`, IPs  
* `DeploymentEvent` (progress log from Heat events)

### **API access**

* `ApiKey` (token \+ scopes)  
* `PermissionScope` (custom permissions)  
* `IpWhitelistRule` (IPv4/IPv6 CIDR, per key or per customer)

### **Billing**

* `LedgerAccount` (internal balance)  
* `LedgerEntry` (append-only credits/debits)  
* `TopUpIntent` (card / crypto / bank transfer)  
* `CryptoDeposit` (chain, asset, address, status)  
* `AutoTopUpRule` (low-limit, amount)  
* `Invoice` (history)  
* `UsageRecord` (hourly/daily usage per deployment)  
* `Transfer` (internal transfers if needed later)

### **Support**

* `Ticket`  
* `TicketMessage`

---

## **FE → BE required endpoint surface (minimum)**

### **1\) Auth (email/password)**

* `POST /auth/register`  
* `POST /auth/login`  
* `POST /auth/logout`  
* `POST /auth/verify-email`  
* `POST /auth/resend-verification`  
* `POST /auth/forgot-password`  
* `POST /auth/reset-password`  
* `GET /me`

**Sessions**

* `GET /sessions`  
* `DELETE /sessions/:id` (revoke a session)

**2FA**

* `POST /security/2fa/setup` (returns QR data / secret)  
* `POST /security/2fa/confirm`  
* `POST /security/2fa/disable`  
* `POST /security/2fa/recovery-codes/regen`

---

### **2\) Deploy (catalog \+ availability \+ create VM)**

**Catalog**

* `GET /regions` (Montreal/Oslo/Texas)  
* `GET /catalog/instance-types` (CPU \+ GPU variants)  
* `GET /catalog/images` (Ubuntu, OblivusAI, Windows, Custom)  
* `GET /catalog/storage-plans` (local/network, min/max GB)  
* `GET /catalog/gpu-availability` (per region \+ GPU type)

**Availability check (recommended)**

* `POST /deployments/quote`  
  * input: region, instanceType, storage, image  
  * output: “available now?”, estimated price, limits, warnings

**Create deployment**

* `POST /deployments` (async)  
  * header: `Idempotency-Key`  
  * body: region, instanceType, imageId, storage, sshKeyId, optional name  
  * returns: `deploymentId`, `operationId`, `status=PROVISIONING`

**List / detail**

* `GET /deployments`  
* `GET /deployments/:id`  
* `GET /deployments/:id/events` (progress timeline)

**Lifecycle (MVP: only delete; Phase 2: more)**

* `DELETE /deployments/:id` (async delete)  
* Phase 2 actions:  
  * `POST /deployments/:id/actions/start|stop|reboot|resize|rebuild`

---

### **3\) OS images (incl. custom image)**

**MVP**

* built-in images only via `GET /catalog/images`

**Custom image (Phase 2\)**

* `POST /images` (create upload intent)  
* `PUT /images/:id/upload` (signed URL or chunk upload)  
* `POST /images/:id/import` (register to OpenStack/Glance)  
* `GET /images`  
* `DELETE /images/:id`

---

### **4\) API (keys \+ permissions \+ IP whitelist \+ docs)**

**API Keys**

* `GET /api-keys`  
* `POST /api-keys` (name, scopes)  
* `DELETE /api-keys/:id`

**Custom permissions**

* `GET /api-scopes`  
* `POST /api-keys/:id/scopes` (replace scopes)

**IP whitelist**

* `GET /api-keys/:id/ip-whitelist`  
* `POST /api-keys/:id/ip-whitelist` (CIDR v4/v6)  
* `DELETE /api-keys/:id/ip-whitelist/:ruleId`

**Docs**

* `GET /docs/openapi.json`  
* `GET /docs/postman.collection.json` (generated from OpenAPI or maintained)

---

### **5\) Billing (internal balance \+ top-ups \+ invoices)**

**Balance / ledger**

* `GET /billing/balance`  
* `GET /billing/ledger?from=&to=&type=`

**Top-up (card/bank/crypto)**

* `POST /billing/topups/card` (create payment intent)  
* `POST /billing/topups/crypto` (asset: USDT/USDC, chain: ETH/BSC/Polygon → returns deposit address)  
* `POST /billing/topups/bank-transfer` (currency USD/EUR/GBP/CAD/CHF/AED → returns instructions \+ reference)  
* `GET /billing/topups` (history)  
* `GET /billing/topups/:id`

**Auto top-up**

* `GET /billing/auto-topup`  
* `PUT /billing/auto-topup` (enabled, lowLimit, topUpAmount, method)

**Invoices**

* `GET /billing/invoices`  
* `GET /billing/invoices/:id` (pdf url or download)

**Webhooks (server-to-server)**

* `POST /billing/webhooks/card-provider`  
* `POST /billing/webhooks/crypto-provider`  
* `POST /billing/webhooks/bank-provider` (if used)

---

### **6\) Account**

* `GET /account/profile`  
* `PUT /account/profile`  
* `GET /account/billing-profile`  
* `PUT /account/billing-profile`  
* `POST /account/change-email`  
* `POST /account/change-password`  
* `GET /account/settings` (2FA status, etc.)

---

### **7\) Referral program**

* `GET /referral` (my code, my stats)  
* `POST /referral/redeem` (apply someone’s code)  
* `GET /referral/rewards` (history)

---

### **8\) Support (tickets)**

* `POST /support/tickets`  
* `GET /support/tickets`  
* `GET /support/tickets/:id`  
* `POST /support/tickets/:id/messages`  
* `POST /support/tickets/:id/close`

---

### **9\) Stats**

* `GET /stats/usage/monthly?month=YYYY-MM`  
* `GET /stats/expenses/distribution?from=&to=`  
* `GET /stats/expenses/monthly?from=&to=`  
* `GET /stats/transfers?from=&to=`  
* `GET /activity/recent?limit=50` (audit/activity log for UI)

---

## **“No gaps” rules BE must implement (so FE stays simple)**

* **Async operations**: create/delete always return `operationId`; FE polls `/deployments/:id` \+ `/events`  
* **Idempotency** on create: `Idempotency-Key` \+ deterministic `stack_name = dep-{deploymentId}`  
* **Per-deployment lock** during reconcile (Redis/DB) to prevent double actions  
* **Reconciler worker**: polls Heat stack status/events → updates DB → fetches outputs  
* **Drift sweeper**: detects orphan stacks and missing stacks and resolves per policy  
* **Billing gating**: deny `/deployments` if balance/plan/limits not satisfied; record denies in activity log

---

## **Tech stack decision (recommended)**

* **BE**: Node.js \+ TypeScript \+ **NestJS**  
* **DB**: PostgreSQL  
* **Queue/locks**: Redis \+ BullMQ (reconcile, sweeps, usage aggregation)  
* **Auth**: self-hosted (JWT \+ refresh \+ httpOnly cookies) or Firebase/Auth0 (but still keep your own User/Customer DB)  
* **OpenStack adapter**: typed clients for Keystone/Heat/Nova/Neutron/Glance  
* **Observability**: OpenTelemetry \+ structured logs

---

If you want, paste your intended **MVP scope** (which regions, which GPU types, whether Windows/custom images/bank transfer are in MVP), and I’ll cut this into an **MVP-only endpoint set** \+ exact request/response shapes for FE.

1. 