# **CloudStack Setup Estimates**

**Assumptions**

* Target: small IaaS similar to your OpenStack plan (VMs \+ public IPs \+ GPU later).  
* Hypervisor: **KVM** (CloudStack uses a **cloudstack-agent** on each KVM host that talks to the Management Server). ([Apache CloudStack Documentation](https://docs.cloudstack.apache.org/en/4.22.0.0/installguide/hypervisor/kvm.html?utm_source=chatgpt.com))  
* You will configure **zones/pods/clusters/hosts/storage/networks** in CloudStack UI/API. ([Apache CloudStack Documentation](https://docs.cloudstack.apache.org/en/latest/installguide/configuration.html?utm_source=chatgpt.com))  
* Storage model: **primary \+ secondary**; secondary is typically **NFS** in CloudStack docs. ([CloudStack](https://qa.cloudstack.cloud/builds/docs-build/pr/354/adminguide/storage.html?utm_source=chatgpt.com))  
* Management Server install supports single-node or multi-node patterns (MySQL same node vs separate). ([Apache CloudStack Documentation](https://docs.cloudstack.apache.org/en/latest/installguide/management-server/?utm_source=chatgpt.com))  
* Point scale: **1 tiny • 2 small • 3 medium • 5 large • 8 very large • 13 huge/risky**

---

## **Phase A — PoC / Lab (1 zone, 1 management server, 1–2 KVM hosts, NFS primary+secondary)**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Network design (mgmt/guest/public IP plan, VLAN vs flat) | 6 / 10 / 18 | 3 |
| Management Server install (single node) \+ DB bootstrap | 6 / 10 / 18 | 3 |
| NFS setup (primary \+ secondary) | 6 / 10 / 20 | 3 |
| KVM host prep \+ cloudstack-agent install \+ libvirt basics | 10 / 18 / 30 | 5 |
| CloudStack initial config (zone/pod/cluster/hosts/storage/network offerings) | 10 / 18 / 35 | 5 |
| System templates \+ system VMs validation (router/SSVM) | 6 / 10 / 18 | 3 |
| Smoke tests (launch VM, public IP/NAT, SSH, delete VM) | 6 / 12 / 24 | 3 |
| **Phase A Total** | **50 / 88 / 163** | **25** |

---

## **Phase B — Small Production (single region, 3–10 KVM hosts, basic reliability & ops)**

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Production network design (VLANs, redundancy, firewall rules, IP pools) | 12 / 24 / 50 | 8 |
| Management Server production install (separate DB host, backups, TLS) | 12 / 24 / 45 | 5 |
| Storage production (NFS hardening, performance tuning, backup strategy) | 16 / 32 / 70 | 8 |
| KVM fleet prep (homogeneous cluster rules, automation via Ansible) | 20 / 40 / 80 | 8 |
| CloudStack config hardening (offerings, templates, quotas, accounts, projects) | 16 / 32 / 60 | 5 |
| Monitoring/logging (basic dashboards, alerts, audit trail) | 12 / 24 / 50 | 5 |
| Failure drills (host down, storage hiccup, recover procedures) | 10 / 20 / 40 | 3 |
| **Phase B Total** | **98 / 196 / 395** | **42** |

---

## **Phase C — GPU Production Enablement (A100/H100 passthrough offerings \+ GPU templates)**

CloudStack supports **GPU passthrough** (physical GPU assigned to a VM). ([Apache CloudStack Documentation](https://docs.cloudstack.apache.org/en/4.15.0.0/adminguide/virtual_machines.html?utm_source=chatgpt.com))  
(Your KVM hosts still must be configured correctly for IOMMU/passthrough; CloudStack’s role is discovery \+ assign/unassign at the platform level.)

| Module | Hours (Low / Typical / High) | Points |
| ----- | ----- | ----- |
| Host-level GPU passthrough prep (BIOS/IOMMU, driver/tooling baseline, validation) | 16 / 32 / 70 | 8 |
| CloudStack GPU config (offerings, GPU groups/assignments, quotas) | 12 / 24 / 50 | 5 |
| GPU templates/images (Ubuntu \+ CUDA stack; “OblivusAI-like” image pipeline) | 16 / 40 / 90 | 8 |
| Scheduling/placement policy (dedicated hosts/clusters for GPU, capacity checks) | 10 / 20 / 45 | 5 |
| End-to-end tests (create GPU VM, validate device, teardown, repeatability) | 10 / 20 / 40 | 3 |
| **Phase C Total** | **64 / 136 / 295** | **29** |

---

## **Combined totals (Typical)**

| Scope | Typical Hours | Points |
| ----- | ----- | ----- |
| **Phase A (PoC/Lab)** | **88h** | **25** |
| **Phase B (Small Production)** | **196h** | **42** |
| **Phase C (GPU Enablement)** | **136h** | **29** |
| **All (A+B+C)** | **420h** | **96** |

---

## **Notes on what can change estimates fast**

* **Networking complexity** (VLAN/L3, public IP routing, provider network constraints).  
* **Storage choice** (simple NFS vs more robust storage stack).  
* **How “productized” GPU is** (just passthrough vs curated ML images and strict placement rules).  
* **HA for management/database** (CloudStack docs cover HA patterns; doing it properly adds extra work). ([Apache CloudStack Documentation](https://docs.cloudstack.apache.org/en/latest/installguide/management-server/?utm_source=chatgpt.com))

