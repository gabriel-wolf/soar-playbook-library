# Autonomous Incident Response: Device Isolation

An advanced Azure Logic App playbook that automatically isolates compromised devices using Microsoft Defender for Endpoint and Azure OpenAI summarization.

## What This Playbook Does

- Triggers from Microsoft Sentinel incident creation or incident updates.
- Retrieves device and alert evidence from Microsoft Graph and Defender for Endpoint.
- Identifies active remediation events and active device entities.
- Uses an Azure OpenAI agent (`gpt-4o-mini`) to generate concise pre-isolation summaries.
- Applies conditional network isolation with server/workstation and after-hours guardrails.
- Confirms isolation completion through MDE machine action polling.
- Sends SOC notification emails based on outcome, asset classification, and business hours.

## Workflow Architecture

### Trigger

- [`deploy/automation-rules/sentinel-on-creation.json`](deploy/automation-rules/sentinel-on-creation.json) fires on incident creation when the title matches configured trigger keywords.
- [`deploy/automation-rules/sentinel-on-update.json`](deploy/automation-rules/sentinel-on-update.json) fires on incident update when new alerts are added and the title matches configured trigger keywords.

### Logic App Design

```mermaid
flowchart TD
    %% Triggers & Initialization
    Trigger([Sentinel Incident Created]) --> GetIncident[Get Incident Details]
    GetIncident --> InitVars[Initialize Variables]
    InitVars --> GetSecret[Get API Authentication Credentials]

    %% Alert Processing Loop
    GetSecret --> ForEachAlertBlock
    
    subgraph ForEachAlertBlock [🔁 For Each Alert in Incident]
        direction TB
        GetAlertDetails[Get Security Alert Details] --> FilterRemediation{Is Alert<br>Still Active?}
        FilterRemediation -->|Yes| AppendLists[Add to Active Alerts List]
        FilterRemediation -->|No| SkipAlert[Skip Alert]
    end

    %% Control Gates
    ForEachAlertBlock --> ClosedControl{Is Incident Already Closed?}
    ClosedControl -->|Yes| TerminateClosed([❌ Terminate Playbook])
    
    ClosedControl -->|No| PreventedControl{Was Attack Automatically Blocked?}
    PreventedControl -->|Yes| TerminatePrevented([❌ Terminate Playbook])

    %% Data Preparation
    PreventedControl -->|No| ComposeIncidentURL[Generate Portal Links]
    ComposeIncidentURL --> ComposeAlerts[Compile Alert Data]
    ComposeAlerts --> SummaryAgent[Generate AI Summary of Incident]

    %% Host Isolation Loop
    SummaryAgent --> ForEachHostBlock

    subgraph ForEachHostBlock [🔁 For Each Unique Host]
        direction TB
        CheckHours[Check if Outside Business Hours] --> SetIDs[Identify Target Machine]
        SetIDs --> GetMachine[Get Device Details]
        GetMachine --> CheckOS{Check OS Type}
        
        CheckOS -->|Server / Linux| SetServerTrue[Mark as Server]
        CheckOS -->|Workstation| SetServerFalse[Mark as Workstation]
        
        SetServerTrue & SetServerFalse --> IsolationLogic{Should Device Be Isolated?}
        
        %% Isolation Actions
        IsolationLogic -->|Yes| IsolateMachine[Trigger Network Isolation]
        IsolateMachine --> TagIncident[Add 'AUTOCONTAIN' Tag to Incident]
        
        %% Do-Until Loop Structure
        TagIncident --> DoUntilBlock
        
        subgraph DoUntilBlock [🔄 Do Until: Isolation Status Confirmed]
            direction TB
            Delay5[Wait 5 Seconds] --> GetIsolateStatus[Check Isolation Progress]
            GetIsolateStatus --> EvalCondition{Is Device Isolated<br>OR 1 Hour Timeout?}
            EvalCondition -->|No| Delay5
        end
        
        %% Simulation / Fallback Path
        IsolationLogic -->|No| TestBoxFallback[Route to Test Sandbox Group]
    end

    %% Nested Notification Branching Tree
    ForEachHostBlock --> Gate1{"Is Isolation Feature Disabled?"}
    
    %% Condition 1: True
    Gate1 -->|True| SetVars1[Draft Notification:<br>Feature Disabled]
    SetVars1 --> SetToCc1[Route to On-Call Analyst]
    
    %% Condition 1: False -> Leads to Gate 2
    Gate1 -->|False| Gate2{"Is Critical Server at Risk?"}
    
    %% Condition 2: True
    Gate2 -->|True| SetVars2[Draft Alert:<br>Server at Risk]
    SetVars2 --> SetToCc2[Route to Emergency SOC Team]
    
    %% Condition 2: False -> Leads to Gate 3
    Gate2 -->|False| Gate3{"Was a Server Isolated?"}
    
    %% Condition 3: True
    Gate3 -->|True| SetVars3[Draft Update:<br>Server Isolated]
    SetVars3 --> SetToCc3[Route to Server Infrastructure Team]
    
    %% Condition 3: False -> Leads to Gate 4
    Gate3 -->|False| Gate4{"Was a Workstation Isolated?"}
    Gate4 -->|True| SetVars4[Draft Update:<br>Workstation Isolated]
    SetVars4 --> SetToCc4[Route to Desktop Support Team]
    Gate4 -->|False| SkipNotifications[Do Nothing]

    %% Reconverging into Email Control Switch
    SetToCc1 & SetToCc2 & SetToCc3 & SetToCc4 & SkipNotifications --> EmailControl{"Is Notification Override Active?"}
    EmailControl -->|No| DoNothing[Use Default Team Routing]
    EmailControl -->|Yes| OverrideRouting[Redirect All Emails to Admin Inbox]

    %% Final Isolation Verification Step
    OverrideRouting & DoNothing --> IsolateSuccessful{"Did Isolation Succeed?"}
    IsolateSuccessful -->|Yes| SendSocEmail[Send Final Status Notification Email]
    IsolateSuccessful -->|No| IsolationFailed[Log Failure Details to Audit History]
   ```


## Notification Behavior

When the playbook runs, it generates SOC-facing notifications that include:

- affected device details and alert context
- isolation outcome and confirmation status
- AI-generated behavioral summary
- a Sentinel incident link for analyst follow-up

![SOC Device Email Notification](../imgs/soc-device-email-notification.png)


## Required Connections and Permissions

### Azure Logic App Connections

- `azuresentinel` - incident trigger and incident read operations
- `wdatp` - Microsoft Defender for Endpoint isolation operations
- `keyvault` - retrieving the Graph API secret
- `office365` - sending SOC notification emails

### Microsoft Graph and Key Vault

- The Logic App retrieves the Graph API secret from Key Vault using the configured `secretName`.
- Grant the Logic App managed identity `Get` permission on the Key Vault secret.
- The Graph client needs permissions to read security alert and device evidence.

## Deployment Files

- [`deploy/logic-app/workflow.json`](deploy/logic-app/workflow.json)
- [`deploy/automation-rules/sentinel-on-creation.json`](deploy/automation-rules/sentinel-on-creation.json)
- [`deploy/automation-rules/sentinel-on-update.json`](deploy/automation-rules/sentinel-on-update.json)
## Deployment Parameters

### Logic App ARM Template Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `logicAppName` | string | `AutoIsolate-Playbook` | Name of the Logic App workflow resource |
| `location` | string | `[resourceGroup().location]` | Azure region for deployment |
| `subscriptionId` | string | `[subscription().subscriptionId]` | Subscription containing the Logic App |
| `resourceGroupName` | string | `[resourceGroup().name]` | Resource group containing the Logic App |
| `keyVaultName` | string | `your-keyvault-name` | Key Vault storing Graph credentials |
| `secretName` | string | `GraphApiSecret` | Key Vault secret name for Graph client secret |
| `graphTenantId` | string | `00000000-0000-0000-0000-000000000000` | Azure AD tenant ID for Graph authentication |
| `graphClientId` | string | `00000000-0000-0000-0000-000000000000` | Graph client ID with `SecurityAlert.ReadWrite.All` permissions |

### Logic App Runtime Parameters

| Parameter | Type | Description |
|---|---|---|
| `CLOSED_CONTROL` | boolean | Skip processing if the incident is already closed |
| `PREVENTED_CONTROL` | boolean | Skip processing if the alert title indicates prevention or blocking |
| `SENDTOSOC` | boolean | Enable SOC email notifications |
| `ISOLATE_CONTROL` | boolean | Master toggle for isolation actions |
| `AFTERHOURS_CONTROL` | boolean | Allow isolation outside business hours |

### Automation Rule Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `workspaceName` | String | none | Sentinel Log Analytics workspace name |
| `subscriptionId` | String | `[subscription().subscriptionId]` | Azure subscription containing the Logic App |
| `resourceGroupName` | String | none | Resource group containing the Logic App |
| `logicAppName` | String | `AutoIsolate-Playbook` | Target Logic App workflow name |
| `triggerKeywords` | Array | See defaults below | Incident titles that trigger automation |

#### Default `triggerKeywords`

- `Generic Threat Condition Alpha`
- `Generic Threat Condition Beta`
- `Generic Threat Condition Gamma`

## Automation Rule Behavior

- `sentinel-on-creation.json` triggers when a matching incident is created.
- `sentinel-on-update.json` triggers when a matching incident is updated with new alerts.
- Both rules invoke the same Logic App workflow.

## Deployment Example

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

## Monitoring & Troubleshooting

- Review Logic App runs for failures in `For_Each_Alert`, `For_Each_Host`, or `Summary_Agent`.
- Verify the `keyvault` connection can retrieve the Graph secret.
- Confirm `office365` is authorized and able to send email.
- If isolation does not occur, verify `ISOLATE_CONTROL` is enabled.
- If notifications do not send, verify `SENDTOSOC` is enabled.

## Notes

- This playbook evaluates business hours in Central Standard Time.
- The AI summary is generated as a 1-2 line pre-isolation summary.
- Customize `triggerKeywords` to match your environment’s incident naming conventions.
