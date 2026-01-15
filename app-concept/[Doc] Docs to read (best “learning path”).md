### **Docs to read (best “learning path”)**

**Big picture**

* OpenStack API Quick Start (how APIs/tokens fit together). ([OpenStack Docs](https://docs.openstack.org/api-quick-start/?utm_source=chatgpt.com))

**Identity & tenancy (Keystone)**

* Projects/users/roles concepts (project \= tenant). ([OpenStack Docs](https://docs.openstack.org/keystone/latest/admin/cli-manage-projects-users-and-roles.html?utm_source=chatgpt.com))  
* Application credentials (best for your backend). ([OpenStack Docs](https://docs.openstack.org/keystone/latest/user/application_credentials.html?utm_source=chatgpt.com))  
* Identity API v3 reference (tokens/auth). ([OpenStack Docs](https://docs.openstack.org/api-ref/identity/v3/?utm_source=chatgpt.com))

**Your “VM \= Stack” layer (Heat)**

* Heat Template Guide (how to write HOT). ([OpenStack Docs](https://docs.openstack.org/heat/latest/template_guide/?utm_source=chatgpt.com))  
* Basic resources: using `OS::Nova::Server` etc. ([OpenStack Docs](https://docs.openstack.org/heat/latest/template_guide/basic_resources.html?utm_source=chatgpt.com))  
* Orchestration API v1 (events/stack endpoints you’ll call). ([OpenStack Docs](https://docs.openstack.org/api-ref/orchestration/v1/?utm_source=chatgpt.com))

**Underlying resources (Nova/Neutron/Glance)**

* Compute API reference \+ guide (servers, flavors, keypairs, etc.). ([OpenStack Docs](https://docs.openstack.org/api-ref/compute/?utm_source=chatgpt.com))  
* Neutron intro \+ Network API v2 (networks/subnets/ports/SG/floating IP). ([OpenStack Docs](https://docs.openstack.org/neutron/2025.1/admin/intro-os-networking.html?utm_source=chatgpt.com))  
* Glance Image API v2 (images). ([OpenStack Docs](https://docs.openstack.org/api-ref/image/v2/?utm_source=chatgpt.com))