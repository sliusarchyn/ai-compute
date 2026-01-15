## **1) Quick "suitability/availability" check**

* What services do you provide: **dedicated bare metal**, **colocation**, **managed validators**, **GPU servers**, **custom builds**, **Kubernetes/VM hosting**?  
* What are your **data center locations** (country/city), can I choose a specific one?  
* What is the current **lead time** for server provisioning according to my specification (in stock vs. custom order)?  
* Minimum contract term? Is there **monthly payment** (month-to-month)?

## **2) Hardware (CPU/RAM/Storage/Network)**

* Can you provide **dedicated bare-metal** that matches my specification?  
* What CPU families are available (EPYC/Intel), can I choose a **specific model**?  
* What RAM is available (DDR4/DDR5 ECC), maximum RAM per node, upgrade possibilities?  
* Disks/storage:  
  * NVMe/SATA options, capacities, brands, RAID (HW/SW), hot-swap?  
  * Separate disks for OS and data?  
* Network:  
  * Port options (1/10/25/40/100GbE)  
  * How many public IPs are included, is IPv6 available?  
  * What is the traffic limit, burst policy, overage price?  
  * Is DDoS protection included? What levels/limits?

## **3) GPU**

* What GPU models are available (A100/H100/L40S/4090 etc.), VRAM variants, PCIe vs. SXM?  
* Can you provide **multi-GPU nodes** (e.g., 4–8 GPUs)? What chassis/PCIe lanes layout?  
* Power/cooling limits for GPUs in your infrastructure (per unit/per server/per rack)?  
* Support for **NVIDIA drivers/CUDA**, persistence mode, MIG (for A100/H100)?

## **4) Pricing Model (Hardware + Monthly Payments)**

* What is the model: **do you sell the hardware** (capex) or **lease it** (monthly payment)?  
* If leasing:  
  * Monthly price breakdown: server, network, IP, management, backup, DDoS  
  * Is there a setup fee?  
* If purchasing:  
  * Hardware price, warranty, support plans  
  * Is there leasing/financing?  
* Do prices vary by location?

## **5) Electricity Price (Very Important)**

* How do you calculate electricity:  
  * Included in the package (dedicated limit/quota) or **monthly per kWh** (metered)?  
* What is the **$/kWh** rate in each location?  
* Do you bill for **actual consumption** or for **reserved power**?  
* Is **PUE (Power Usage Effectiveness)** included in the tariff? What is the typical PUE?  
* Are there peak/off-peak rates, minimum power payment, penalties/overages?

## **6) Management / Operations (unmanaged vs. managed)**

* What management levels are available:  
  * Unmanaged (I do everything) vs. fully managed (you patch/monitor/reboot)  
* What is included in "managed":  
  * OS installation, hardening, firewall, monitoring, alerts  
  * Automatic reboot / kernel updates policy  
  * Disk replacement and RAID rebuild process  
* Remote access:  
  * IPMI/iDRAC/iLO included?  
  * Remote KVM / rescue mode?  
* Do you support **Terraform/Ansible** workflows or provide an API?

## **7) Support, SLA, and Incidents**

* Support schedule: 24/7? Channels (tickets/Slack/Telegram/phone)?  
* SLA:  
  * Network uptime SLA (%)  
  * Hardware replacement SLA (e.g., "within 4 hours")  
* Typical response times for:  
  * Hardware failure  
  * Network incident  
  * "Remote hands" tasks  
* Is there an escalation path / on-call contact?

## **8) Security and Compliance**

* Physical security: access control, CCTV, staff access policies?  
* Network security: DDoS, filtering, BGP, WAF options?  
* Do you support:  
  * Disk encryption (LUKS), HSM, TPM  
  * Key management practices (especially for validators)  
* Is there compliance (SOC2/ISO) or at least data center certifications?

## **9) Backups, Snapshots, DR**

* Are there backups/snapshots? What is the price?  
* Where are backups stored (same DC vs. different region)?  
* Expected recovery time and limitations.

## **10) Scaling**

* If I start with 1–2 nodes, can I quickly scale to N?  
* Can you ensure **identical hardware configurations** for subsequent servers?  
* Is **capacity reservation** possible in advance?

## **11) Contract, Billing, Exit**

* Contract terms, cancellation notice, early termination fee?  
* Currency, payment methods (card/wire/crypto), invoices/VAT?  
* Upon termination:  
  * Data wipe policy and confirmation  
  * Do you assist with migration?

## **12) Validation of "My Specification"**

* Here is my target spec: **[CPU] [RAM] [Storage] [Network] [GPU if needed] [location preferences]**  
  * Can you match it 1:1? If not — what are the closest options and price?  
  * What alternative configuration do you recommend for better price/performance?