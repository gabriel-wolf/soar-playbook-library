# Autonomous Incident Response: AI-Driven Host Isolation

An advanced, context-aware Azure Logic App playbook engineered to autonomously contain compromised network assets using the Microsoft Defender for Endpoint (MDE) API. This workflow integrates an Azure OpenAI LLM agent directly into the orchestration loop to interpret alert data, determine context, and generate natural language summaries for the SOC.

## Analyst Notification Output
When an endpoint containment event triggers, the SOC receives a priority notification detailing the machine states, the isolation outcome, and the exact LLM summary mapping the root-cause alert behavior:

![SOC Device Email Notification](../imgs/soc-device-email-notification.png)

## Workflow Architecture & Defensive Guardrails
1. **Trigger:** Activated via Microsoft Sentinel incident webhook definitions.
2. **Pre-Disruption Validation:** - **Closed Control:** Evaluates if the incident status has updated to `Closed` before executing containment logic.
   - **Prevented Control:** Evaluates incident metadata to detect if the underlying endpoint controls already successfully blocked or prevented the malicious process, canceling downstream containment to reduce operational disruption.
3. **Host Discovery & Scoping:** Queries the Graph Security API to dynamically iterate through all active network host entities linked to the alert (`For_Each_Host`).
4. **Autonomous AI Summarization (`Summary_Agent`):** Calls an Azure OpenAI model (`gpt-4o-mini`) within the loop. The model parses the raw, unformatted alert metadata stream and synthesizes a concise, 1-2 line natural language briefing on the malicious behavior from a pre-isolation perspective.
5. **Operating System & Asset Classification:** Queries MDE to evaluate the endpoint OS family (`osPlatform`). If the target system is identified as critical infrastructure (e.g., `Server` or `Linux`), the playbook dynamically routes alerts and scales down aggressive automation loops to maintain system availability.
6. **State-Validation Loop (`Until` Block):** Initiates a conditional check looping every 60 seconds (up to 15 retries) checking the MDE machine actions log to programmatically confirm if full network isolation successfully completed on the wire.
7. **Resilient Notification Routing:** Depending on the isolation state (Success vs. Failure) and asset classification (Workstation vs. Server), the playbook modifies parameter variables to target the appropriate tier-level analyst response queues.

## Configuration & Environment Requirements
This playbook depends on API connections configured for the following managed identities:
* `azuresentinel`
* `wdatp` (Microsoft Defender for Endpoint)
* `keyvault` (Hosting Service Principal tokens for Azure OpenAI connectivity)
* `office365`

## Deployment Parameters
Tailor the workflow parameters to define environment tolerances:
* `ISOLATE_CONTROL`: Master toggle for blocking active network host paths.
* `AFTERHOURS_CONTROL`: restrains/allows automated overrides outside core university hours.
* `SIMULATE_EMAIL` / `CLOSED_CONTROL`: Diagnostic integers to test the processing paths without inducing broad environment impact.