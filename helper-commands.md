# Azure Deployment & Validation Helper Guide

This reference document outlines the command-line procedures for discovering active Azure resource configurations and validating Infrastructure-as-Code (IaC) deployment templates locally prior to staging deployments.

---

## 1. Environment Discovery Commands
Use these native Azure PowerShell cmdlets to extract the parameters and environment configurations required to execute validation and deployment scripts against your target subscription tenant.

### Get Target Subscription Information
Lists all accessible Azure subscriptions. Use the string value under the Id column for subscription-level parameter scoping.
```powershell
Get-AzSubscription | Format-Table Name, Id
```

### Get Resource Group Configurations
Lists all active resource groups within the current authenticated subscription context.
```powershell
Get-AzResourceGroup | Format-Table ResourceGroupName, Location
```

### Get Log Analytics Workspace Properties
Queries the active subscription context for all operational Log Analytics workspace components (underlying Microsoft Sentinel architecture). Use the value under Name for the workspace routing variable.
```powershell
Get-AzOperationalInsightsWorkspace | Format-Table Name, ResourceGroupName, Location
```

---

## 2. Infrastructure-as-Code Template Validation
These validation commands run pre-flight structural checks against your ARM template files. They evaluate internal expressions, bracket alignment, parameter mapping, and schema compliance directly against Azure's template processing engine without spinning up live billing resources or modifying system states.

### Validate "On Creation" Automation Rule
Run this block from your local ./deploy/automation-rules/ folder context to validate the high-fidelity incident creation automation template syntax.

```powershell
Test-AzResourceGroupDeployment `
  -ResourceGroupName "RG-NAME" `
  -TemplateFile "sentinel-on-creation.json" `
  -TemplateParameterObject @{
     workspaceName     = "WORKSPACE-NAME"
     resourceGroupName = "RG-NAME"
     logicAppName      = "LOGIC-APP-NAME"
  }
  ```

### Validate "On Update" Automation Rule
Run this block from your local ./deploy/automation-rules/ folder context to validate the updated incident automation template structure.

```powershell
Test-AzResourceGroupDeployment `
  -ResourceGroupName "RG-NAME" `
  -TemplateFile "./sentinel-on-update.json" `
  -TemplateParameterObject @{
     workspaceName     = "WORKSPACE-NAME"
     resourceGroupName = "RG-NAME"
     logicAppName      = "LOGIC-APP-NAME"
  }
  ```

### Validate Playbook Workflow Template
Run this block from your local ./deploy/logic-app/ folder context to validate the complete SOAR logic workflow envelope structure.

```powershell
Test-AzResourceGroupDeployment `
  -ResourceGroupName "RG-NAME" `
  -TemplateFile "./workflow.json" `
  -TemplateParameterObject @{
     logicAppName             = "LOGIC-APP-NAME"
     itsmEmailRecipient       = "itsupport@contoso.com"
     socEmailRecipient        = "soc@contoso.com"
     executiveEmailRecipients = "soc@contoso.com,soc-manager@contoso.com,ciso@contoso.com"
  }
  ```

---

## 3. Interpreting Script Results

* Successful Validation Pass: The terminal executes cleanly and returns an empty new prompt line with zero output text. In PowerShell tooling logic, an unrendered error pipeline denotes a 100% compliant execution check.
* Failed Validation Pass: The terminal prints an active InvalidTemplate error block object (ParameterBindingException or ParameterAlreadyBound). Review the trailing exception object strings to target the non-compliant parameters or syntax line values within the native file payload.