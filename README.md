# Enterprise SOAR Playbooks: Automated Incident Response
> **Author:** Gabriel Wolf

A collection of production-grade, context-aware Security Orchestration, Automation, and Response (SOAR) playbooks built using Azure Logic Apps and integrated with Microsoft Sentinel and Microsoft Defender for Endpoint (MDE). 

> [!IMPORTANT]
> These are production-grade playbooks I created (and sanitized). They are actively utilized by a large-scale enterprise with 100,000+ users and endpoints. Because they were engineered for a specific organization, they will need to be modified to meet your company's specific business attributes.

> [!WARNING]
> These templates are provided as anonymized reference implementations, not one-click marketplace-style deployments. They are based on working Microsoft Sentinel / Logic Apps automation currently used in a large enterprise environment, but they depend on tenant-specific resources that cannot be safely or reliably exported into a public repository.
> The goal of this repository is to share the architecture, workflow logic, and implementation patterns behind the automation—not to provide a guaranteed plug-and-play deployment for every Azure environment.

---

# 1. [Automated Device Isolation](./sentinel-device-isolation/)
* **Primary Function:** Automated endpoint containment utilizing the Microsoft Defender for Endpoint (MDE) API.
* **Key Features:**
  * Pre-containment state evaluation checking to prevent unnecessary disruptions.
  * Automatic system classification loops separating standard workstations from mission-critical servers.
  * An asynchronous verification engine polling the MDE machine actions API every 60 seconds to guarantee confirmation of isolation.
  * Integrated Azure OpenAI (`gpt-4o-mini`) agent block generating briefs for SOC analysts with Enterprise Data Protection (EDP).

# 2. [Automated User Containment](./sentinel-user-disruption/)
* **Primary Function:** Automated user profile lookup via Microsoft Graph and identity containment via a custom corporate Identity API.
* **Key Features:**
  * Dynamic active-hours logic parsing (for SOC on-call).
  * Automated VIP/Executive Identity protection bypass loops targeting high-profile roles to mitigate critical business downtime risks.
  * Native ITSM ticketing integration for rapid service-desk handoffs.
    
---


## Getting Started & Validation

Before deploying any templates to a live production tenant, you can discover your environment configurations and run pre-flight local syntax validation checks using our [Azure Deployment & Validation Helper Guide](./helper-commands.md).

---

## License

This repository is licensed under the [MIT License](./LICENSE) — meaning the frameworks are free to adapt, modify, and build upon with zero warranty or liability implied.
