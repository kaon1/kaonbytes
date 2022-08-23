+++
author = "Kaon Thana"
title = "Ansible Configuration Backup Manager"
date = "2022-08-22"
description = "Create a vendor agnostic network backup utility with Ansible and AWS S3"
categories = [
    "ansible",
    "netdevops",
    "cloud",
    "netbox"
]

aliases = ["ansible-config-backups"]
image = "ansible-network-config-front.png"
+++

**There are many commericial tools** that can perform configuration backups of network gear (for example Solarwinds NCM). However, we can build an alternative tool using **Ansible** + a versioned Amazon AWS S3 Bucket + a daily scheduler. 

## Overview

In this guide, I will detail how we can build such a tool and meet the following **requirments**:
* Dynamic Inventory
* Vendor neutral system
* Version Controlled Backups
* Notfication of successful and failed backups
* Health Check of the Backup Tool
- Currently Supported Vendors/OS:
  - Opengear
  - ArubaOS
  - Cisco ASA and IOS
  - Fortios
  - Junos
  - Citrix Netscaler
  - Big-IP F5
  - Arista EOS

A **read-only** recurring Ansible playbook is a great way to get started in network automation of your enterprise gear because:

* The playbook does not make confuguration changes to your remote devices
* Your playbook has to touch every device and every type of device you have in your network (different vendors such as Cisco, Juniper, Aruba)
* Authentication and connectivty from the ansible server to the remote network devices need to function correclty on every run



## Workflow
Below is a diagram of the expected workflow operation of the ansible playbook.
1. Dynamically populate Ansible Inventory list with current devices **tagged** for backup. 
2. Depending on device OS (grabbed from netbox) call a specific **Ansible task** to perform system backup on each device
3. Store all system backups locally and sync to a versioned S3 Bucket with **Ansible S3 Sync Task**
4. Maintain a **summary of failed or successful** device backup actions and create report to Email
5. Update local status.txt file to be polled by outside **monitoring** system

![](workflow-backups.png)

## The Results

Github Repo can be found here - [Ansible Configuration Backup Manager](https://github.com/kaon1/configuration-backup-manager)

### Running the playbook

Here is an example output of running the playbook against 8 hosts each on a different network operating system (the vendors listed above). This playbook has been tested to run on over 1000 network devices in a single execution.

Command:

```sh
[root@ncm]# ansible-playbook -i netbox_inventory.yml -e "var_hosts=aruba1:asa1:f51:nss1:fortios1:ios1:junos1:opengear1" ncm-engine.yml
```

Output:

```sh
PLAY [PLAY TO BACKUP NETWORK CONFIGURATIONS] **********

TASK [delete exisiting successful_hosts file] **********
changed: [aruba1 -> localhost]

TASK [delete exisiting failed_hosts file] **********
changed: [aruba1 -> localhost]

TASK [delete total_hosts file] **********
changed: [aruba1 -> localhost]

TASK [create successful_hosts file] **********
changed: [aruba1 -> localhost]

TASK [create failed_hosts file] **********
changed: [aruba1 -> localhost]

TASK [create total_hosts file] **********
changed: [aruba1 -> localhost]

TASK [update total_hosts file] **********
changed: [aruba1 -> localhost]

TASK [include_role : generate-backups] **********

TASK [generate-backups : include_tasks] **********
included: /root/ncm/roles/generate-backups/tasks/aos-backup-config.yml for aruba1
included: /root/ncm/roles/generate-backups/tasks/asa-backup-config.yml for asa1
included: /root/ncm/roles/generate-backups/tasks/big-ip-backup-config.yml for f51
included: /root/ncm/roles/generate-backups/tasks/citrix-backup-config.yml for nss1
included: /root/ncm/roles/generate-backups/tasks/fortios-backup-config.yml for fortios1
included: /root/ncm/roles/generate-backups/tasks/ios-backup-config.yml for ios1
included: /root/ncm/roles/generate-backups/tasks/junos-backup-config.yml for junos1
included: /root/ncm/roles/generate-backups/tasks/opengear-backup-config.yml for opengear1

TASK [generate-backups : grab and download aruba config] **********
ok: [aruba1]

TASK [generate-backups : Save the backup information.] **********
changed: [aruba1 -> localhost]

TASK [generate-backups : Add SUCCESS line to file] **********
changed: [aruba1 -> localhost]

TASK [generate-backups : Backup ASA Device] **********
ok: [asa1]

TASK [generate-backups : Add SUCCESS line to file] **********
changed: [asa1 -> localhost]

TASK [generate-backups : grab and download big-ip config] **********
ok: [f51 -> localhost]

TASK [generate-backups : Save the backup information.] **********
changed: [f51 -> localhost]

TASK [generate-backups : Add SUCCESS line to file] **********
changed: [f51 -> localhost]

TASK [generate-backups : grab and download citrix config] **********
ok: [nss1]

TASK [generate-backups : Save the backup information.] **********
changed: [nss1 -> localhost]

TASK [generate-backups : Add SUCCESS line to file] **********
changed: [nss1 -> localhost]

TASK [generate-backups : Backup Fortigate Device] **********
ok: [fortios1]

TASK [generate-backups : Save the backup information.] **********
changed: [fortios1]

TASK [generate-backups : Add SUCCESS line to file] **********
changed: [fortios1 -> localhost]

TASK [generate-backups : Backup IOS Device] *****************
ok: [ios1]

TASK [generate-backups : Add SUCCESS line to file] **********
changed: [ios1 -> localhost]

TASK [generate-backups : grab and download junos config] **********
ok: [junos1]

TASK [generate-backups : Add SUCCESS line to file] **********
changed: [junos1 -> localhost]

TASK [generate-backups : grab and download opengear config] **********
changed: [opengear1]

TASK [generate-backups : Save the backup information.] **********
ok: [opengear1 -> localhost]

TASK [generate-backups : Add SUCCESS line to file] **********
changed: [opengear1 -> localhost]

PLAY [SYNC NETWORK CONFIGURATIONS TO S3 BUCKET] *************

TASK [delete exisiting s3_sync file] ************************
changed: [localhost]

TASK [create s3_sync file] **********************************
changed: [localhost]

TASK [include_role : sync-to-s3] ****************************

TASK [sync-to-s3 : Sync to S3] ******************************
changed: [localhost]

TASK [sync-to-s3 : Add status to s3 state file] *************
changed: [localhost]

PLAY [Build Email Template] *********************************

TASK [lookup file successful_hosts.txt] *********************
ok: [localhost]

TASK [lookup file failed_hosts.txt] *************************
ok: [localhost]

TASK [lookup file total_hosts.txt] **************************
ok: [localhost]

TASK [lookup file failed_s3.txt] ****************************
ok: [localhost]

TASK [Generate Backup State File] ***************************
changed: [localhost]

TASK [send email] *******************************************
ok: [localhost]

PLAY RECAP **************************************************
asa1 : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
opengear1 : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
nss1 : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
fortios1      : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
localhost                  : ok=10   changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
aruba1   : ok=11   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ios1 : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
junos1 : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
f51  : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[root@ncm]# 
```

#### Email Summary Report

The playbook generates an Email Template which gets sent out as the final task.

![](ncm-summary.png)

### The Main File
```yaml
---
- name: "PLAY TO BACKUP NETWORK CONFIGURATIONS"
  hosts: "{{ var_hosts }}"
  roles:
    - role: arubanetworks.aos_wlan_role
  vars:
    network_backup_dir: "/root/configuration-backup-manager/config-backups/"
    net_backup_filename: "{{ inventory_hostname }}-{{ ansible_host }}-config.txt"
  tasks:
    - name: delete exisiting successful_hosts file
      ansible.builtin.file:
        path: /root/configuration-backup-manager/templates/successful_hosts.txt
        state: absent
      run_once: True
      delegate_to: localhost

    - name: delete exisiting failed_hosts file
      ansible.builtin.file:
        path: /root/configuration-backup-manager/templates/failed_hosts.txt
        state: absent
      run_once: True
      delegate_to: localhost

    - name: delete total_hosts file
      ansible.builtin.file:
        path: /root/configuration-backup-manager/templates/total_hosts.txt
        state: absent
      run_once: True
      delegate_to: localhost

    - name: create successful_hosts file
      ansible.builtin.file:
        path: /root/configuration-backup-manager/templates/successful_hosts.txt
        state: touch
      run_once: True
      delegate_to: localhost

    - name: create failed_hosts file
      ansible.builtin.file:
        path: /root/configuration-backup-manager/templates/failed_hosts.txt
        state: touch
      run_once: True
      delegate_to: localhost

    - name: create total_hosts file
      ansible.builtin.file:
        path: /root/configuration-backup-manager/templates/total_hosts.txt
        state: touch
      run_once: True
      delegate_to: localhost

    - name: update total_hosts file
      ansible.builtin.lineinfile:
        path: /root/configuration-backup-manager/templates/total_hosts.txt
        line: "{{ groups['all'] | length }}"
      run_once: True
      delegate_to: localhost

    - include_role:
        name: generate-backups

- name: "SYNC NETWORK CONFIGURATIONS TO S3 BUCKET"
  hosts: localhost
  vars:
    network_backup_dir: "/root/configuration-backup-manager/config-backups/"
    net_backup_filename: "{{ inventory_hostname }}-{{ ansible_host }}-config.txt"
  tasks:
    - name: delete exisiting s3_sync file
      ansible.builtin.file:
        path: /root/configuration-backup-manager/templates/failed_s3.txt
        state: absent
      run_once: True

    - name: create s3_sync file
      ansible.builtin.file:
        path: /root/configuration-backup-manager/templates/failed_s3.txt
        state: touch
      run_once: True

    - include_role:
        name: sync-to-s3

- name: "Build Email Template"
  hosts: localhost
  tasks:
    - name: "lookup file successful_hosts.txt"
      set_fact:
        success_data: "{{ lookup('file', '/root/configuration-backup-manager/templates/successful_hosts.txt').splitlines() }}"

    - name: "lookup file failed_hosts.txt"
      set_fact:
        failed_data: "{{ lookup('file', '/root/configuration-backup-manager/templates/failed_hosts.txt').splitlines() }}"

    - name: "lookup file total_hosts.txt"
      set_fact:
        total_data: "{{ lookup('file', '/root/configuration-backup-manager/templates/total_hosts.txt') }}"

    - name: "lookup file failed_s3.txt"
      set_fact:
        s3_error: "{{ lookup('file', '/root/configuration-backup-manager/templates/failed_s3.txt').splitlines() }}"

    - name: Generate Backup State File
      template:
        src: "/root/configuration-backup-manager/templates/backup_state.j2"
        dest: "/root/configuration-backup-manager/templates/backup_state.txt"

    - name: send email
      mail:
        host: localhost
        port: 25
        sender: '<email>'
        to: '<email>'
        subject: 'Ansible NCM Job Completion'
        body: "{{ lookup('file', '/root/configuration-backup-manager/templates/backup_state.txt')}}"
```

### The Backup Tasks

#### Cisco IOS
```yaml
---
- name: IOS CISCO
  block:
    - name: Backup IOS Device
      vars:
        ansible_user: "username"
        ansible_password: pwd
      ios_config:
         backup: yes
         backup_options:
           filename: "{{ net_backup_filename }}"
           dir_path: "{{ network_backup_dir }}"
      register: backupinfo

    - name: Add SUCCESS line to file
      ansible.builtin.lineinfile:
        path: /root/configuration-backup-manager/templates/successful_hosts.txt
        line: "{{ inventory_hostname }}"
      when: backupinfo is defined
      delegate_to: localhost
      throttle: 1

  rescue:
    - name: Add ERROR line to file
      ansible.builtin.lineinfile:
        path: /root/configuration-backup-manager/templates/failed_hosts.txt
        line: "{{ inventory_hostname }}"
      delegate_to: localhost
      throttle: 1
```
#### Cisco ASA
```yaml
```
#### Juniper Junos
```yaml
```
#### Aruba AOS
```yaml
```
#### Fortinet FortiOS
```yaml
```
#### Citrix Netscaler
```yaml
```
#### Big IP F5
```yaml
```
#### OpenGear
```yaml
```
#### Arista EOS
```yaml
```

### The Netbox Dynamic Inventory

### Syncing to S3

### Notifications and Health Checking

## Final Thoughts