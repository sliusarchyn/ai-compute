## **BE–FE concept (MVP) with simple WebSockets**

### **Goal**

* FE loads state via HTTP  
* FE subscribes via WS to a specific deployment (VM) and receives live status/progress updates  
* No polling loop; on WS reconnect FE does one HTTP refetch

---

## **Tech stack**

### **Frontend**

* React.js \+ TypeScript  
* TailwindCSS \+ shadcn/ui  
* TanStack Query (recommended for caching \+ invalidation)  
* WS client: native `WebSocket` (keep it simple)

### **Backend**

* Node.js \+ TypeScript  
* NestJS (or Express/Fastify; Nest is cleaner for modules \+ WS gateway)  
* PostgreSQL (Prisma or TypeORM)  
* Redis (optional in MVP; add later for multi-instance WS scaling \+ locks)  
* Background jobs: BullMQ (optional; or a simple worker loop in MVP)

### **OpenStack integration (BE-only)**

* Keystone auth (application credentials)  
* Heat stack lifecycle (create/delete, events, outputs)  
* Optional: Nova/Neutron/Glance for catalog

---

## **Core entities (what FE renders)**

### **Deployment (VM product object)**

* `id`  
* `name`  
* `regionId`  
* `spec` `{ instanceTypeId, imageId, storage, sshKeyId }`  
* `status` `REQUESTED | PROVISIONING | ACTIVE | ERROR | DELETING | DELETED`  
* `runtime` `{ publicIp?, privateIp?, sshUser? }`  
* `lastError? { code, message }`  
* `createdAt`, `updatedAt`

### **DeploymentEvent (optional but very useful)**

* `id`, `deploymentId`, `ts`  
* `level` `INFO|WARN|ERROR`  
* `message`

---

## **HTTP API (minimal for FE)**

### **Deployments**

* `POST /deployments`  
  **Returns:** `{ deploymentId }` (status starts as `REQUESTED/PROVISIONING`)  
  * accepts `Idempotency-Key` header  
* `GET /deployments?page=&pageSize=`  
* `GET /deployments/:id`  
* `DELETE /deployments/:id` (async delete)

### **Optional**

* `GET /deployments/:id/events?page=&pageSize=`

### **Auth (minimal set)**

* `POST /auth/register`, `POST /auth/login`, `POST /auth/logout`  
* `POST /auth/verify-email`, `POST /auth/resend-verification`  
* `GET /me`

**Auth mode recommended for browsers:** httpOnly session cookies.

---

## **WebSocket API (simple subscription model)**

### **Connection**

* `wss://api.yourdomain.com/ws`  
* Authenticate with cookie (preferred) or bearer token.  
* On connect, server identifies `userId/customerId`.

### **Client messages**

**Subscribe to one deployment**

{ "type": "subscribe", "deploymentId": "dep\_123" }

**Unsubscribe**

{ "type": "unsubscribe", "deploymentId": "dep\_123" }

### **Server messages**

**Ack**

{ "type": "subscribed", "deploymentId": "dep\_123" }

**Deployment update (send on any meaningful change)**

{  
  "type": "deployment.updated",  
  "deploymentId": "dep\_123",  
  "data": {  
    "status": "PROVISIONING",  
    "runtime": { "publicIp": null, "privateIp": "10.0.0.5", "sshUser": "ubuntu" },  
    "lastError": null,  
    "updatedAt": "2026-01-06T10:10:00Z"  
  }  
}

**Progress log line (optional)**

{  
  "type": "deployment.event",  
  "deploymentId": "dep\_123",  
  "data": { "ts": "2026-01-06T10:10:05Z", "level": "INFO", "message": "Creating server…" }  
}

**Error**

{ "type": "error", "code": "FORBIDDEN", "message": "No access to deployment" }

---

## **FE behavior (how pages work)**

### **Deployment list page**

1. `GET /deployments`  
2. Render list (no WS needed)  
3. Optionally, when user opens a row → navigate to details

### **Deployment details page (the main WS use case)**

1. `GET /deployments/:id` (initial state)  
2. Open WS (if not already)  
3. Send `subscribe`  
4. On `deployment.updated`:  
   * update UI state (or update TanStack Query cache for `['deployment', id]`)  
5. On `deployment.event`:  
   * append to local log list (and/or refetch events if you store them)  
6. On WS reconnect:  
   * call `GET /deployments/:id` once  
   * re-send `subscribe`

**No polling loop.**

---

## **BE internal flow (what triggers WS updates)**

### **Create deployment (async)**

1. `POST /deployments` creates DB row `status=REQUESTED`  
2. Enqueue a job (or start worker) “provision deployment”  
3. Worker calls Heat “create stack”  
4. Worker periodically checks Heat stack status/events  
5. When status changes or IP appears:  
   * update DB (`deployments` \+ insert `deployment_events`)  
   * emit WS message to subscribers of `deploymentId`

### **Delete deployment**

1. `DELETE /deployments/:id` sets `status=DELETING` and desired\_state=ABSENT  
2. Worker deletes Heat stack  
3. Worker updates DB to `DELETED`  
4. Emit `deployment.updated` (and optionally final event log)

---

## **Minimal “correctness” rules (without complex WS replay)**

* WS is **best-effort UX**, not source of truth.  
* On reconnect FE does **one HTTP refetch**.  
* BE emits updates **only after DB commit** (so emitted state always matches what HTTP would return).  
* Subscription authorization: server checks `deployment.customerId == socket.customerId`.

That gives a robust MVP without building an outbox/replay system yet.

---

## **Suggested module layout (BE)**

* `auth/`  
* `deployments/` (controller \+ service)  
* `openstack/` (keystone \+ heat clients)  
* `workers/` (provision/delete/reconcile loop)  
* `ws/` (gateway \+ subscription registry)

---

