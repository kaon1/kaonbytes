+++
author = "Kaon Thana"
title = "Golden Configuration Deployment with Ansible and Netbox"
date = "2022-11-01"
description = "Enforce intended network state using ansible and a rendered golden configuration template"
categories = [
    "ansible",
    "automation",
    "netbox",
    "juniper",
    "netdevops"
]

aliases = ["golden-config-engine"]
image = "golden-config-front.png"
+++

**The first step in creating a self-healing network** is to implement a golden configuration standard. Ensuring that your network devices are in compliance with intended parameters improves operational overhead, limits security risks and potential outages caused by human error. 

However, this is a **complex** problem to solve because a universal one-size-fits-all configuration is not common. Many factors cause slight changes in configurations such as: 
  - device hardware
  - firmware
  - region specific parameters
  - non-idempotent secret keys
  
... just to name a few. So how can we do it?

## Purpose
In this blog post, I will share a technical architecture that I have deployed in **production** on hundreds of network nodes accross dozens of sites and global regions. 

### The Steps
1. Populate **Source of Truth (Netbox)** with desired configuration context to be loaded as dynamic inventory hostvars
    * Use weighted config context to populate region or site wide specific configuration
    * Use custom fields to populate device specific configuration (if needed)
2. Populate **Secret Passwords** in Ansible Automation Platform Credentials library, Ansible Vault, HashiCorp Vault or some other secure method to be called later
3. Build custom python module to **create idempotent secret keys** per host. 
    * Securely encrypted keys which can be re-produced on each host (i.e not change on every run)
4. Render intended confgiuratio with Ansible **Jinja2 Templating**
5. **Deploy** desired configuration with Ansible
    * In this case, we are deploying on Juniper Devices using the junos_config module with the **replace** argument
6. Use the junos_config option **commit_check** to perform Compliance Checking. i.e. Dry Run of changes to be made

![](golden-config-flow.png)

## The End Result

Github Repo can be found here - [Golden Config Compliance Engine](https://github.com/kaon1/golden-config-engine)

[The Juniper Configuration Template](https://github.com/kaon1/golden-config-engine/blob/main/junos-golden-config-template.j2)

[The Ansible Playbook](https://github.com/kaon1/golden-config-engine/blob/main/junos-golden-config-engine.yml)

### Running the playbook

Example of **out-of-compliance** run:

```sh
(py38env_ansible) [root@nettools config-templates]# ansible-playbook -i netbox_inventory.yml --e "@extra-vars.yml" --diff junos-golden-config-engine.yml -u kaon -k
SSH password: 

PLAY [Standardize System Params on Junos Config] ***

TASK [Lookup Timezone Info by Device Site] ***
ok: [TEST01 -> localhost]

TASK [Set Time Zone Fact to be used in jinja2 template] ***
ok: [TEST01]

TASK [Grab switch software fw version from Netbox and set as integer Value] ***
ok: [TEST01]

TASK [get snmp info] ***
ok: [TEST01]

TASK [parse snmp engine id] ***
ok: [TEST01]

TASK [generate snmp_9key] ***
ok: [TEST01]

TASK [generate tacacs_key] ***
ok: [TEST01]

TASK [generate root login key per device] ***
ok: [TEST01]

TASK [Template Lookup and Config Generation] ***
ok: [TEST01 -> localhost]

TASK [Load Standard System Parameters to Juniper Device] ***
[edit system login]
-    user bob-super-admin {
-        uid 2006;
-        class super-user;
-        authentication {
-            encrypted-password "$6$Hyk9Ve...."; ## SECRET-DATA
-        }
-    }
[edit system services ssh]
-    root-login allow;
+    root-login deny;
[edit system]
+   processes {
+       web-management disable;
+   }
changed: [TEST01]

TASK [debug] ***
fatal: [TEST01]: FAILED! => {
    "msg": "Change detected, action needed"
}

PLAY RECAP ***
TEST01         : ok=10   changed=1    unreachable=0    failed=1    skipped=1    rescued=0    ignored=0   
```

Example of **in-compliance** run:
```sh
(py38env_ansible) [root@nettools config-templates]# ansible-playbook -i netbox_inventory.yml --e "@extra-vars.yml" --diff junos-golden-config-engine.yml -u kaon -k
SSH password: 

PLAY [Standardize System Params on Junos Config] ***

TASK [Lookup Timezone Info by Device Site] ***
ok: [TEST01 -> localhost]

TASK [Set Time Zone Fact to be used in jinja2 template] ***
ok: [TEST01]

TASK [Grab switch software fw version from Netbox and set as integer Value] ***
ok: [TEST01]

TASK [get snmp info] ****
ok: [TEST01]

TASK [parse snmp engine id] ***
ok: [TEST01]

TASK [generate snmp_9key] ***
ok: [TEST01]

TASK [generate tacacs_key] ***
ok: [TEST01]

TASK [generate root login key per device] ***
ok: [TEST01]

TASK [Template Lookup and Config Generation] ***
ok: [TEST01 -> localhost]

TASK [Load Standard System Parameters to Juniper Device] ***
ok: [TEST01]

PLAY RECAP ***
TTEST01         : ok=10   changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   

(py38env_ansible) [root@server config-templates]# 

```



## The Nuts and Bolts How-To
some text

### Declaring Intended State 

### Source Of Truth - Netbox

### Config Context

### Netbox Hostvars and Other Vars (TimeZone etc)

### Ansible Vault Secrets

### Jinja2 Template

### Juniper Encrypted Secrets Idempotency

### Juniper Commit_Check Feature

### First runs 

### Recurring Enforcement

### Compliance Check Mode

### Notifications

## Final Thoughts
Some text