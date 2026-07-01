# Automated Incident Disruption: Context-Aware User Containment

An enterprise-grade, production-ready Azure Logic App playbook designed to trigger automatically from Microsoft Sentinel incidents. This workflow provides rapid, automated isolation of compromised accounts while implementing strict guardrails around Executive/VIP identities and operational business hours.

## Notification Output & ServiceNow Ticketing Ingestion
When the playbook triggers, for example, on a high-severity `Risky Logon Detected - TOR involving one user` alert, the workflow dispatches targeted notification payloads:

1. **SOC & Help Desk Triage:** Instantly delivers a highly detailed, formatted incident email outlining the exact automated response state.

![SOC Email Notification](../imgs/soc-email-notification.png)

2. **Automated ServiceNow Ticketing:** For standard users, a secondary, structured notification payload is dispatched to the corporate IT Service Management (ITSM) queue. This email is formatted specifically for automated ingestion, instantly creating a Security Incident Response (SIR) ticket in **ServiceNow** to track help desk remediation and account recovery.

## Workflow Logic Architecture
1. **Trigger:** Activated via Microsoft Sentinel Webhook upon incident generation.
2. **Identity Evaluation:** Queries the Microsoft Graph API using a Managed Service Identity (MSI) to parse user metadata and check Active Directory on-premises attributes (e.g., matching the user's account identifier).
3. **Context Evaluation:** 
   - Evaluates if the current local timestamp falls within core operational business hours (8:00 AM - 8:00 PM CST).
   - Scans the user's `jobTitle` for executive keywords (`Chancellor`, `Provost`, `President`, `CIO`, `CFO`, `CMO`, `COO`) to determine **VIP Status**.
4. **Conditional Execution:**
   - **Standard Users:** Account is automatically disabled via a custom identity API patch request, password pre-expired, and a structured payload is generated to trigger the ServiceNow ticketing flow.
   - **VIP Users:** Bypasses automatic lockout to prevent critical operational downtime. The workflow instead escalates the alert directly to Tier 3 SOC leadership and the Executive Recovery Team for white-glove, hands-on remediation.
5. **Testing Guardrails:** Built-in condition checks for `DisableInProduction` and `SendProductionEmail` parameters to support safe testing and staging simulations.

## Required Microsoft Graph Permissions
To execute securely under the principle of least privilege, the underlying Managed Service Identity (MSI) requires the following permissions:
* `User.Read.All` (To query employee titles and directory information)
* `User.ReadWrite.All` or custom API proxy permissions (To execute the account containment action)

## Deployment Instructions
1. Deploy the `workflow.json` template into your Azure Logic Apps environment.
2. Ensure an Azure Key Vault connection is established containing an app token named `IdentityAPI`.
3. Configure your API connections for `azuresentinel`, `office365`, and `keyvault`.
4. Configure the Playbook parameters (`DisableInProduction`, `SendProductionEmail`) to control the blast radius during initial deployment testing.