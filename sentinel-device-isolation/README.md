# Autonomous Incident Response: Device Isolation

An advanced, context-aware Azure Logic App playbook engineered to autonomously contain compromised network assets using the Microsoft Defender for Endpoint (MDE) API. This workflow integrates an Azure OpenAI LLM agent (`gpt-4o-mini`) directly into the orchestration loop to interpret alert data, determine context, and generate natural language summaries for the SOC to facilitate rapid incident triage.

## Analyst Notification Output

When an endpoint containment event triggers, the SOC receives a priority notification detailing:
- The affected device name and machine status
- The isolation outcome (Success or Failure)
- An AI-generated 1-2 line summary of the malicious behavior from the pre-isolation perspective
- Direct link to the Microsoft Sentinel incident for additional context


![SOC Device Email Notification](../imgs/soc-device-email-notification.png)

## Workflow Architecture

### Execution Flow

1. **Trigger:** Activated via Microsoft Sentinel incident webhook when incidents are created or updated.

2. **Pre-Disruption Validation:**
   - **Closed Control:** Evaluates if the incident status has already been updated to `Closed`, preventing unnecessary containment actions.
   - **Prevented Control:** Checks whether the underlying endpoint controls (Defender, EDR, etc.) already successfully blocked or prevented the malicious process, canceling downstream containment to reduce operational disruption.

3. **Alert Processing Loop (`For_Each_Alert`):**
   - Iterates through all alerts associated with the incident.
   - Queries the Microsoft Graph Security API to retrieve detailed alert evidence.
   - Filters for file evidence with `active` remediation status.
   - Extracts device identifiers, Azure AD device IDs, OS platform, and DNS names.

4. **Autonomous AI Summarization (`Summary_Agent`):**
   - Calls Azure OpenAI (`gpt-4o-mini`) to analyze all collected alert and device data.
   - Generates a concise, 1-2 line natural language briefing describing the malicious behavior **before isolation**.
   - Output format: `DEVICENAME.domain.com: [AI-generated summary]`

5. **Host Isolation Loop (`For_Each_Host`):**
   - Iterates through all active entities (devices) identified in the alerts.
   - For each device:
     - **Business Hours Check:** Determines if the current time is within core operating hours (8:00 AM - 8:00 PM CST).
     - **OS Classification:** Evaluates the endpoint OS platform via MDE.
     - **Server Detection:** If the target is identified as critical infrastructure (`Server` platform), scales down aggressive isolation parameters to maintain system availability.
     - **Device Details:** Retrieves computer DNS name and system properties.

6. **Conditional Isolation Logic:**
   - **Workstations:** If not after-hours, executes full network isolation.
   - **Servers:** Implements bounded isolation or escalation to human operators.
   - **After-Hours:** Routes to on-call response teams; may delay isolation pending approval.

7. **State-Validation Loop (`Until` Block):**
   - Polls the MDE machine actions log every 60 seconds.
   - Confirms network isolation completion on the wire.
   - Retries up to 15 times (max 15 minutes).
   - Confirms `Isolation Confirmed` state based on MDE machine action results.

8. **Resilient Notification Routing:**
   - Routes notifications based on:
     - Isolation outcome (Success or Failure)
     - Device classification (Workstation or Server)
     - Analyst tier and email groups
     - Business hours status

## Configuration & Environment Requirements

### Azure Resources & API Connections

This playbook requires the following Azure resources and managed identity connections:

- **Microsoft Sentinel** (`azuresentinel` connection)
- **Microsoft Defender for Endpoint** (`wdatp` connection)
- **Azure Key Vault** (`keyvault` connection) - for storing Graph API Service Principal credentials
- **Office 365 Email** (`office365` connection) - for SOC analyst notifications
- **Log Analytics Workspace** - where Sentinel incidents are stored

### Required Permissions

The Logic App's System Assigned Managed Identity requires:

- **Microsoft Sentinel:**
  - `SecurityIncident/Read` - to read incident data
  - API connection to Microsoft Sentinel

- **Microsoft Defender for Endpoint:**
  - Machine isolation and containment rights via `wdatp` connector

- **Microsoft Graph API:**
  - `SecurityAlert.ReadWrite.All` - to query security alerts and evidence

- **Azure Key Vault:**
  - `Get` secret permissions for retrieving Service Principal credentials

- **Office 365:**
  - Permission to send emails

### Azure Key Vault Setup

1. Create or identify an Azure Key Vault.
2. Store the Graph API Service Principal secret with the name specified in `secretName` parameter (default: `GraphApiSecret`).
3. Grant the Logic App managed identity `Get` secret permissions on this Key Vault.

## Deployment Parameters

Configure the following parameters when deploying the ARM template:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `logicAppName` | string | `AutoIsolate-Playbook` | The name of the Logic App workflow resource |
| `location` | string | Resource Group location | Azure region for resource deployment |
| `subscriptionId` | string | Current subscription | Target Azure subscription ID |
| `resourceGroupName` | string | Current resource group | Target resource group containing the Logic App |
| `keyVaultName` | string | `your-keyvault-name` | Name of the Key Vault storing Graph API credentials |
| `secretName` | string | `GraphApiSecret` | Name of the Key Vault secret containing the Service Principal client secret |
| `graphTenantId` | string | `00000000-0000-0000-0000-000000000000` | Azure AD tenant ID for Graph API authentication |
| `graphClientId` | string | `00000000-0000-0000-0000-000000000000` | App Registration client ID with `SecurityAlert.ReadWrite.All` permissions |

### Workflow Parameters (Inside Logic App Definition)

The Logic App workflow supports the following runtime control parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `CLOSED_CONTROL` | boolean | If `true`, skips processing if the incident is already closed |
| `PREVENTED_CONTROL` | boolean | If `true`, skips processing if alert titles contain "was prevented" or "was blocked" |
| `SENDTOSOC` | boolean | If `true`, sends email notifications to SOC team |
| `ISOLATE_CONTROL` | boolean | Master toggle for enabling network device isolation |
| `AFTERHOURS_CONTROL` | boolean | If `true`, allows automated isolation actions to run after business hours |

## Automation Rules

Two complementary automation rules trigger this playbook:

### 1. On Incident Creation
- **File:** `deploy/automation-rules/sentinel-on-creation.json`
- **Trigger:** When incidents are created
- **Conditions:**
  - Incident title contains specified threat keywords
  - Incident severity is not "Informational"
- **Action:** Runs the AutoIsolate playbook

### 2. On Incident Update
- **File:** `deploy/automation-rules/sentinel-on-update.json`
- **Trigger:** When incidents are updated
- **Conditions:**
  - Incident title contains specified threat keywords
  - New alerts were added to the incident
  - Incident severity is not "Informational"
- **Action:** Runs the AutoIsolate playbook

Both rules reference the trigger keywords array for incident title matching. Customize `triggerKeywords` in the automation rule templates for your environment.

## Deployment Instructions

1. **Prepare Azure Resources:**
   - Ensure Microsoft Sentinel is deployed in your Log Analytics workspace.
   - Create or identify an Azure Key Vault.
   - Create a Service Principal with Graph API permissions.
   - Store the Service Principal secret in Key Vault.

2. **Deploy the Logic App:**
   ```bash
   az deployment group create \
     --resource-group <your-rg-name> \
     --template-file deploy/logic-app/workflow.json \
     --parameters \
       logicAppName="AutoIsolate-Playbook" \
       location="<your-region>" \
       keyVaultName="<your-keyvault-name>" \
       secretName="GraphApiSecret" \
       graphTenantId="<your-tenant-id>" \
       graphClientId="<your-app-registration-id>"
   ```

3. **Configure API Connections:**
   - Authorize the `azuresentinel` connection to your Sentinel instance.
   - Authorize the `wdatp` connection to your tenant.
   - Authorize the `office365` connection with an account that can send SOC notifications.
   - Ensure `keyvault` connection has access to your Key Vault.

4. **Deploy Automation Rules:**
   ```bash
   az deployment group create \
     --resource-group <your-rg-name> \
     --template-file deploy/automation-rules/sentinel-on-creation.json \
     --parameters \
       workspaceName="<your-workspace-name>" \
       resourceGroupName="<your-rg-name>" \
       logicAppName="AutoIsolate-Playbook"
   
   az deployment group create \
     --resource-group <your-rg-name> \
     --template-file deploy/automation-rules/sentinel-on-update.json \
     --parameters \
       workspaceName="<your-workspace-name>" \
       resourceGroupName="<your-rg-name>" \
       logicAppName="AutoIsolate-Playbook"
   ```

5. **Customize Trigger Keywords:**
   - Edit the `triggerKeywords` array in both automation rule templates to match your organization's alert naming conventions.

6. **Test in Non-Production:**
   - Use the `CLOSED_CONTROL` and `ISOLATE_CONTROL` parameters to test the workflow safely before enabling in production.
   - Monitor the Logic App run history and MDE machine action logs to validate isolation capabilities.

## Monitoring & Troubleshooting

### Logic App Run History
- Monitor execution runs in the Azure Portal under the Logic App resource.
- Check for failures in the "For_Each_Alert" or "For_Each_Host" loops.
- Verify AI summarization outputs in the "Summary_Agent" action.

### Common Issues

| Issue | Cause | Resolution |
|-------|-------|-----------|
| Workflow fails on `Get_SP_Secret` | Key Vault connection not authorized or secret doesn't exist | Verify Logic App managed identity has `Get` permissions on Key Vault secret |
| `For_Each_Alert` loop fails | Graph API query error | Confirm Service Principal has `SecurityAlert.ReadWrite.All` permissions |
| Devices not isolated | `ISOLATE_CONTROL` is false or MDE API call fails | Check ISOLATE_CONTROL parameter and wdatp connection status |
| Email not sent | `office365` connection not authorized | Re-authorize Office 365 connection |
| Business hours detection incorrect | Timezone mismatch | Verify CST timezone is correctly offset for your region |

## Variables Used in Workflow

| Variable | Type | Purpose |
|----------|------|---------|
| `DeviceName` | string | DNS name of the device being processed |
| `IsServer` | boolean | Indicates if device is critical infrastructure (Server OS) |
| `MachineID` | string | Microsoft Defender for Endpoint device ID |
| `AadDeviceID` | string | Azure AD device ID |
| `IsAfterHours` | boolean | Whether current time is outside 8 AM - 8 PM CST |
| `IsIsolated` | boolean | Whether network isolation was successfully applied |
| `OS Platform` | string | Operating system platform of the device |
| `Isolation Confirmed` | boolean | Whether isolation was confirmed via MDE actions log |
| `ActiveEntities` | array | List of devices extracted from alert evidence |
| `ActiveAlerts` | array | List of active file evidence from processed alerts |
| `TO` | string | Email recipient(s) for SOC notification |
| `CC` | string | Carbon copy recipient(s) for SOC notification |
| `SUBJECT` | string | Email subject line |
| `BODY` | string | Email body with isolation results and AI summary |