## **1\) \[BE\] Core principles (non-negotiable)**

* **One product VM \= one Heat Stack** (single “resource boundary”). Everything (port, SG, floating IP, server, volume later) lives inside the stack.  
* **DB stores intent \+ business state**, OpenStack stores infra state.  
  * DB \= who owns it, plan, billing, desired action, timestamps, your VM id  
  * OpenStack \= actual resources and their runtime status  
* Treat OpenStack as **eventually consistent**: never assume “request completed” \== “resource ready”.  
* All commands must be **idempotent** and **serial per VM**.

---

## **2\) Abstraction layers in BE**

### **A. Domain API (what your app exposes)**

Keep your public API small and stable:

* `CreateVm(spec)`  
* `DeleteVm(vmId)`  
* `GetVm(vmId)`  
* `ListVms(filters)`  
* `GetVmEvents(vmId)` (optional)

### **B. Control-plane services (business logic)**

* `VmService` (validates spec, enforces quotas, writes DB intent)  
* `ProvisioningService` (translates intent into stack operations)  
* `CatalogService` (images/flavors/network profiles)  
* `QuotaService` (app-level quotas; OpenStack quotas optional)  
* `AuditService` (append-only)

### **C. OpenStack adapter layer (thin, typed)**

* `KeystoneClient` (token)  
* `HeatClient` (create/delete/get/events/outputs)  
* Optional: `NovaClient` (flavors, server details)  
* Optional: `NeutronClient` (networks/subnets if you don’t hardcode profiles)  
* Optional: `GlanceClient` (images)

**Rule:** only adapter knows OpenStack specifics; domain never sees raw OpenStack errors.

---

## **3\) Data model to prevent inconsistency**

### **Minimal VM table (add these fields)**

* `id` (your UUID)  
* `tenant_id` (your tenant/workspace)  
* `stack_name`, `stack_id`  
* `desired_state`: `PRESENT | ABSENT`  
* `desired_spec` (json): image/flavor/key/network\_profile/gpu\_count etc.  
* `observed_state` (json snapshot): stack\_status, server\_id, ips  
* `status`: `REQUESTED | PROVISIONING | ACTIVE | DELETING | DELETED | ERROR`  
* `last_error_code`, `last_error_message`  
* `operation_id` (idempotency)  
* `locked_until` / `lock_version` (for safe concurrency)  
* timestamps

### **Separate `vm_operations` table (recommended)**

Tracks command execution safely:

* `operation_id` (idempotency key)  
* `vm_id`  
* `type`: CREATE/DELETE/UPDATE  
* `status`: PENDING/RUNNING/SUCCEEDED/FAILED  
* `attempts`, `next_retry_at`, `last_error`

This prevents “double create” and lets you retry safely.

---

## **4\) How to run safely: command → reconcile loop**

### **Pattern: “write intent, then reconcile”**

1. API receives request  
2. Validate \+ quota check  
3. **Write intent to DB** (transaction)  
4. Enqueue/trigger reconciliation  
5. Reconciler performs OpenStack actions

**Never** do “call OpenStack first, then write DB”.

---

## **5\) Reconciler design (the heart of correctness)**

Run a worker loop (or several) that continuously converges reality to intent.

### **Reconcile algorithm (per VM)**

* Acquire **distributed lock** on `vm_id` (Redis lock or DB row lock)  
* Read VM row  
* If `desired_state = PRESENT`:  
  * If no stack\_id: **create stack** (idempotent naming), store stack\_id  
  * Else: fetch stack status  
  * If stack COMPLETE: fetch outputs, set `ACTIVE`  
  * If stack FAILED: set `ERROR` \+ store reason (and optionally allow retry)  
* If `desired_state = ABSENT`:  
  * If stack exists: **delete stack**  
  * If stack gone: mark `DELETED` and clear outputs

### **Global drift sweeper (also required)**

Periodically:

* List stacks that match your naming/tag convention  
* Find:  
  * stacks without DB record → orphan cleanup policy  
  * DB records whose stacks are missing → mark error or recreate depending on desired\_state

This closes “gaps” caused by crashes/network outages.

---

## **6\) Idempotency rules (how to avoid duplicates/orphans)**

### **For CREATE**

* Require `Idempotency-Key` from client (or generate `operation_id`)  
* Deterministic stack naming:  
  * `stack_name = vm-{vm_id}`  
* Before create:  
  * If DB already has stack\_id → don’t create again  
  * If DB missing stack\_id but stack exists by name → **adopt** it (lookup stack id by name)

### **For DELETE**

* DELETE is always safe:  
  * “set desired\_state=ABSENT”  
  * reconciler deletes if exists; if already gone \-\> mark deleted

---

## **7\) Concurrency / race conditions to handle explicitly**

* User clicks “delete” while provisioning:  
  * Set desired\_state=ABSENT  
  * Reconciler sees stack in progress → issue delete once allowed; keep retrying  
* Two creates in parallel:  
  * idempotency \+ per-VM lock prevents double stack creation  
* Poller updating status while delete in flight:  
  * status must be derived from (desired\_state \+ observed stack status), not just last poll

---

## **8\) Retry policy (safe vs unsafe)**

### **Safe to retry automatically**

* GET/list operations  
* Fetch outputs/events  
* Token fetch (Keystone)  
* Stack status polling

### **Unsafe to retry unless idempotent**

* Create stack  
* Delete stack

Make create/delete retryable by:

* deterministic stack name  
* “check existence by name/id” before re-issuing

---

## **9\) Template safety (avoid “partial resource” mistakes)**

* Keep templates **versioned** in repo (e.g., `ubuntu-basic@v1`, `gpu-ubuntu@v1`)  
* Store `template_version` used per VM in DB  
* Standardize outputs across templates:  
  * `server_id`, `private_ip`, `public_ip`  
* Prefer creating *all infra inside Heat* (SG/port/fip). Don’t mix “some via Neutron directly” unless you must.

---

## **10\) Security model (safe OpenStack access)**

* Use **Application Credentials per tenant/project**  
* Encrypt secrets at rest (envelope encryption)  
* Least privilege:  
  * only permissions needed for Heat/Nova/Neutron/Glance actions you expose  
* Never expose OpenStack endpoints/tokens to browser  
* Add rate limiting per user/tenant (to protect OpenStack control plane)

---

## **11\) Observability \+ audit (to detect gaps early)**

* Store an append-only audit log:  
  * who requested create/delete  
  * operation\_id  
  * stack\_id  
* Persist key Heat events (last N) for UI/debug  
* Metrics:  
  * time to ACTIVE  
  * failure rate by template/version  
  * stuck stacks count (\> X minutes)  
* Alert on:  
  * reconcile backlog  
  * repeated failures  
  * orphan stacks detected

---

## **12\) Minimal “abstraction contract” you enforce**

* Your BE owns:  
  * VM identity, tenancy, quotas, lifecycle, billing, user access  
* Heat owns:  
  * infrastructure orchestration as a stack  
* Reconciler owns:  
  * convergence \+ drift repair

If you want, I can write the exact **state machine** (statuses \+ transitions) and the **DB constraints** (unique indexes, idempotency constraints, lock strategy) that make this rock-solid in practice.

