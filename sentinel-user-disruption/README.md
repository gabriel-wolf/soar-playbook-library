# Automated Incident Response: User Containment

An enterprise-grade Azure Logic App playbook that automatically responds to high-fidelity Sentinel incidents by disabling at-risk user accounts, generating ITSM tickets, and routing urgent executive notifications for VIP accounts.

## What This Playbook Does

- Triggers from Microsoft Sentinel incident creation or incident updates.
- Resolves account details using Microsoft Graph via Managed Service Identity.
- Detects VIP users based on configured executive job titles.
- Applies automated account disablement through a custom Identity API call.
- Sends structured ITSM ticket notifications for standard users.
- Sends elevated VIP notifications to executive and SOC distribution lists.
- Supports safe test mode via `DisableInProduction` and `SendProductionEmail` runtime flags.


## Workflow Architecture

### Trigger

- [`deploy/automation-rules/sentinel-on-creation.json`](deploy/automation-rules/sentinel-on-creation.json) fires on incident creation when the title matches one of the configured high-fidelity keywords.
- [`deploy/automation-rules/sentinel-on-update.json`](deploy/automation-rules/sentinel-on-update.json) fires on incident update when new alerts are added and the title matches one of the configured keywords.

### Simplified Logic App Flowchart

```mermaid
flowchart TD
    %% Trigger & Initialization
    Trigger([Sentinel Incident Created]) --> StatusCheck{Is Incident Eligible?}
    StatusCheck -->|No| Cancel([❌ Terminate Playbook])
    StatusCheck -->|Yes| Init[Initialize Variables]

    %% User Correlation
    Init --> DiscoverUsers

    subgraph DiscoverUsers [🔍 Identify the Correct User]
        direction TB

        CollectUsers[Extract Related User Entities]
        CollectUsers --> Normalize[Normalize & Remove Duplicates]

        Normalize --> SingleUser{Only One User Found?}

        SingleUser -->|Yes| SelectUser[Select Matching User]
        SingleUser -->|No| SearchAlerts[Query Sentinel Alerts]

        SearchAlerts --> AlertReady{Matching Alerts Found?}
        AlertReady -->|No| Delay[Wait 30 Seconds]
        Delay --> SearchAlerts

        AlertReady -->|Yes| Correlate[Correlate Accounts to Alert Titles]
        Correlate --> MatchFound{Unique Match Identified?}

        MatchFound -->|Yes| SelectUser
        MatchFound -->|No| CompareIncident[Compare Incident Alert Metadata]
        CompareIncident --> SelectUser
    end

    %% Identity Processing
    SelectUser --> IdentityLoop

    subgraph IdentityLoop [🔁 Process Selected Account]
        direction TB

        GetProfile[Retrieve User & On-Prem Identity]
        GetProfile --> BuildContext[Generate Incident Context]
        BuildContext --> CheckHours[Determine Business Hours]
        CheckHours --> CheckVIP[Determine VIP Status]

        CheckVIP --> DisableGate{Should Account Be Automatically Disabled?}

        DisableGate -->|Yes| DisableAccount[Disable Account & Reset Password]
        DisableGate -->|No| NotifyVIP[Escalate VIP Without Disabling]
    end

    %% Notification Routing
    DisableAccount --> EmailMode
    NotifyVIP --> EmailMode

    EmailMode{Production Notifications Enabled?}

    EmailMode -->|Yes| ITSM[Create ITSM Recovery Ticket]
    ITSM --> VIPRoute{VIP User?}

    VIPRoute -->|Yes| ExecEmail[Notify Executive Recovery Team]
    VIPRoute -->|No| SOCEmail[Notify SOC]

    EmailMode -->|No| TestTicket[Test ITSM Ticket]
    TestTicket --> TestEmail[Test SOC Notification]

    %% Completion
    ExecEmail --> Complete([✅ Playbook Complete])
    SOCEmail --> Complete
    TestEmail --> Complete
```

## Notification Behavior

When the playbook runs, it generates contextual incident notifications that include:

- User identity details (display name, UPN, sAMAccountName, job title)
- Containment status and audit context
- ITSM ticket creation and account recovery guidance
- Escalation instructions for VIP accounts

![SOC Email Notification](../imgs/soc-email-notification.png)

## Required Connections and Permissions

### Azure Logic App Connections

- `azuresentinel` - incident trigger and incident update handling
- `office365` - sending email notifications
- `keyvault` - retrieving the Identity API token secret

### Azure Managed Identity

The playbook uses the Logic App system-assigned managed identity to authenticate to Microsoft Graph.

### Key Vault Secret

- The workflow retrieves `IdentityAPI` from Key Vault.
- Grant the Logic App managed identity `Get` permission for the Key Vault secret.

## Deployment Files

- [`deploy/logic-app/workflow.json`](deploy/logic-app/workflow.json)
- [`deploy/automation-rules/sentinel-on-creation.json`](deploy/automation-rules/sentinel-on-creation.json)
- [`deploy/automation-rules/sentinel-on-update.json`](deploy/automation-rules/sentinel-on-update.json)

## Deployment Parameters

### Logic App ARM Template Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `logicAppName` | String | `AutoIsolate-Playbook` | Name of the Logic App resource |
| `location` | String | `[resourceGroup().location]` | Azure deployment region |
| `itsmEmailRecipient` | String | `itsupport@contoso.com` | ITSM ticket recipient |
| `socEmailRecipient` | String | `soc@contoso.com` | SOC alert recipient |
| `executiveEmailRecipients` | String | `soc@contoso.com,soc-manager@contoso.com,ciso@contoso.com` | VIP notification recipients |

### Logic App Runtime Parameters

| Parameter | Type | Default | Purpose |
|---|---|---|---|
| `DisableInProduction` | Bool | `true` | Enables production account disablement |
| `SendProductionEmail` | Bool | `true` | Enables production email notifications |
| `DISABLE` | String | `Active` | Indicates active deployment mode |

### Automation Rule Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `workspaceName` | String | none | Sentinel Log Analytics workspace name |
| `subscriptionId` | String | `[subscription().subscriptionId]` | Azure subscription containing the Logic App |
| `resourceGroupName` | String | none | Resource group containing the Logic App |
| `logicAppName` | String | `AutoContain-Playbook` | Target Logic App workflow name |
| `highFidelityIncidentKeywords` | Array | See defaults below | Incident titles that trigger automation |

#### Default `highFidelityIncidentKeywords`

- `Generic High-Risk Condition Alpha`
- `Generic High-Risk Condition Beta`
- `Suspicious Credential Activity Example`
- `Potential Attacker Infrastructure Detection Pattern`

## Automation Rule Behavior

- `sentinel-on-creation.json` triggers when a matching incident is created.
- `sentinel-on-update.json` triggers when a matching incident is updated with new alerts.
- Both rules run the same Logic App workflow.

## Deployment Example

```bash
az deployment group create \
  --resource-group <your-rg-name> \
  --template-file deploy/logic-app/workflow.json \
  --parameters \
    logicAppName="AutoIsolate-Playbook" \
    location="<your-region>" \
    itsmEmailRecipient="itsupport@contoso.com" \
    socEmailRecipient="soc@contoso.com" \
    executiveEmailRecipients="soc@contoso.com,soc-manager@contoso.com,ciso@contoso.com"

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

- Review Logic App run history for failures in `For_Each_Account`, Graph API calls, or the identity disablement steps.
- Confirm the `keyvault` connection can retrieve the `IdentityAPI` secret.
- Ensure the `office365` connection is authorized and can send mail.
- If account disablement does not occur, verify `DisableInProduction` is `true`.
- If emails do not send, verify `SendProductionEmail` is `true`.

## Notes

- This playbook evaluates business hours in Central Standard Time.
- VIP detection is based on `jobTitle` matching and may need customization.
- Update the account containment endpoint from `https://api.contoso.com/identity/user/.../Account` to your production API endpoint.
