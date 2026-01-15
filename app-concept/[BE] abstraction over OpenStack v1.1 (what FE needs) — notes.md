## **BE abstraction over OpenStack v1.1 (what FE needs) — notes**

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

