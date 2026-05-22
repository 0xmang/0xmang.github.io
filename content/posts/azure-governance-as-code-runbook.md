---
date: '2026-04-20T20:43:04-04:00'
title: 'Governance-as-Code on Azure'
draft: false
cover:
  image: "/images/gac.jpeg"
  alt: "Governance-as-code"
  caption: "Governance-as-code"
---

This project implements governance as code on Azure by defining, deploying, and testing custom Azure Policy controls for storage account security.
It enforces three baseline controls:

- Storage accounts must use customer-managed keys for encryption at rest.
- Public blob access must be disabled.
- Secure transfer must be enabled, requiring HTTPS instead of HTTP.

The runbook walks through creating policy definitions, grouping them into an Azure Policy initiative, assigning the initiative to a resource group, triggering compliance scans, and simulating policy violations to confirm that non-compliant storage accounts are denied. It also includes an Activity Log alert so failed policy-violating storage deployments generate a notification.

This runbook shows a practical Azure governance-as-code implementation using Azure CLI and policy JSON files.

## Controls implemented

| Control | Azure enforcement method | Effect |
|---|---|---|
| Storage accounts should use customer-managed keys for encryption at rest | Azure Policy on storage encryption key source | `Deny` |
| Storage account public access should be disallowed | Azure Policy on blob public access | `Deny` |
| Secure transfer to storage accounts should be enabled | Azure Policy on `supportsHttpsTrafficOnly` | `Deny` |

> Important: These controls are enforced directly through Azure Policy because they are Azure Resource Manager properties on storage accounts.

---

## 0. Prerequisites

You need:

- Azure CLI
- Contributor or Owner on the test subscription/resource group
- `Microsoft.PolicyInsights` and `Microsoft.Insights` providers registered

```bash
az version
az login
az account show --output table
```

output:

```text
EnvironmentName    HomeTenantId                          IsDefault    Name                 State    TenantId
-----------------  ------------------------------------  -----------  -------------------  -------  ------------------------------------
AzureCloud         11111111-1111-1111-1111-111111111111  True         Dev Subscription     Enabled  11111111-1111-1111-1111-111111111111
```

Set variables:

```bash
export SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
export LOCATION="eastus"
export RG="rg-gov-code-demo"
export POLICY_PREFIX="govcode"
export EMAIL_RECEIVER="you@example.com"

az group create \
  --name "$RG" \
  --location "$LOCATION"
```

output:

```json
{
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-gov-code-demo",
  "location": "eastus",
  "name": "rg-gov-code-demo",
  "properties": {
    "provisioningState": "Succeeded"
  }
}
```

Register providers:

```bash
az provider register --namespace Microsoft.PolicyInsights
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Compute
```

Check registration:

```bash
az provider show --namespace Microsoft.PolicyInsights --query registrationState -o tsv
az provider show --namespace Microsoft.Insights --query registrationState -o tsv
```

output:

```text
Registered
Registered
```

---

## 1. Create repository structure

```bash
mkdir -p azure-governance-as-code/policies/storage
mkdir -p azure-governance-as-code/initiative
mkdir -p azure-governance-as-code/scripts

cd azure-governance-as-code
```

Expected tree:

```text
azure-governance-as-code/
├── initiative/
├── policies/
│   └── storage/
└── scripts/
```

---

## 2. Policy 1: Deny storage accounts that do not use customer-managed keys

This policy denies storage accounts unless encryption at rest uses customer-managed keys.

Create `policies/storage/deny-storage-no-cmk.rules.json`:

```bash
cat > policies/storage/deny-storage-no-cmk.rules.json <<'EOF'
{
  "if": {
    "allOf": [
      {
        "field": "type",
        "equals": "Microsoft.Storage/storageAccounts"
      },
      {
        "field": "Microsoft.Storage/storageAccounts/encryption.keySource",
        "notEquals": "Microsoft.Keyvault"
      }
    ]
  },
  "then": {
    "effect": "[parameters('effect')]"
  }
}
EOF
```

Create `policies/storage/deny-storage-no-cmk.parameters.json`:

```bash
cat > policies/storage/deny-storage-no-cmk.parameters.json <<'EOF'
{
  "effect": {
    "type": "String",
    "metadata": {
      "displayName": "Effect",
      "description": "Enable or disable this policy."
    },
    "allowedValues": [
      "Deny",
      "Audit",
      "Disabled"
    ],
    "defaultValue": "Deny"
  }
}
EOF
```

Create the policy definition:

```bash
az policy definition create \
  --name "${POLICY_PREFIX}-deny-storage-no-cmk" \
  --display-name "Governance: storage accounts should use customer-managed keys for encryption at rest" \
  --description "Deny storage accounts unless encryption at rest uses customer-managed keys." \
  --mode Indexed \
  --rules policies/storage/deny-storage-no-cmk.rules.json \
  --params policies/storage/deny-storage-no-cmk.parameters.json
```

output:

```json
{
  "displayName": "Governance: storage accounts should use customer-managed keys for encryption at rest",
  "mode": "Indexed",
  "name": "govcode-deny-storage-no-cmk",
  "policyType": "Custom",
  "type": "Microsoft.Authorization/policyDefinitions"
}
```

---

## 3. Policy 2: Deny storage accounts that allow public blob access

Create `policies/storage/deny-storage-public-access.rules.json`:

```bash
cat > policies/storage/deny-storage-public-access.rules.json <<'EOF'
{
  "if": {
    "allOf": [
      {
        "field": "type",
        "equals": "Microsoft.Storage/storageAccounts"
      },
      {
        "field": "Microsoft.Storage/storageAccounts/allowBlobPublicAccess",
        "notEquals": false
      }
    ]
  },
  "then": {
    "effect": "[parameters('effect')]"
  }
}
EOF
```

Create `policies/storage/deny-storage-public-access.parameters.json`:

```bash
cat > policies/storage/deny-storage-public-access.parameters.json <<'EOF'
{
  "effect": {
    "type": "String",
    "metadata": {
      "displayName": "Effect",
      "description": "Enable or disable this policy."
    },
    "allowedValues": [
      "Deny",
      "Audit",
      "Disabled"
    ],
    "defaultValue": "Deny"
  }
}
EOF
```

Create the policy definition:

```bash
az policy definition create \
  --name "${POLICY_PREFIX}-deny-storage-public-access" \
  --display-name "Governance: storage account public access should be disallowed" \
  --description "Deny storage accounts unless public blob access is disabled." \
  --mode Indexed \
  --rules policies/storage/deny-storage-public-access.rules.json \
  --params policies/storage/deny-storage-public-access.parameters.json
```

output:

```json
{
  "displayName": "Governance: storage account public access should be disallowed",
  "mode": "Indexed",
  "name": "govcode-deny-storage-public-access",
  "policyType": "Custom",
  "type": "Microsoft.Authorization/policyDefinitions"
}
```

---

## 4. Policy 3: Deny storage accounts that do not enable secure transfer

This policy denies storage accounts unless secure transfer is enabled, which requires clients to use HTTPS.

Create `policies/storage/deny-storage-no-secure-transfer.rules.json`:

```bash
cat > policies/storage/deny-storage-no-secure-transfer.rules.json <<'EOF'
{
  "if": {
    "allOf": [
      {
        "field": "type",
        "equals": "Microsoft.Storage/storageAccounts"
      },
      {
        "field": "Microsoft.Storage/storageAccounts/supportsHttpsTrafficOnly",
        "notEquals": true
      }
    ]
  },
  "then": {
    "effect": "[parameters('effect')]"
  }
}
EOF
```

Create `policies/storage/deny-storage-no-secure-transfer.parameters.json`:

```bash
cat > policies/storage/deny-storage-no-secure-transfer.parameters.json <<'EOF'
{
  "effect": {
    "type": "String",
    "metadata": {
      "displayName": "Effect",
      "description": "Enable or disable this policy."
    },
    "allowedValues": [
      "Deny",
      "Audit",
      "Disabled"
    ],
    "defaultValue": "Deny"
  }
}
EOF
```

Create the policy definition:

```bash
az policy definition create \
  --name "${POLICY_PREFIX}-deny-storage-no-secure-transfer" \
  --display-name "Governance: secure transfer to storage accounts should be enabled" \
  --description "Deny storage accounts unless secure transfer is enabled." \
  --mode Indexed \
  --rules policies/storage/deny-storage-no-secure-transfer.rules.json \
  --params policies/storage/deny-storage-no-secure-transfer.parameters.json
```

output:

```json
{
  "displayName": "Governance: secure transfer to storage accounts should be enabled",
  "mode": "Indexed",
  "name": "govcode-deny-storage-no-secure-transfer",
  "policyType": "Custom",
  "type": "Microsoft.Authorization/policyDefinitions"
}
```

## 5. Create an initiative that groups all 3 storage controls

Get policy definition IDs:

```bash
export POLICY_STORAGE_CMK_ID="$(az policy definition show \
  --name "${POLICY_PREFIX}-deny-storage-no-cmk" \
  --query id -o tsv)"

export POLICY_STORAGE_PUBLIC_ACCESS_ID="$(az policy definition show \
  --name "${POLICY_PREFIX}-deny-storage-public-access" \
  --query id -o tsv)"

export POLICY_STORAGE_SECURE_TRANSFER_ID="$(az policy definition show \
  --name "${POLICY_PREFIX}-deny-storage-no-secure-transfer" \
  --query id -o tsv)"

echo "$POLICY_STORAGE_CMK_ID"
echo "$POLICY_STORAGE_PUBLIC_ACCESS_ID"
echo "$POLICY_STORAGE_SECURE_TRANSFER_ID"
```

output:

```text
/subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.Authorization/policyDefinitions/govcode-deny-storage-no-cmk
/subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.Authorization/policyDefinitions/govcode-deny-storage-public-access
/subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.Authorization/policyDefinitions/govcode-deny-storage-no-secure-transfer
```

Create `initiative/govcode-baseline.json`:

```bash
cat > initiative/govcode-baseline.json <<EOF
{
  "policyDefinitions": [
    {
      "policyDefinitionId": "$POLICY_STORAGE_CMK_ID",
      "parameters": {
        "effect": {
          "value": "Deny"
        }
      }
    },
    {
      "policyDefinitionId": "$POLICY_STORAGE_PUBLIC_ACCESS_ID",
      "parameters": {
        "effect": {
          "value": "Deny"
        }
      }
    },
    {
      "policyDefinitionId": "$POLICY_STORAGE_SECURE_TRANSFER_ID",
      "parameters": {
        "effect": {
          "value": "Deny"
        }
      }
    }
  ]
}
EOF
```

Create the initiative definition:

```bash
az policy set-definition create \
  --name "${POLICY_PREFIX}-baseline" \
  --display-name "Governance Baseline: CMK encryption, public access disabled, secure transfer enabled" \
  --description "Baseline initiative with 3 controls: customer-managed keys for storage encryption at rest, public access disabled, and secure transfer enabled." \
  --definitions initiative/govcode-baseline.json
```

output:

```json
{
  "displayName": "Governance Baseline: CMK encryption, public access disabled, secure transfer enabled",
  "name": "govcode-baseline",
  "policyType": "Custom",
  "type": "Microsoft.Authorization/policySetDefinitions"
}
```

---

## 6. Assign the initiative to the demo resource group

```bash
export SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG"

az policy assignment create \
  --name "${POLICY_PREFIX}-baseline-assignment" \
  --display-name "Governance baseline assignment" \
  --policy-set-definition "${POLICY_PREFIX}-baseline" \
  --scope "$SCOPE" \
  --location "$LOCATION" \
  --identity-type SystemAssigned
```

output:

```json
{
  "displayName": "Governance baseline assignment",
  "enforcementMode": "Default",
  "identity": {
    "type": "SystemAssigned"
  },
  "location": "eastus",
  "name": "govcode-baseline-assignment",
  "scope": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-gov-code-demo"
}
```

Check assignment:

```bash
az policy assignment list \
  --scope "$SCOPE" \
  --query "[].{name:name, displayName:displayName, enforcementMode:enforcementMode}" \
  -o table
```

output:

```text
Name                         DisplayName                     EnforcementMode
---------------------------  ------------------------------  -----------------
govcode-baseline-assignment  Governance baseline assignment  Default
```

---

## 7. Trigger an on-demand compliance scan

```bash
az policy state trigger-scan \
  --resource-group "$RG"
```

output:

```json
{
  "status": "Succeeded"
}
```

Query compliance states:

```bash
az policy state list \
  --resource-group "$RG" \
  --query "[].{resource:resourceId, policy:policyDefinitionName, compliance:complianceState}" \
  -o table
```

output:

```text
Resource                                                                 Policy                                      Compliance
-----------------------------------------------------------------------  ------------------------------------------  -----------
/subscriptions/.../resourceGroups/rg-gov-code-demo/.../storageAccounts  govcode-deny-storage-public-access       Compliant
/subscriptions/.../resourceGroups/rg-gov-code-demo/.../storageAccounts  govcode-deny-storage-no-cmk                 Compliant
```

---

## 8. Create alerting for policy-denied compliance violation attempts

This alert fires when Azure Activity Log records a failed storage account write operation in the demo resource group. A policy denial appears as a failed Administrative operation.

### 8.1 Create an action group

```bash
az monitor action-group create \
  --name "ag-govcode-policy-alerts" \
  --resource-group "$RG" \
  --short-name "govcode" \
  --action email "EmailAdmin" "$EMAIL_RECEIVER"
```

output:

```json
{
  "enabled": true,
  "groupShortName": "govcode",
  "name": "ag-govcode-policy-alerts",
  "resourceGroup": "rg-gov-code-demo"
}
```

Get the action group ID:

```bash
export ACTION_GROUP_ID="$(az monitor action-group show \
  --name "ag-govcode-policy-alerts" \
  --resource-group "$RG" \
  --query id -o tsv)"
```

### 8.2 Create an Activity Log alert rule

```bash
az monitor activity-log alert create \
  --name "alert-policy-denied-storage-write" \
  --resource-group "$RG" \
  --scope "$SCOPE" \
  --condition category=Administrative and status=Failed and operationName=Microsoft.Storage/storageAccounts/write \
  --action-group "$ACTION_GROUP_ID" \
  --description "Alert when a storage account write fails, including policy-denied storage configuration attempts."
```

output:

```json
{
  "enabled": true,
  "name": "alert-policy-denied-storage-write",
  "resourceGroup": "rg-gov-code-demo",
  "scopes": [
    "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-gov-code-demo"
  ]
}
```

Verify alert rule:

```bash
az monitor activity-log alert list \
  --resource-group "$RG" \
  --query "[].{name:name, enabled:enabled, description:description}" \
  -o table
```

output:

```text
Name                                Enabled    Description
----------------------------------  ---------  ------------------------------------------------------------
alert-policy-denied-storage-write   True       Alert when a storage account write fails, including policy...
```

---

## 9. Simulate compliance violation 1: storage account without customer-managed keys

This should be denied by policy and trigger the Activity Log alert because it is a failed storage account write operation.

```bash
export BAD_STORAGE_CMK="badcmk$RANDOM$RANDOM"

az storage account create \
  --name "$BAD_STORAGE_CMK" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --https-only true \
  --allow-blob-public-access false
```

Expected output:

```text
(ResourceRequestDisallowedByPolicy) Resource 'badcmk123456' was disallowed by policy.
Policy identifiers:
'[{"policyAssignment":{"name":"govcode-baseline-assignment"},"policyDefinition":{"name":"govcode-deny-storage-no-cmk"}}]'
```

Check Activity Log:

```bash
az monitor activity-log list \
  --resource-group "$RG" \
  --status Failed \
  --offset 1h \
  --query "[?operationName.value=='Microsoft.Storage/storageAccounts/write'].{time:eventTimestamp,status:status.value,operation:operationName.value,subStatus:subStatus.value}" \
  -o table
```

output:

```text
Time                         Status    Operation                                  SubStatus
---------------------------  --------  -----------------------------------------  ---------------------------
2026-05-22T15:42:11.123456Z  Failed    Microsoft.Storage/storageAccounts/write    Forbidden
```

You should receive an email alert from the action group.

---

## 10. Simulate compliance violation 2: storage account public access enabled

This should be denied by policy.

```bash
export BAD_STORAGE_PUBLIC="badpublic$RANDOM$RANDOM"

az storage account create \
  --name "$BAD_STORAGE_PUBLIC" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --https-only true \
  --allow-blob-public-access true
```

Expected output:

```text
(ResourceRequestDisallowedByPolicy) Resource 'badpublic123456' was disallowed by policy.
Policy identifiers:
'[{"policyAssignment":{"name":"govcode-baseline-assignment"},"policyDefinition":{"name":"govcode-deny-storage-public-access"}}]'
```

---

## 11. Simulate compliance violation 3: secure transfer disabled

This should be denied by policy.

```bash
export BAD_STORAGE_TRANSFER="badtransfer$RANDOM$RANDOM"

az storage account create \
  --name "$BAD_STORAGE_TRANSFER" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --https-only false \
  --allow-blob-public-access false
```

Expected output:

```text
(ResourceRequestDisallowedByPolicy) Resource 'badtransfer123456' was disallowed by policy.
Policy identifiers:
'[{"policyAssignment":{"name":"govcode-baseline-assignment"},"policyDefinition":{"name":"govcode-deny-storage-no-secure-transfer"}}]'
```

---

## 12. Create a compliant storage account

A fully compliant storage account needs secure transfer enabled, public access disabled, and customer-managed keys configured.

For customer-managed keys, create a Key Vault and key first.

```bash
export KV_NAME="kvgovcode$RANDOM$RANDOM"
export KEY_NAME="storage-cmk"
export GOOD_STORAGE="goodsa$RANDOM$RANDOM"

az keyvault create \
  --name "$KV_NAME" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --enable-purge-protection true \
  --enable-rbac-authorization true

az keyvault key create \
  --vault-name "$KV_NAME" \
  --name "$KEY_NAME" \
  --protection software
```

Create the storage account with a system-assigned managed identity and compliant network/security settings:

```bash
az storage account create \
  --name "$GOOD_STORAGE" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --https-only true \
  --allow-blob-public-access false \
  --assign-identity
```

Grant the storage account managed identity access to the key:

```bash
export STORAGE_PRINCIPAL_ID="$(az storage account show \
  --name "$GOOD_STORAGE" \
  --resource-group "$RG" \
  --query identity.principalId -o tsv)"

export KEYVAULT_SCOPE="$(az keyvault show \
  --name "$KV_NAME" \
  --resource-group "$RG" \
  --query id -o tsv)"

az role assignment create \
  --assignee "$STORAGE_PRINCIPAL_ID" \
  --role "Key Vault Crypto Service Encryption User" \
  --scope "$KEYVAULT_SCOPE"
```

Apply the customer-managed key to the storage account:

```bash
az storage account update \
  --name "$GOOD_STORAGE" \
  --resource-group "$RG" \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault "$KV_NAME" \
  --encryption-key-name "$KEY_NAME"
```

Verify:

```bash
az storage account show \
  --name "$GOOD_STORAGE" \
  --resource-group "$RG" \
  --query "{name:name, httpsOnly:enableHttpsTrafficOnly, publicAccess:allowBlobPublicAccess, keySource:encryption.keySource}" \
  -o table
```

output:

```text
Name          HttpsOnly    PublicAccess    KeySource
------------  -----------  --------------  ------------------
goodsa123456  True         False           Microsoft.Keyvault
```

## 13. CI/CD example: run this from GitHub Actions

Create `.github/workflows/deploy-governance.yml`:

```yaml
name: Deploy Azure Governance as Code

on:
  push:
    branches:
      - main
    paths:
      - "policies/**"
      - "initiative/**"
      - ".github/workflows/deploy-governance.yml"
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  deploy-governance:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Azure login with OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy policy definitions and initiative
        shell: bash
        run: |
          set -euo pipefail

          POLICY_PREFIX="govcode"
          RG="rg-gov-code-demo"
          LOCATION="eastus"
          SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
          SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG"

          az policy definition create \
            --name "${POLICY_PREFIX}-deny-storage-no-cmk" \
            --display-name "Governance: storage accounts should use customer-managed keys for encryption at rest" \
            --description "Deny storage accounts unless encryption at rest uses customer-managed keys." \
            --mode Indexed \
            --rules policies/storage/deny-storage-no-cmk.rules.json \
            --params policies/storage/deny-storage-no-cmk.parameters.json

          az policy definition create \
            --name "${POLICY_PREFIX}-deny-storage-public-access" \
            --display-name "Governance: storage account public access should be disallowed" \
            --description "Deny storage accounts unless public blob access is disabled." \
            --mode Indexed \
            --rules policies/storage/deny-storage-public-access.rules.json \
            --params policies/storage/deny-storage-public-access.parameters.json

          az policy definition create \
            --name "${POLICY_PREFIX}-deny-storage-no-secure-transfer" \
            --display-name "Governance: secure transfer to storage accounts should be enabled" \
            --description "Deny storage accounts unless secure transfer is enabled." \
            --mode Indexed \
            --rules policies/storage/deny-storage-no-secure-transfer.rules.json \
            --params policies/storage/deny-storage-no-secure-transfer.parameters.json

          POLICY_STORAGE_CMK_ID="$(az policy definition show \
            --name "${POLICY_PREFIX}-deny-storage-no-cmk" \
            --query id -o tsv)"

          POLICY_STORAGE_PUBLIC_ACCESS_ID="$(az policy definition show \
            --name "${POLICY_PREFIX}-deny-storage-public-access" \
            --query id -o tsv)"

          cat > initiative/govcode-baseline.generated.json <<EOF
          {
            "policyDefinitions": [
              {
                "policyDefinitionId": "$POLICY_STORAGE_CMK_ID",
                "parameters": {
                  "effect": {
                    "value": "Deny"
                  }
                }
              },
              {
                "policyDefinitionId": "$POLICY_STORAGE_PUBLIC_ACCESS_ID",
                "parameters": {
                  "effect": {
                    "value": "Deny"
                  }
                }
              }
            ]
          }
          EOF

          az policy set-definition create \
            --name "${POLICY_PREFIX}-baseline" \
            --display-name "Governance Baseline" \
            --description "Baseline initiative for storage controls." \
            --definitions initiative/govcode-baseline.generated.json

          az policy assignment create \
            --name "${POLICY_PREFIX}-baseline-assignment" \
            --display-name "Governance baseline assignment" \
            --policy-set-definition "${POLICY_PREFIX}-baseline" \
            --scope "$SCOPE" \
            --location "$LOCATION" \
            --identity-type SystemAssigned
```

> The GitHub Actions example includes the three storage governance controls. Add Key Vault and role-assignment deployment steps if you want the pipeline to also create compliant customer-managed-key storage accounts.

---

## 14. Cleanup

```bash
az policy assignment delete \
  --name "${POLICY_PREFIX}-baseline-assignment" \
  --scope "$SCOPE"

az policy set-definition delete \
  --name "${POLICY_PREFIX}-baseline"

az policy definition delete \
  --name "${POLICY_PREFIX}-deny-storage-no-cmk"

az policy definition delete \
  --name "${POLICY_PREFIX}-deny-storage-public-access"

az policy definition delete \
  --name "${POLICY_PREFIX}-deny-storage-no-secure-transfer"

az group delete \
  --name "$RG" \
  --yes \
  --no-wait
```

output:

```text
Deleting resource group rg-gov-code-demo
```

---

## Troubleshooting

### `AuthorizationFailed`

You need permissions to create policy definitions, policy assignments, alert rules, and action groups.

Common needed roles:

- Owner
- Resource Policy Contributor
- Monitoring Contributor
- Contributor for test resources

### `az policy assignment list` returns nothing

Make sure you are querying the same scope where the policy was assigned.

```bash
az policy assignment list --scope "$SCOPE" -o table
```

Also check subscription:

```bash
az account show -o table
```

### Policy violation is not showing immediately

Run:

```bash
az policy state trigger-scan --resource-group "$RG"
```

Then wait a few minutes and query again:

```bash
az policy state list --resource-group "$RG" -o table
```

### Activity Log alert did not email me

Check:

```bash
az monitor action-group show \
  --name "ag-govcode-policy-alerts" \
  --resource-group "$RG"

az monitor activity-log alert show \
  --name "alert-policy-denied-storage-write" \
  --resource-group "$RG"
```

Make sure the action group email receiver confirmed the subscription email if Azure asks for confirmation.

---

## References

- Microsoft Learn: Azure Policy custom definitions describe the standard process: identify requirements, map requirements to resource properties and aliases, choose an effect, and compose the policy definition.
- Microsoft Learn: Azure Policy examples for storage accounts include `supportsHttpsTrafficOnly` and built-in storage controls.
- Microsoft Learn: `az policy definition create` is the Azure CLI command for creating custom policy definitions.
- Microsoft Learn: `az monitor activity-log alert create` creates Activity Log alert rules.
