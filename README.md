
# STIG-CISCO-IOS-XE-L2S
Automated STIG Compliance for Cisco IOS-XE Layer 2 Switches.

Status: 🛠️ In Development - Functional for CAT II and CAT III Evaluation/Remediation.

This repository provides an Ansible-based framework to automate the security hardening of Cisco IOS-XE switches in accordance with the Security Technical Implementation Guides (STIG). It handles the full lifecycle of compliance: from discovering the current state of the network to evaluating security gaps and remediating vulnerabilities.

* [What is a STIG?](#what-is-a-stig)
* [Roles Overview](#roles-overview)
* [1. Discover](#1-discover)
* [2. Evaluate](#2-evaluate)
* [3. Remediate](#3-remediate)
* [STIG Viewer](#stig-viewer)
* [Repository Structure](#repository-structure)
* [Requirements](#requirements)
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
* Enforces 802.1X (Dot1x) authentication on access ports.
* Configures BPDU Guard, Portfast, and Storm Control.
* Disables unneeded services (HTTP, Telnet, CDP on untrusted ports).
* Ensures secure VTY line access and AAA configurations.
* More function in progress

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

~~~
#This section is to enable the roles to provide CKLB formatted file for STIG Viewer
# To enable or disable this functionality
stig_viewer: true
~~~~

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
* **Collections:** 
  * `cisco.ios`
  * `ansible.utils`

* **Connectivity:** SSH access to switches with Enable/Privileged EXEC rights.
* **STIG Version:** Designed against the Cisco IOS-XE L2S STIG.

---

## Usage
perform a full check-and-fix for a switch:

This repo is best designed to run the STIG roles as part of an Ansible Automation Platform `Job-Template` or `Workflow`. 

### Step1: Clone Repo

### Step2: Define the Global Variables `vars/main.yml
# Variables Checklist: `vars/main.yml`

The following variables must be defined in `vars/main.yml` to ensure the **Evaluate** and **Remediate** roles function correctly. These variables control global settings, STIG viewer integration, and the desired security state of the switch.

## Core Framework Variables

| Variable Name | Purpose | Example Value |
| :--- | :--- | :--- |
| `stig_viewer` | Enables or disables the generation of CKLB files for the DISA STIG Viewer. | `true` |
| `cklb_path` | The local directory where generated CKLB audit files are saved. | `/tmp/checklists/` |
| `repository['path']` | The base path to the project directory used by handlers to write findings. | `/runner/project/` |

---

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

## AAP Screenshots
########## NEED TO DO ###########


