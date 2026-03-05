
# STIG-CISCO-IOS-XE-L2S

Automated STIG Compliance for Cisco IOS-XE Layer 2 Switches.

This repository provides an Ansible-based framework to automate the security hardening of Cisco IOS-XE switches in accordance with the **Security Technical Implementation Guides (STIG)**. It handles the full lifecycle of compliance: from discovering the current state of the network to evaluating security gaps and remediating vulnerabilities.

## Table of Contents

* [What is a STIG?](#what_is_stig)
* [Roles Overview](#roles_overview)
* [1. Discover](#1._discover)
* [2. Evaluate](#2._evaluate)
* [3. Remediate](#3._remediate)
* [STIG Viewer](#stig_viewer)
* [Repository Structure](repository_structure)
* [Requirements](#requirements)

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
* Uses `fact_diff` to highlight configuration drift.
* Groups findings by STIG Category (CAT I, II, III).
* **Reporting:** Generates `.cklb` files that can be imported into the **STIG Viewer** desktop application for formal auditing and submission.

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

Key Benefits of the Checker:
Eliminates Human Error: No more manual "copy-pasting" from a terminal to a spreadsheet.

Mass Audit Capability: Audit 100+ switches in the time it would take a human to audit one.

Continuous Compliance: Use the checker as a "Canary" in your CI/CD pipeline to alert you the moment a configuration drift occurs.

STIG Viewer Compatibility: The generated files are 100% compatible with the DISA STIG Viewer tool, allowing security teams to review findings, add comments, and sign off on compliance packages.

Workflow:
The intention is to run the included roles as part of an Ansible Automation Platform `Job-Template` or `Workflow` 
* evaluate.yml

However, it can be ran directly as a playbook for basic functionality.

Run the Checker: ansible-playbook example-site.yml --tags evaluate

Retrieve Results: Files are saved to your defined cklb_path (e.g., /tmp/inventory_hostname.cklb).

Review: Open the generated file in STIG Viewer to see a rule-by-rule breakdown of your switch’s security posture.

🛠️ DISA STIG Viewer
The STIG Viewer is the official tool used to view, manage, and answer STIG Checklists (.cklb files) generated by this repository. It allows security officers to review the findings and sign off on compliance.

Download Links
You can download the latest version of the STIG Viewer from the DISA Cyber Exchange (publicly available):

Official STIG Viewer Download Page: https://public.cyber.mil/stigs/srg-stig-tools/

Direct Link (Standard Desktop Version): Check the link above for the latest version (e.g., Version 3.x) for Windows, Linux, or macOS.

How to use with this Repo:
Run the Evaluate role in this repository to generate .cklb files.

Open the STIG Viewer application.

Go to File -> Import Checklist and select the files generated in your cklb_path.

All automated "Not a Finding" or "Open" statuses will be pre-populated based on the Ansible audit.

---

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
├── vars/               # Global variables and STIG rule toggles
├── group_vars/         # Credentials and connection settings
└── example-site.yml    # Main playbook to run the roles from ansible-playbook
└── discover.yml        # Main playbook to run the discover role from Ansible Automation Platform
└── evaluate.yml        # Main playbook to run the discover role from Ansible Automation Platform
└── remediate.yml        # Main playbook to run the discover role from Ansible Automation Platform


```

---

## Requirements

* **Ansible:** 2.10 or higher.
* **Collections:** * `cisco.ios`
* `ansible.utils`


* **Connectivity:** SSH access to switches with Enable/Privileged EXEC rights.
* **STIG Version:** Designed against the Cisco IOS-XE L2S STIG.

---

## Usage
perform a full check-and-fix for a switch:

The intention is to run these roles as part of an Ansible Automation Platform `Job-Template` or `Workflow` 
However, it can be ran directly as a playbook for basic functionality.

```bash
ansible-playbook example-site.yml -i inventory.ini --tags discover,evaluate

```

