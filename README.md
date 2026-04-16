
# STIG-CISCO-IOS-XE-L2S
Automated STIG Compliance for Cisco IOS-XE Layer 2 Switches.

Status: 🛠️ Functional for CAT I CAT II and CAT III Discovery/Evaluation/Remediation.

This repository provides an Ansible-based framework to automate the security hardening of Cisco IOS-XE switches in accordance with the Security Technical Implementation Guides (STIG). It handles the full lifecycle of compliance: from discovering the current state of the network to evaluating security gaps and remediating vulnerabilities.

* [What is a STIG?](#what-is-a-stig)
* [Roles Overview](#roles-overview)
* [1. Discover](#1-discover)
* [2. Evaluate](#2-evaluate)
* [3. Remediate](#3-remediate)
* [STIG Viewer](#stig-viewer)
* [Repository Structure](#repository-structure)
* [Requirements](#requirements)
* [Usage](#usage)
* [AAP Screenshots](#aap-screenshots)

---

## What is a STIG?

A **Security Technical Implementation Guide (STIG)** is a cybersecurity methodology or product used to enhance the security of Department of Defense (DoD) information systems and networks. For Cisco IOS-XE switches, the STIG defines specific configuration settings (Rules) categorized by severity:

* **CAT I (High):** Critical vulnerabilities that allow immediate access to a system.
* **CAT II (Medium):** Vulnerabilities that result in loss of confidentiality or availability.
* **CAT III (Low):** Vulnerabilities that degrade measures to protect against loss of data.

---
## Roles Overview

The project is divided into three primary roles, allowing for a phased approach to security management.

### 1. Discover

The **Discovery** role is used to build a baseline of your current network environment. Instead of manual data entry, this role queries your switches to determine their actual hardware and configuration status.

* **Key Functions:**
* Retrieves active Layer 2 interfaces.
* Identifies which ports are host-facing (access) vs. network-facing (trunk).
* Detects Voice VLAN assignments.
* **Output:** Generates `host_vars` files that serve as the "Source of Truth" for subsequent roles.

### 2. Evaluate

The **Evaluate** role performs a non-disruptive audit of the switches. It compares the current "Running Configuration" against the "STIG Desired State."

* **Key Functions:**
* Resilient Parsing: Uses "Bulletproof" Jinja2 logic to handle Cisco CLI whitespace and column alignment variations, preventing "False Passes".
* Uses `fact_diff` to highlight configuration drift.
* Groups findings by STIG Category (CAT I, II, III).
* **Optional Reporting:** Generates `.cklb` files that can be imported into the **STIG Viewer** desktop application for formal auditing and submission.
* Creates host_vars and scoped configurations for subsequent remediation.

### 3. Remediate

The **Remediate** role is the "Fix-it" engine. It takes the findings from the Evaluate role and pushes the necessary configuration commands to bring the device into 100% compliance.

* **Key Functions:**
* Template-Based Pushing: Uses Jinja2 templates to bundle interface changes into a single session, improving performance over standard Ansible loops.
* Intelligent Thresholds: Automatically scales values (like Storm Control BPS) based on detected interface speed (Gigabit vs. TenGigabit).
* VLAN Safety: Ensures "Safe" VLANs (e.g., VLAN 34) are created and named before moving violating ports.

## Stig Viewer

🔍 The STIG Viewer: From Audit to Evidence **Optional**
The STIG Viewer functionality (integrated into the evaluate role) is designed to automate the manual, labor-intensive process of security auditing. In high-security environments, simply being compliant isn't enough; you must provide Checklist (CKLB) files as evidence to the Authorizing Official (AO).

How it works: 
Automated Inspection: The tool logs into each switch and executes show commands to inspect specific STIG rules (e.g., verifying if VTP is disabled or BPDU Guard is active).

Logic-Based Evaluation: Using Jinja2 logic, the tool compares the live state against the STIG requirements.

If the configuration matches: The status is set to Open (Manual review needed) or Not a Finding.

If the configuration is missing/wrong: The status is set to Open.

CKLB Generation: The tool uses the stig_viewer.j2 template to generate a .cklb file for every switch in the inventory.
* This functionality can be enabled in the `vars/main.yml` file.

**STIG VIEWER SETUP**
[stigviewer](https://github.com/andubiel/STIG-CISCO-IOS-XE-L2S/blob/main/stigviewer.MD)

~~~
#This section is to enable the roles to provide CKLB formatted file for STIG Viewer
# To enable or disable this functionality
stig_viewer: true
~~~
## Handlers
1. Remediation Toggling (remediation.yml)
Purpose: Sets a specific boolean flag (e.g., SV_220661_remediate: true) for each Rule ID.

Function: This ensures that the Remediate role only attempts to fix rules that actually failed the audit, preventing unnecessary configuration overhead.

2. Persistence of Violators (scoped.yml)
Purpose: Writes the specific list of violating interfaces to the host_vars/iosstig_scoped.yaml file on the control node.

Function: Because standard Ansible facts are lost between playbook runs, this handler "pins" the data. For example, if ports Gi1/0/2 and Gi1/0/4 are missing Portfast, this handler saves that list so the Remediation role knows exactly which ports to target later.

3. STIG Viewer Reporting (stig_viewer.yml)
Purpose: Dynamically updates the status (Open or Not a Finding) and provides a descriptive "Finding Details" string for the switch.

Function: This data is fed into the stig_viewer.j2 template to generate the .cklb file used by security officers for formal compliance reporting.

## Repository Structure

```text
STIG-CISCO-IOS-XE-L2S/
├── roles/
│   ├── discover/       # Tasks for gathering live device data
│   ├── evaluate/       # Tasks for auditing and diffing config
│   └── remediate/      # Tasks for pushing hardening commands
├── templates/          # Jinja2 templates for STIG-compliant configs
│   ├── stig_viewer.j2  # Template for generating CKLB audit files
│   ├── cat1/           # High-severity rule templates
│   └── cat2/           # Medium-severity rule templates
|   └── cat3/           # Low-severity rule templates
├── vars/               # Global variables and STIG rule toggles
└── discover.yml        # Main playbook to run the discover role from Ansible Automation Platform
└── evaluate.yml        # Main playbook to run the evaluate role from Ansible Automation Platform
└── remediate.yml       # Main playbook to run the remediate role from Ansible Automation Platform

```
---

## Requirements

* **Ansible:** 2.10 or higher.
* **Ansible Automation Platform:** 2.4 or higher
* **Execution Environment** Default should work in most cases
* **Collections:** 
  * `cisco.ios`
  * `ansible.utils`
* **Inventory:**  A working inventory of the ios-xe switches in AAP
* **Connectivity:** SSH access to switches with Enable/Privileged EXEC rights.
* **STIG Version:** Designed against the Cisco IOS-XE L2S STIG.
* **Option STIG Viewer application installed where CFLB files are stored

---

## Usage
perform a full check-and-fix for a switch:

This repo is best designed to run the STIG roles as part of an Ansible Automation Platform `Job-Template` or `Workflow`. 

### Step1: Fork or Clone Repo
The intention is to make changes from AAP and playbooks to a repo with read/write access.

#### Option 1: Fork

![fork](/images/fork.png)


#### Option 2: Clone
~~~
git clone https://github.com/andubiel/STIG-CISCO-IOS-XE-L2S.git
~~~
When cloning a directory, you must delete the `.git` folder prior copying files into your own read/write repository
~~~
rm -rf STIG-CISCO-IOS-XE-L2S/.git
~~~

### Step2: Define the Global Variables `vars/main.yml
# Variables Checklist: `vars/main.yml`
**See Below for examples of variable usage**
**Refer to `vars/main.yml` for a complete list of required variables**

The following variables must be defined in `vars/main.yml` to ensure the **Evaluate** and **Remediate** roles function correctly. These variables control global settings, STIG viewer integration, and the desired security state of the switch.

## Core Framework Variables

| Variable Name | Purpose | Example Value |
| :--- | :--- | :--- |
| `stig_viewer` | Enables or disables the generation of CKLB files for the DISA STIG Viewer. | `true` |
| `cklb_server` | Remote server/pc to save cklb files - colocated with Stig Viewer install | `laptop` |
| `cklb_user` | User Account to access `cklb_server` | `laptop` |
| `cklb_path` | The directory where generated CKLB audit files are saved. | `/home/{{ cklb_user }}/cklb` |
| `git_url` | The base path to the project directory used by handlers to write findings. | `"http://gitea:{{password}}@aap:3000/gitea/STIG-CISCO-IOS-XE-L2S.git" ` |

## Secure Variables
The following variables are defined as custom credential types or vaulted.
| Variable Name | Purpose | Example Value |
| :--- | :--- | :--- |
| `cklb_password` | Password to access `cklb_server`. | `changeme` |
| `cklb_become_password` | Password to escalate privilege | `changeme` |
| `iosstig_radius_server_one_key` | Radius key | `keyme` |
| `iosstig_radius_server_two_key` | Radius key | `keyme` |
| `iosstig_vtp_user_password` | VTP Password | `changeme` |
| `ansible_user` | Account to SSH to switch | `netops` |
| `ansible_password` | Password/SSH Key to login to switch | `changeme` |

## Global Variables
These variables enable/disable entire roles and STIG categories
| Variable Name | Purpose | Example Value |
| :--- | :--- | :--- |
| `iosl2sstig_discover` | This example enables the discover role | `true` |
| `iosl2sstig_cat1_evaluate` | This example enables the evaluate role for Cat1 Stigs | `true` |

These variables enable/disable individual rules
| Variable Name | Purpose | Example Value |
| :--- | :--- | :--- |
| `SV_220650r539671` | This example enables the Cat 2 rules for VTP | `true` |
| `SV_220656r856278` | This example disables the Cat 2 rules for BPDU Guard | `false` |

## STIG Configuration Variables

These variables represent the "Desired State" for specific STIG rules. The **Evaluate** role compares the switch's running configuration against these values to identify violations.

| Variable Name | Relevant STIG ID | Purpose | Recommended Value |
| :--- | :--- | :--- | :--- |
| `iosstig_native_vlan` | CISC-L2-000260 | The dedicated VLAN ID to be used as the Native VLAN on all trunks. | `"33"` |
| `iosstig_not_native_vlan` | CISC-L2-000270 | A "safe" VLAN ID used to isolate access ports that were previously in the native VLAN. | `"34"` |
| `iosstig_not_native_vlan_name` | CISC-L2-000270 | The descriptive name assigned to the safe isolation VLAN. | `"NOT_NATIVE"` |
| `iosstig_storm_control_broadcast` | CISC-L2-000160 | The BPS threshold for broadcast storm control on Gigabit interfaces. | `20000000` |
| `iosstig_storm_control_unicast` | CISC-L2-000160 | The BPS threshold for unicast storm control on Gigabit interfaces. | `62000000` |

---

> **Implementation Note:** While the **Discovery** role automatically generates lists like `iosstig_access_ports`, the variables in the table above must be manually configured by the administrator in `vars/main.yml` to reflect the specific security policy of the organization. These values act as the benchmark for the "Logic-Based Evaluation" performed by the framework.

### Step3: Setup a Project in AAP to `Your` Git Repo
Refer to Step 1 above

**Example Project**
![project](/images/stig_project.png)


### Step4: Configure and Run the Discover Job-Template
Populates the source of truth for your switch interfaces.

* Creates host_vars for current access, voice, and trunk ports

#### 
Launch STIG-CISCO-IOS-XE-L2S-Discover
![discover](/images/stig_discover.png)

The following playbook and role tasks are used:
~~~
[tdubiel@aap STIG-CISCO-IOS-XE-L2S]$ tree roles/discover/
roles/discover/
└── tasks
    └── main.yml
discover.yml
~~~

The following host_vars files are created:

~~~
[tdubiel@aap STIG-CISCO-IOS-XE-L2S]$ tree host_vars/
host_vars/
└── clab-switch-leaf1
    ├── iosstig_access_ports.yaml
    ├── iosstig_trunk_ports.yaml
    ├── iosstig_unused_ports.yaml
    ├── iosstig_user_vlans.yaml
    ├── iosstig_voice_ports.yaml
    └── iosstig_voice_vlans.yaml
~~~

Example configuration:

~~~
[tdubiel@aap STIG-CISCO-IOS-XE-L2S]$ cat host_vars/clab-switch-leaf1/iosstig_access_ports.yaml 
---
iosstig_access_ports:
-   access:
        vlan: 12
    mode: access
    name: GigabitEthernet1/0/2
-   access:
        vlan: 12
    mode: access
    name: GigabitEthernet1/0/4
-   access:
        vlan: 12
    mode: access
    name: GigabitEthernet1/0/6
-   access:
        vlan: 12
    mode: access
    name: GigabitEthernet1/0/8
~~~

### Step5: Verification to move Forward

1. Review all Global and STIG variables
`vars/main.yml`

2. Review `host_vars/`

### Step6: Configure and Run the Evaluate Job-Template

Launch STIG-CISCO-IOS-XE-L2S-Evaluate

![evaluate](/images/stig_evaluate.png)

The following playbook and role tasks are used:
~~~
[tdubiel@aap STIG-CISCO-IOS-XE-L2S]$ tree roles/evaluate
roles/evaluate
├── handlers
│   ├── git_finish.yml
│   ├── git_start.yml
│   ├── main.yml
│   ├── remediation.yml
│   ├── scoped.yml
│   └── stig_viewer.yml
└── tasks
    ├── cat1
    │   ├── cat1_step1_8021x_access_interfaces.yml
    │   ├── cat1_step1_8021x_voice_interfaces.yml
    │   ├── cat1_step2_8021x_auth.yml
    │   └── cat1.yml
    ├── cat2
    │   ├── cat2_blackhole_eval.yml
    │   ├── cat2_bpdu_access_eval.yml
    │   ├── cat2_bpdu_voice_eval.yml
    │   ├── cat2_dai_eval.yml
    │   ├── cat2_dhcp_snooping_eval.yml
    │   ├── cat2_dos_access_eval.yml
    │   ├── cat2_dos_qos_eval.yml
    │   ├── cat2_dos_trunk_eval.yml
    │   ├── cat2_dos_voice_eval.yml
    │   ├── cat2_dtp_eval.yml
    │   ├── cat2_ipsg_access_eval.yml
    │   ├── cat2_ipsg_voice_eval.yml
    │   ├── cat2_stp_loop_eval.yml
    │   ├── cat2_stp_mode_eval.yml
    │   ├── cat2_udld_eval.yml
    │   ├── cat2_user_port_eval.yml
    │   ├── cat2_uufb_access_eval.yml
    │   ├── cat2_uufb_voice_eval.yml
    │   ├── cat2_vlan1_eval.yml
    │   ├── cat2_vlan1_mgmt_eval.yml
    │   ├── cat2_vlan1_trunk_eval.yml
    │   ├── cat2_vlan1_trunk_native_eval.yml
    │   ├── cat2_vtp_eval.yml
    │   └── cat2.yml
    ├── cat3
    │   ├── cat3_access_native_eval.yml
    │   ├── cat3_igmp_snooping_eval.yml
    │   ├── cat3_root_guard_eval.yml
    │   ├── cat3_storm_control_eval.yml
    │   └── cat3.yml
    └── main.yml
evaluate.yml
~~~

Additional host_vars are created:
* iosstig_remediate.yml sets `true` to rules that need to be remediated
* iosstig_scoped.yml refines remediation to only the affected ports

~~~
[tdubiel@aap STIG-CISCO-IOS-XE-L2S]$ tree host_vars/
host_vars/
└── clab-switch-leaf1
    ├── iosstig_access_ports.yaml
    ├── iosstig_remediate.yaml
    ├── iosstig_scoped.yaml
    ├── iosstig_trunk_ports.yaml
    ├── iosstig_unused_ports.yaml
    ├── iosstig_user_vlans.yaml
    ├── iosstig_voice_ports.yaml
    └── iosstig_voice_vlans.yaml
~~~

Example `iosstig_scoped.yml`
~~~
[tdubiel@aap STIG-CISCO-IOS-XE-L2S]$ cat host_vars/clab-switch-leaf1/iosstig_scoped.yaml 
# BEGIN 8021x ACCESS PORTS
access_8021x:
  - name: GigabitEthernet1/0/2
  - name: GigabitEthernet1/0/8
# END 8021x ACCESS PORTS
# BEGIN 8021x VOICE PORTS
voice_8021x:
  - name: GigabitEthernet1/0/8
# END 8021x VOICE PORTS
# BEGIN DOS ACCESS PORTS
access_dos:
  - name: GigabitEthernet1/0/2
  - name: GigabitEthernet1/0/4
  - name: GigabitEthernet1/0/6
  - name: GigabitEthernet1/0/8
# END DOS ACCESS PORTS
# BEGIN DOS VOICE PORTS
voice_dos:
  - name: GigabitEthernet1/0/8
# END DOS VOICE PORTS
# BEGIN DOS TRUNK PORTS
trunk_dos:
  - name: GigabitEthernet1/0/1
# END DOS TRUNK PORTS
# BEGIN BPDU ACCESS PORTS
access_bpdu:
  - name: GigabitEthernet1/0/2
  - name: GigabitEthernet1/0/4
  - name: GigabitEthernet1/0/6
  - name: GigabitEthernet1/0/8
# END BPDU ACCESS PORTS
# BEGIN BPDU VOICE PORTS
voice_bpdu:
  - name: GigabitEthernet1/0/8
# END BPDU VOICE PORTS
# BEGIN UUFB ACCESS PORTS
access_uufb:
  - name: GigabitEthernet1/0/2
  - name: GigabitEthernet1/0/4
  - name: GigabitEthernet1/0/6
  - name: GigabitEthernet1/0/8
# END UUFB ACCESS PORTS
# BEGIN UUFB VOICE PORTS
voice_uufb:
  - name: GigabitEthernet1/0/8
# END UUFB VOICE PORTS
# BEGIN IPSG ACCESS PORTS
access_ipsg:
  - name: GigabitEthernet1/0/2
  - name: GigabitEthernet1/0/4
  - name: GigabitEthernet1/0/6
  - name: GigabitEthernet1/0/8
# END IPSG ACCESS PORTS
# BEGIN IPSG VOICE PORTS
voice_ipsg:
  - name: GigabitEthernet1/0/8
# END IPSG VOICE PORTS
# BEGIN DTP TRUNK PORTS
trunk_dtp:
  - name: Gi1/0/3
  - name: Gi1/0/5
  - name: Gi1/0/7
# END DTP TRUNK PORTS
# BEGIN DEFAULT VLAN
vlan1_violators_admin_up:
    - name: Gi1/0/3
# END DEFAULT VLAN
# BEGIN NATIVE VLAN
native_vlan_exists: False
native_vlan_violators:
  - name: Gi1/0/1
    current_native: 1
# END NATIVE VLAN
# BEGIN ROOT GUARD FOR ACCESS SWITCH UPLINKS
root_guard_violators:
  - name: GigabitEthernet1/0/1
# END ROOT GUARD FOR ACCESS SWITCH UPLINKS
# BEGIN STORM CONTROL FOR USER PORTS
storm_control_violators:
  - name: GigabitEthernet1/0/2
  - name: GigabitEthernet1/0/4
  - name: GigabitEthernet1/0/6
  - name: GigabitEthernet1/0/8
# END STORM CONTROL FOR USER PORTS
~~~

### Step7: Optional - Check STIG VIEWER
1. Open STIG VIEWER and find the latest `saved` CKLB file

![stig_viewer1](/images/stig_viewer1.png)


2. Review findings from the open `failed` rules

![stig_viewer2](/images/stig_viewer2.png)


### Step8: Configure and Run the Remediate Job-Template

Launch STIG-CISCO-IOS-XE-L2S-Remediate
![stig_remediate](/images/stig_remediate.png)


The following playbook, role tasks, and templates are used:
~~~
[tdubiel@aap STIG-CISCO-IOS-XE-L2S]$ tree roles/remediate
roles/remediate
├── README.md
└── tasks
    ├── cat1
    │   ├── cat1_8021x_remediate.yml
    │   └── cat1.yml
    ├── cat2
    │   ├── cat2_blackhole_remediate.yml
    │   ├── cat2_bpdu_remediate.yml
    │   ├── cat2_dai_remediate.yml
    │   ├── cat2_dhcp_snooping_remediate.yml
    │   ├── cat2_dos_remediate.yml
    │   ├── cat2_dtp_remediate.yml
    │   ├── cat2_ipsg_remediate.yml
    │   ├── cat2_stp_loop_remediate.yml
    │   ├── cat2_stp_mode_remediate.yml
    │   ├── cat2_udld_remediate.yml
    │   ├── cat2_user_port_remediate.yml
    │   ├── cat2_uufb_remediate.yml
    │   ├── cat2_vlan1_mgmt_remediate.yml
    │   ├── cat2_vlan1_remediate.yml
    │   ├── cat2_vlan1_trunk_native_remediate.yml
    │   ├── cat2_vlan1_trunk_remediate.yml
    │   ├── cat2_vtp_remediate.yml
    │   └── cat2.yml
    ├── cat3
    │   ├── cat3_access_native_remediate.yml
    │   ├── cat3_igmp_snooping_remediate.yml
    │   ├── cat3_root_guard_remediate.yml
    │   ├── cat3_storm_control_remediate.yml
    │   └── cat3.yml
    └── main.yml
remediate.yml
~~~

~~~
[tdubiel@aap STIG-CISCO-IOS-XE-L2S]$ tree templates/
templates/
├── cat1
│   ├── auth.j2
│   ├── radius.j2
│   ├── remediate_8021x_access_interfaces.j2
│   ├── remediate_8021x_auth.j2
│   ├── remediate_8021x_radius.j2
│   └── remediate_8021x_voice_interfaces.j2
├── cat2
│   ├── remediate_blackhole.j2
│   ├── remediate_bpdu_access_interfaces.j2
│   ├── remediate_bpdu_voice_interfaces.j2
│   ├── remediate_dai.j2
│   ├── remediate_dhcp_snooping.j2
│   ├── remediate_dos_access_interfaces.j2
│   ├── remediate_dos_qos.j2
│   ├── remediate_dos_trunk_interfaces.j2
│   ├── remediate_dos_voice_interfaces.j2
│   ├── remediate_dtp_trunk.j2
│   ├── remediate_ipsg_access_interfaces.j2
│   ├── remediate_ipsg_voice_interfaces.j2
│   ├── remediate_stp_mode.j2
│   ├── remediate_udld.j2
│   ├── remediate_user_port.j2
│   ├── remediate_uufb_access_interfaces.j2
│   ├── remediate_uufb_voice_interfaces.j2
│   ├── remediate_vlan1.j2
│   ├── remediate_vlan1_mgmt.j2
│   ├── remediate_vlan1_trunk.j2
│   ├── remediate_vlan1_trunk_native.j2
│   └── remediate_vtp.j2
├── cat3
│   ├── remediate_cat3_access_native.j2
│   ├── remediate_igmp_snooping.j2
│   ├── remediate_root_guard.j2
│   └── remediate_storm_control.j2
└── stig_viewer.j2
~~~

### Step9: Validate the Remediation

At this point the STIG should be remediated.

1. Run the Evaluation again and review handler output
2. Check STIG VIEWER with updated CKLB file

Example Completed STIG VIEWER

![stig_viewer3](/images/stig_viewer3.png)


## Recommended Improvements
With AAP you can stitch together the aforementioned Job-Templates into a Workflow

![workflow](/images/workflow.png)

## Validated Content
These `ROLES` are in process of being refactored into a validated content collection.

**contributers are welcome to add new network STIGS here**

Reference: [network.compliance](https://github.com/redhat-cop/network.compliance)



