## **BE requirements for integrating with OpenStack (notes)**

### **1\) Credentials \+ auth (Keystone)**

* Store **per-tenant** OpenStack credentials (recommended: **Application Credential** id/secret)  
* Module: `keystoneAuth`  
  * `POST /v3/auth/tokens` (get token)  
  * Cache token \+ expiry in memory/redis  
  * Auto-refresh on expiry / retry once on `401`  
* Decide endpoint strategy:  
  * **Use Keystone service catalog** (discover Heat/Nova/Neutron/Glance URLs dynamically), or  
  * Hardcode endpoints per region (simpler for MVP)

### **2\) Typed OpenStack API clients (service adapters)**

Create a small client wrapper per service (HTTP \+ TS types \+ error mapping):

* `heatClient` (stacks lifecycle)  
  * create stack, get stack, delete stack  
  * list events, get outputs  
* `novaClient` (mostly catalog)  
  * list flavors, keypairs (optional), server details (optional)  
* `neutronClient` (catalog \+ validation)  
  * list networks/subnets, SGs (optional), floating IPs (optional)  
* `glanceClient` (image catalog)  
  * list images, image details

Implementation notes:

* Use one HTTP layer (`fetch`/`axios`) with:  
  * `X-Auth-Token` injection  
  * request timeout \+ retry policy (safe retries only)  
  * correlation/request id header propagation

### **3\) Domain model \+ DB entities (minimum)**

* `Tenant/Project` (maps your org/workspace → OpenStack project)  
* `OpenStackCredential` (encrypted app\_cred\_id/secret or equivalent)  
* `VM`  
  * `stack_id`, `stack_name`  
  * `status` (REQUESTED/PROVISIONING/ACTIVE/ERROR/DELETING/DELETED)  
  * outputs snapshot: `server_id`, `private_ip`, `public_ip`  
  * spec snapshot: `image`, `flavor`, `key`, `network_profile`  
* Catalog tables (or cached in memory for MVP):  
  * `Image`, `Flavor`, `NetworkProfile`, `SSHKey`

### **4\) Heat template management**

* Template library in repo (`templates/ubuntu-basic.yaml`, `templates/gpu-ubuntu.yaml`, etc.)  
* `templateRenderer`:  
  * validate inputs (zod)  
  * render params into HOT  
  * keep outputs consistent (`public_ip`, `private_ip`, `server_id`) so BE can parse uniformly

### **5\) VM lifecycle “workflow” inside BE (without Temporal)**

You still need a durable lifecycle controller:

* API handler creates `VM` row → calls Heat `create stack` → stores `stack_id`  
* Background worker / poller:  
  * polls Heat stack status \+ events  
  * updates VM status and progress log  
  * on COMPLETE: fetch outputs and persist them  
* Optional: “reconciler” job (every N minutes)  
  * verify stacks referenced by DB still exist  
  * mark drifted records, attempt repair, or surface to admin

### **6\) Quotas \+ access control**

* Define RBAC in your app:  
  * user can only act in their tenant/project  
* Enforce quotas (app-level MVP):  
  * max VMs, max vCPU/RAM/GPU count, max floating IPs  
* (Optional) Mirror OpenStack quotas if you enable them, but you can start app-only.

### **7\) Idempotency \+ safety**

* Idempotency key on create VM (`client_request_id`)  
* Resource tagging strategy:  
  * include your `vm_id` in stack name/tags/metadata  
* Safe retry rules:  
  * retry GET/list freely  
  * retry create/delete only if you can prove idempotency (existing stack detected)

### **8\) Observability \+ audit**

* Structured logs (include `tenant_id`, `vm_id`, `stack_id`, `request_id`)  
* Store key actions as audit events (create/delete, credential changes)  
* Metrics:  
  * stack create latency, failure rate, OpenStack API error rate  
* Admin debug tools:  
  * fetch raw Heat events for a VM  
  * link to OpenStack IDs (server\_id, port\_id) when available

### **9\) Secrets \+ security hardening**

* Encrypt app credentials at rest (KMS/Cloud HSM later; for MVP use envelope encryption \+ env master key)  
* Rotate app credentials (manual admin flow first)  
* Least-privilege OpenStack roles for app creds (only what Heat/Nova/Neutron/Glance need)

### **10\) Local/dev integration setup**

* A dev OpenStack environment (choose one):  
  * MicroStack / DevStack / single-node lab  
* Integration tests:  
  * create stack → wait → verify outputs → delete stack  
* “Debug mode” using `openstack` CLI only for troubleshooting (not as a dependency)