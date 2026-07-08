# Cloud-Native SOAR Playbook Library

A collection of production-grade, context-aware Security Orchestration, Automation, and Response (SOAR) playbooks built using Azure Logic Apps and integrated with Microsoft Sentinel and Microsoft Defender for Endpoint (MDE). 

These workflows are engineered to bridge the gap between high-fidelity threat detection and rapid automated containment, minimizing operational blast radiuses while maintaining strict corporate safety guardrails.

---

## Repository Architecture

This repository is structured as a monorepo, where each subfolder represents a standalone, parameterizable automation module complete with its own dedicated technical blueprint and architecture breakdown.

* **imgs/** — Global assets & analyst artifact verification
* **sentinel-device-isolation/** — MDE-driven endpoint isolation with OpenAI analysis
* **sentinel-user-disruption/** — Context-aware user containment & VIP bypass logic

---

## Library Directory

### 1. [Context-Aware User Disruption](./sentinel-user-disruption/)
* **Primary Function:** Automated user profile lookup via Microsoft Graph and identity containment via a custom corporate Identity API.
* **Key Features:**
  * Dynamic active-hours logic parsing (8:00 AM – 8:00 PM CST evaluations).
  * Automated **VIP/Executive Identity protection bypass loops** targeting high-profile roles (Chancellors, Presidents, C-Suite) to mitigate critical business downtime risks.
  * Native ITSM ticketing integration for rapid service-desk handoffs.

### 2. [Autonomous Device Isolation](./sentinel-device-isolation/)
* **Primary Function:** Automated endpoint containment utilizing the Microsoft Defender for Endpoint (MDE) API.
* **Key Features:**
  * Integrated **Azure OpenAI (`gpt-4o-mini`) agent** block running zero-shot synthesis to generate unformatted, 1-2 line natural language briefs for SOC analysts.
  * Pre-containment state evaluation checking for pre-existing system blocking mechanisms (`Prevented Control`) to reduce unnecessary disruptions.
  * Automatic system classification loops separating standard workstations from mission-critical servers (`Server` / `Linux`).
  * An asynchronous verification engine (`Until` loop) polling the MDE machine actions API every 60 seconds to guarantee confirmation of isolation on the wire.

---

## Design Principles & Core Standards

Every blueprint in this framework is designed under the following engineering principles:
* **Least Privilege Identity Management:** All cloud tasks are scoped strictly around Azure Managed Service Identities (MSI) or tight Service Principal OAuth tokens, eliminating hardcoded long-lived secrets.
* **Blast Radius Containment:** Workflows include explicit testing and simulation parameters (such as `DisableInProduction` and `SendProductionEmail`) to support safe integration testing and mock execution branches in non-production landing zones.
* **Analyst-First Design:** Automation shouldn't hide data. Every playbook terminates by outputting a highly visual, HTML-formatted data card straight to priority SOC analyst communication streams, summarizing the action taken, current isolation tracking status, and direct-to-portal incident hyperlinks.

---


## Getting Started & Validation

Before deploying any templates to a live production tenant, you can discover your environment configurations and run pre-flight local syntax validation checks using our [Azure Deployment & Validation Helper Guide](./helper-commands.md).

---

## License

This repository is licensed under the [MIT License](./LICENSE) — meaning the frameworks are free to adapt, modify, and build upon with zero warranty or liability implied.
