# Cloud-Native Enterprise SOAR Device and User Containment

A collection of production-grade, context-aware Security Orchestration, Automation, and Response (SOAR) playbooks built using Azure Logic Apps and integrated with Microsoft Sentinel and Microsoft Defender for Endpoint (MDE). 

> [!IMPORTANT]
> These are production-grade playbooks I created (and sanitized). They are actively utilized by a large-scale enterprise with 100,000+ users and endpoints. Because they were engineered for a specific organization, they will need to be modified to meet your company's specific business attributes.

> [!WARNING]
> These templates are provided as anonymized reference implementations, not one-click marketplace-style deployments. They are based on working Microsoft Sentinel / Logic Apps automation currently used in a large enterprise environment, but they depend on tenant-specific resources that cannot be safely or reliably exported into a public repository.
> The goal of this repository is to share the architecture, workflow logic, and implementation patterns behind the automation—not to provide a guaranteed plug-and-play deployment for every Azure environment.

---

# 1. [Automaed Device Isolation](./sentinel-device-isolation/)
* **Primary Function:** Automated endpoint containment utilizing the Microsoft Defender for Endpoint (MDE) API.
* **Key Features:**
  * Pre-containment state evaluation checking to prevent unnecessary disruptions.
  * Automatic system classification loops separating standard workstations from mission-critical servers.
  * An asynchronous verification engine polling the MDE machine actions API every 60 seconds to guarantee confirmation of isolation.
  * Integrated Azure OpenAI (`gpt-4o-mini`) agent block generating briefs for SOC analysts with Enterprise Data Protection (EDP).


## Simplified Logic App Flowchart

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

# 2. [Automaed User Containment](./sentinel-user-disruption/)
* **Primary Function:** Automated user profile lookup via Microsoft Graph and identity containment via a custom corporate Identity API.
* **Key Features:**
  * Dynamic active-hours logic parsing (for SOC on-call).
  * Automated VIP/Executive Identity protection bypass loops targeting high-profile roles to mitigate critical business downtime risks.
  * Native ITSM ticketing integration for rapid service-desk handoffs.

## Simplified Logic App Flowchart


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
