# Enterprise SOAR Playbooks: Automated Incident Response

> **Author:** Gabriel Wolf

A collection of production-grade, context-aware Security Orchestration, Automation, and Response (SOAR) playbooks built using Azure Logic Apps and integrated with Microsoft Sentinel and Microsoft Defender for Endpoint (MDE). 

> [!NOTE]
> These are production-grade playbooks I created (and sanitized). They are actively utilized by a large-scale enterprise with 100,000+ users and endpoints. Because they were engineered for a specific organization, they will need to be modified to meet your company's specific business attributes.

> [!WARNING]
> These templates are provided as anonymized reference implementations, not one-click marketplace-style deployments. 
> The goal of this repository is to share the architecture, workflow logic, and implementation patterns behind the automation—not to provide a guaranteed plug-and-play deployment for every Azure environment.

---

# 1. [Automated Device Isolation](./sentinel-device-isolation/)
![Status](https://img.shields.io/badge/Status-In%20Production-2ea44f?style=flat)
![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat&logo=data:image/svg%2bxml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA5NiA5NiI+PHBhdGggZmlsbD0iI2ZmZmZmZiIgZD0iTTI5LjMgNzkuNUgxLjdMNDIuNyA3LjVoMjkuM0wyOS4zIDc5LjV6TTQ4LjYgNzkuNWwxMS42LTIyLjMgMjQuMSAyMi4zSDQ4LjZ6Ii8+PC9zdmc+)
![Logic Apps](https://img.shields.io/badge/Logic%20Apps-80C010?style=flat&logo=data:image/svg%2bxml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAxOCAxOCI+PHBhdGggZD0iTTEzLjg1MSw5LjA0N0gxMC45MzlBMS41MTgsMS41MTgsMCwwLDEsOS40MjEsNy41Mjl2LTMuMkg4LjU3OXYzLjJBMS41MTgsMS41MTgsMCwwLDEsNy4wNjEsOS4wNDdINC4xNDlhMS4yLDEuMiwwLDAsMC0xLjIsMS4ydjIuMzM4aC44NDFWMTAuMjQ0YS4zNTUuMzU1LDAsMCwxLC4zNTYtLjM1NUg3LjA2MUEyLjM1MywyLjM1MywwLDAsMCw4LjgsOS4xMjVhLjI3OC4yNzgsMCwwLDEsLjQwOCwwLDIuMzUzLDIuMzUzLDAsMCwwLDEuNzM1Ljc2NGgyLjkxNmEuMzU4LjM1NCwwLDAsMSwuMzU1LjM1NXYyLjMzOGguODQxVjEwLjI0NEExLjIsMS4yLDAsMCwwLDEzLjg1MSw5LjA0N1oiIGZpbGw9IiNmZmZmZmYiLz48cmVjdCB4PSI1LjYyNiIgeT0iLTAuMDIiIHdpZHRoPSI2Ljc0NyIgaGVpZ2h0PSI2Ljc0NyIgcng9IjAuNjA0IiBmaWxsPSIjZmZmZmZmIi8+PHJlY3QgeT0iMTEuMjczIiB3aWR0aD0iNi43NDciIGhlaWdodD0iNi43NDciIHJ4PSIwLjYwNCIgZmlsbD0iI2ZmZmZmZiIvPjxyZWN0IHg9IjExLjI1MyIgeT0iMTEuMjczIiB3aWR0aD0iNi43NDciIGhlaWdodD0iNi43NDciIHJ4PSIwLjYwNCIgdHJhbnNmb3JtPSJ0cmFuc2xhdGUoMjkuMjczIDAuMDIpIHJvdGF0ZSg5MCkiIGZpbGw9IiNmZmZmZmYiLz48L3N2Zz4=)
![Sentinel](https://img.shields.io/badge/Sentinel-7F52FF?style=flat&logo=bitwarden&logoColor=white)
* **Primary Function:** Automated endpoint containment utilizing the Microsoft Defender for Endpoint (MDE) API.
* **Key Features:**
  * Pre-containment state evaluation checking to prevent unnecessary disruptions.
  * Automatic system classification loops separating standard workstations from mission-critical servers.
  * An asynchronous verification engine polling the MDE machine actions API every 60 seconds to guarantee confirmation of isolation.
  * Integrated Azure OpenAI (`gpt-4o-mini`) agent block generating briefs for SOC analysts with Enterprise Data Protection (EDP).

# 2. [Automated User Containment](./sentinel-user-disruption/)
![Status](https://img.shields.io/badge/Status-In%20Production-2ea44f?style=flat)
![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat&logo=data:image/svg%2bxml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA5NiA5NiI+PHBhdGggZmlsbD0iI2ZmZmZmZiIgZD0iTTI5LjMgNzkuNUgxLjdMNDIuNyA3LjVoMjkuM0wyOS4zIDc5LjV6TTQ4LjYgNzkuNWwxMS42LTIyLjMgMjQuMSAyMi4zSDQ4LjZ6Ii8+PC9zdmc+)
![Logic Apps](https://img.shields.io/badge/Logic%20Apps-80C010?style=flat&logo=data:image/svg%2bxml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAxOCAxOCI+PHBhdGggZD0iTTEzLjg1MSw5LjA0N0gxMC45MzlBMS41MTgsMS41MTgsMCwwLDEsOS40MjEsNy41Mjl2LTMuMkg4LjU3OXYzLjJBMS41MTgsMS41MTgsMCwwLDEsNy4wNjEsOS4wNDdINC4xNDlhMS4yLDEuMiwwLDAsMC0xLjIsMS4ydjIuMzM4aC44NDFWMTAuMjQ0YS4zNTUuMzU1LDAsMCwxLC4zNTYtLjM1NUg3LjA2MUEyLjM1MywyLjM1MywwLDAsMCw4LjgsOS4xMjVhLjI3OC4yNzgsMCwwLDEsLjQwOCwwLDIuMzUzLDIuMzUzLDAsMCwwLDEuNzM1Ljc2NGgyLjkxNmEuMzU4LjM1NCwwLDAsMSwuMzU1LjM1NXYyLjMzOGguODQxVjEwLjI0NEExLjIsMS4yLDAsMCwwLDEzLjg1MSw5LjA0N1oiIGZpbGw9IiNmZmZmZmYiLz48cmVjdCB4PSI1LjYyNiIgeT0iLTAuMDIiIHdpZHRoPSI2Ljc0NyIgaGVpZ2h0PSI2Ljc0NyIgcng9IjAuNjA0IiBmaWxsPSIjZmZmZmZmIi8+PHJlY3QgeT0iMTEuMjczIiB3aWR0aD0iNi43NDciIGhlaWdodD0iNi43NDciIHJ4PSIwLjYwNCIgZmlsbD0iI2ZmZmZmZiIvPjxyZWN0IHg9IjExLjI1MyIgeT0iMTEuMjczIiB3aWR0aD0iNi43NDciIGhlaWdodD0iNi43NDciIHJ4PSIwLjYwNCIgdHJhbnNmb3JtPSJ0cmFuc2xhdGUoMjkuMjczIDAuMDIpIHJvdGF0ZSg5MCkiIGZpbGw9IiNmZmZmZmYiLz48L3N2Zz4=)
![Sentinel](https://img.shields.io/badge/Sentinel-7F52FF?style=flat&logo=bitwarden&logoColor=white)
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
