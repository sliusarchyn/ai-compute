# **FE Implementation Estimates (3 MVP Phases)**

**Assumptions**

* React.js \+ TypeScript \+ TailwindCSS \+ shadcn/ui  
* Data fetching: TanStack Query (recommended)  
* WebSocket: native `WebSocket` (simple subscribe to a deployment; no SSE; no polling loops)  
* Simple pagination UI (`page`, `pageSize`, `total`) but expected low volumes per user  
* Customer UI has **no roles** (admin is separate app)

Point scale: **1 tiny • 2 small • 3 medium • 5 large • 8 very large • 13 huge/risky**

---

## **MVP-1 — Core Cloud UI (Auth \+ Deploy \+ WS status)**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| App scaffold (Vite/Next? routing, layout, tailwind+shadcn setup, env, build) | 10 / 16 / 28 | 3 |
| Design system basics (theme, common components, form patterns, toasts) | 10 / 18 / 30 | 3 |
| API client \+ typing (OpenAPI client or manual fetch wrapper, error handling) | 12 / 20 / 35 | 3 |
| Auth pages (sign up, sign in, email verify screen, reset password) | 18 / 30 / 50 | 5 |
| Session handling UX (logout, “verify your email” banners, guarded routes) | 8 / 14 / 24 | 3 |
| Deployments list page (table/cards, pagination, filters minimal) | 12 / 20 / 35 | 3 |
| Create deployment wizard (region → instance type → image → storage → ssh key) | 24 / 40 / 70 | 8 |
| Deployment details page (spec, status, IPs, SSH instructions, delete) | 14 / 26 / 45 | 5 |
| WebSocket integration v1 (connect, subscribe/unsubscribe, update UI state) | 10 / 18 / 35 | 3 |
| Empty/error/loading states \+ basic i18n-ready structure (optional) | 8 / 14 / 24 | 2 |
| Basic QA \+ fixes (cross-browser, responsive, edge cases) | 16 / 28 / 45 | 3 |
| **MVP-1 Total** | **142 / 244 / 421** | **41** |

---

## **MVP-2 — Billing \+ API Keys \+ Support UI**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Billing overview (balance card, quick actions) | 10 / 18 / 30 | 3 |
| Top-up flows (card) UI (amount form → redirect/confirm states) | 14 / 26 / 45 | 5 |
| Ledger/history screens (ledger list, top-ups list, invoices list) | 18 / 32 / 55 | 5 |
| Invoice details page (line items, download link) | 8 / 14 / 24 | 3 |
| Usage view v1 (monthly usage summary, simple charts/tables) | 14 / 26 / 45 | 5 |
| API keys UI (list/create/show-once secret/delete) | 14 / 24 / 45 | 5 |
| Scopes picker UI (multi-select, descriptions) | 8 / 14 / 24 | 3 |
| IP whitelist UI (add/remove CIDR v4/v6, validation) | 10 / 18 / 30 | 3 |
| Support tickets UI (list, create, details, messages) | 18 / 32 / 55 | 5 |
| Docs links/download (Postman/OpenAPI link page) | 4 / 6 / 12 | 1 |
| QA \+ fixes | 16 / 28 / 45 | 3 |
| **MVP-2 Total** | **134 / 256 / 410** | **41** |

---

## **MVP-3 — Expansion UI (Crypto top-up \+ Auto top-up \+ Optional extras)**

### **MVP-3 Lite (crypto hosted invoice \+ auto top-up)**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Crypto top-up UI (select asset/chain → create intent → redirect \+ status) | 12 / 22 / 40 | 5 |
| Top-up status UX (pending/confirmed/failed, refresh states) | 8 / 14 / 24 | 3 |
| Auto top-up settings UI (low-limit, amount, enable/disable, safeguards copy) | 10 / 18 / 30 | 3 |
| QA \+ fixes | 10 / 18 / 30 | 2 |
| **MVP-3 Lite Total** | **40 / 72 / 124** | **13** |

### **MVP-3 Full (optional add-ons)**

| Feature Block | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Bank transfer UI (currency selection, instructions, reference codes) | 14 / 26 / 45 | 5 |
| Custom images UI (upload flow, progress, list/manage images) | 20 / 40 / 70 | 8 |
| Windows options in deploy wizard \+ hints/warnings | 6 / 12 / 24 | 2 |
| Stats dashboards (expense distribution, activity log views, charts) | 18 / 36 / 70 | 8 |
| Security extras UI (2FA setup/confirm, recovery codes; referral pages) | 18 / 40 / 80 | 8 |

---

## **Phase Totals (Typical)**

| Phase | Typical Hours | Points |
| ----- | ----- | ----- |
| **FE MVP-1** | **244h** | **41** |
| **FE MVP-2** | **256h** | **41** |
| **FE MVP-3 Lite** | **72h** | **13** |

**Main variability drivers**

* How polished the “create deployment wizard” must be (validation, stepper UX, pricing quote, availability UX).  
* Charting complexity for usage/stats (tables vs rich charts).  
* Upload UI complexity for custom images (chunking/progress/retry).

