
---
title: "Governance As Code on Azure"
date: 2026-05-21
draft: false
---

I create a portfolio-ready cloud governance project spanning AWS, Azure, Terraform, and ISO/IEC 27001 thinking. Using Terraform, AWS native governance services, and Azure Policy / monitoring services, this lab shows how to:

- Codify baseline security guardrails
- Deploy governance controls consistently across cloud platforms
- Generate compliance-oriented evidence
- Explain technical controls in the language of risk, auditability, and ISO 27001

For background:
- [check project architecture]({{% relref "posts/GaC-Project-Architecture.md" %}})
- [check ISO27001:2022 Control Mapping]({{% relref "posts/iso27001mapping.md" %}})


Below are steps followed to implement it on Azure.

# Steps

## Get your account subscription Id

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
```

## Login

```bash
az login --tenant "...."
az login
```

Once logged in successfully, create a service principal to be used by terraform and/or github whenever needed

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

az ad sp create-for-rbac \
--name "sp-governance-as-code-lab" \
--role "Contributor" \
--scopes "/subscriptions/$SUBSCRIPTION_ID"
```

If SP needs to have other roles, like User Access Administrator, then add

```bash
--role "User Access Administrator"
--scope "subscriptions/$SUBSCRIBER_ID
```

Now azure cli is ready.

## Setup terraform

appId, password, subscription Id and tenant id are of service principal "sp-governance-as-code-lab"

```bash
$ export ARM_CLIENT_ID="<appId>"
$ export ARM_CLIENT_SECRET="<password>"
$ export ARM_SUBSCRIPTION_ID="<subscriptionId>"
$ export ARM_TENANT_ID="<tenantId>"
```

Login to AZ using SP

```bash
❯ az login --service-principal \
--username "6baced.......55cd67191" \
--password=$ARM_CLIENT_SECRET \
--tenant=$ARM_TENANT_ID
```

Check

```bash
❯ az account show --output table
```

Create a resource group using SP

```bash
❯ az group create --name rg-governance-lab-dev --location eastus --tags Project=GovernanceasCode Environment=Dev Owner="Rabih"
```

## setup input variables

```bash
❯ cp terraform.tfvars.example terraform.tfvars
```

Then update variable values:

```bash
❯ nano terraform.tfvars
```

## Create the storage account

```bash
❯ az storage account create --name strg312 --resource-group rg-governance-lab-dev --location eastus --sku Standard_LRS --kind StorageV2
(SubscriptionNotFound) Subscription 3xxxxxxxxxxxxxxxx was not found.
Code: SubscriptionNotFound
Message: Subscription 3xxxxxxxxxxxxxxxx was not found.
```

Check provider registration, maybe it is not registered

```bash
❯ az provider show \
--namespace Microsoft.Storage \
--query "registrationState" \
--output table
Result
-------------
NotRegistered
```

Register it

```bash
❯ az provider register --namespace Microsoft.Storage
```

Wait a bit and check again

```bash
❯ az provider show \
--namespace Microsoft.Storage \
--query "registrationState" \
--output table
Result
----------
Registered
❯
```

Now that we are ready. Initialize terraform:

```powershell
❯ terraform init
Initializing the backend...
Initializing modules...

Initializing provider plugins...

- Finding hashicorp/azurerm versions matching "~> 4.0"...
- Finding hashicorp/random versions matching "~> 3.6"...
- Installing hashicorp/azurerm v4.71.0...
- Installed hashicorp/azurerm v4.71.0 (signed by HashiCorp)
- Installing hashicorp/random v3.8.1...
- Installed hashicorp/random v3.8.1 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.
Terraform has been successfully initialized!
You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.
If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Check formatting

```powershell
❯ terraform fmt
terraform.tfvars
```

Validate

```powershell
❯ terraform validate
Success! The configuration is valid.
```

Check terraform plan. Plan should be reviewed carefully to ensure that only what is expect is created and nothing more/less.

```powershell
❯ terraform apply -ver-file=“terraform.tfvars”
…

…

Do you want to perform these actions?

Terraform will perform the actions described above.

Only 'yes' will be accepted to approve.

Enter a value: yes

module.azure_baseline.azurerm_resource_group.main: Creating...

module.azure_baseline.azurerm_policy_definition.require_storage_https_only: Creating...

Error:

Error: creating/updating Policy Definition "require-storage-https-only-Dev": policy.DefinitionsClient#CreateOrUpdate: Failure responding to request: StatusCode=403 -- Original Error: autorest/azure: Service returned an error. Status=403 Code="AuthorizationFailed" Message="The client ‘6……91' with object id '3c9…….b291' does not have authorization to perform action 'Microsoft.Authorization/policyDefinitions/write' over scope '/subscriptions/3d4……c197dbf/providers/Microsoft.Authorization/policyDefinitions/require-storage-https-only-Dev' or the scope is invalid. If access was recently granted, please refresh your credentials."
```

Why?

Because the service principal does **not** have permission to create Azure Policy definitions.

Check current role:

```powershell
❯ az role assignment list \
--assignee "$(az ad signed-in-user show --query id -o tsv)" \
--scope "/subscriptions/$SUBSCRIPTION_ID” \
--output table
```

Assign role to SP:

```powershell
❯  az role assignment create \
--assignee-object-id "3c996f.........291" \
--assignee-principal-type ServicePrincipal \
--role "Resource Policy Contributor" \
--scope "/subscriptions/3d.......197dbf"
```

This failed as SP is not owner, or User Access Administrator

Do

```powershell
❯ az logout
❯ az login --use-device-code
❯ az account set --subscription $SUBSCRIPTION_ID
❯ az role assignment create \
--assignee-object-id "3c996f33-..........-bc36-6552e693b291" \
--assignee-principal-type ServicePrincipal \
--role "Resource Policy Contributor" \
--scope "/subscriptions/3......f"
{
"condition": null,
"conditionVersion": null,
"createdBy": null,
"createdOn": "2026-05-06T20:58:35.525368+00:00",
"delegatedManagedIdentityResourceId": null,
"description": null,
"id": "/subscriptions/3.......f/providers/Microsoft.Authorization/roleAssignments/f56e0b......42b4c",
"name": "f56e........b4c",
"principalId": "3c996f33-............-e693b291",
"principalType": "ServicePrincipal",
"roleDefinitionId": "/subscriptions/3d4bb......bf/providers/Microsoft.Authorization/roleDefinitions/36243......608",
"scope": "/subscriptions/3d4b................c197dbf",
"type": "Microsoft.Authorization/roleAssignments",
"updatedBy": "05408a29-...............2ea3bba5e5c",
"updatedOn": "2026-05-06T20:58:35.994232+00:00"
}
```

Logout then log back in as SP

```powershell
❯ az logout

❯ az login --service-principal \
- -username "6ba...........67191" \
- -password=$ARM_CLIENT_SECRET \
- -tenant=$ARM_TENANT_ID
```

Apply terraform again:

```powershell
❯ terraform apply --var-file="terraform.tfvars"
…
…
…
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
Outputs:
log_analytics_workspace_name = "law-governance-lab-dev"
policy_assignment_names = [
"assign-GovernanceAsCode-baseline-Dev",
]
policy_definition_names = [
"require-storage-https-only-Dev",
"deny-public-storage-access-Dev",
]
resource_group_name = "rg-governance-lab-dev"
storage_account_name = "sagvrn312"
```

SO WHAT DID WE BUILD??

Terraform = governance-as-code delivery tool
Azure Policy = governance control/enforcement
Log Analytics = monitoring/audit evidence
Storage Account = test resource to enforce rules against
Resource Group = boundary/scope for testing

To check whether governance is active, run:

```powershell
az policy assignment list --output table
```

```bash
❯ az policy definition list \
  --query "[?contains(name, 'storage') || contains(name, 'Storage')].[name, policyType, mode]" \
  --output table
Column1                         Column2    Column3
------------------------------  ---------  ---------
deny-public-storage-access-Dev  Custom     Indexed
require-storage-https-only-Dev  Custom     Indexed

```

get Policy Assignments

```bash
❯ az policy assignment list \
  --scope "/subscriptions/3d4bb...5c197dbf/resourceGroups/rg-governance-lab-dev" \
  --output table
DefinitionVersion    Description                                                           DisplayName                               EnforcementMode    Location    Name                                  PolicyDefinitionId                                                                                                                        ResourceGroup          Scope
-------------------  --------------------------------------------------------------------  ----------------------------------------  -----------------  ----------  ------------------------------------  ----------------------------------------------------------------------------------------------------------------------------------------  ---------------------  ----------------------------------------------------------------------------------------
1.*.*                Assigns the portfolio governance baseline to the lab resource group.  Assign Governance-as-Code Baseline (Dev)  Default            eastus      assign-GovernanceAsCode-baseline-Dev  /subscriptions/3d4b...dbf/providers/Microsoft.Authorization/policySetDefinitions/GovernanceAsCode-baseline-Dev  rg-governance-lab-dev  /subscriptions/3d4bba...-....1715c197dbf/resourceGroups/rg-governance-lab-dev

```

NEXT —> How to create a compliance alert:

- Assign your policy with **Audit** effect.
- Create a **non-compliant storage account**.
- Check compliance.
- Create an alert that triggers when non-compliant resources exist.
