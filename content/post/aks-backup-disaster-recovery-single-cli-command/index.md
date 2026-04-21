---
title: 'AKS Backup and Disaster Recovery: Configuring Azure Kubernetes Service Backup with a Single CLI Command'
date: 2026-04-21T13:17:05+00:00
author: Mike Hacker
tags:
- App Modernization
- Security
- How To
categories:
- Infrastructure
summary: A hands-on infrastructure guide covering the simplified AKS backup experience via Azure CLI, including backup vault setup, Bicep templates, restore procedures, and business continuity considerations for government agencies.
draft: false
image_prompt: A massive steel vault door standing partially open in a cavernous industrial space, with warm golden light spilling outward from inside. Behind the vault, a grid of interconnected gears and clockwork mechanisms fills the wall, some spinning slowly. In the foreground, a thick chain links a heavy iron anchor to the vault handle, symbolizing secure, immovable protection. Dramatic cinematic lighting casts deep shadows across brushed metal surfaces. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
audio: audio.mp3
---

Kubernetes workloads are increasingly the backbone of government digital services, from citizen portals to internal data pipelines. But running containers in production is only half the equation: protecting those workloads against data loss, accidental deletion, and regional outages is a compliance requirement that many agencies are still catching up on.

Azure Backup for AKS has matured significantly over the past year. The most impactful recent change is a **single Azure CLI command** that automates the entire backup configuration workflow, replacing what used to be a multi-step, error-prone process. This post walks through the end-to-end experience: infrastructure-as-code setup with Bicep, the simplified CLI onboarding, restore procedures, and the compliance considerations that matter for state and local government.

## What AKS Backup Actually Protects

Before diving into commands, it is worth understanding the scope. Azure Backup for AKS protects two things:

1. **Cluster state** (Kubernetes resource manifests): Deployments, Services, ConfigMaps, Secrets, Ingress rules, CRDs, and other API objects are serialized and stored in a blob container you designate.
2. **Persistent Volume data**: CSI driver-based Azure Disks and Azure Files (SMB) volumes are backed up as incremental snapshots.

Backups are organized into two tiers:

- **Operational Tier** - snapshots and blob data stored in your own subscription, same region as the cluster. Supports both Azure Disk and Azure Files volumes.
- **Vault Tier** - offsite copies stored outside your tenant (Azure Disk-based volumes only, up to 100 disks at 1 TB each). When the Backup vault uses geo-redundant storage with Cross Region Restore enabled, Vault Tier backups are available in the Azure paired region for disaster recovery.

The minimum backup frequency is every 4 hours, with options for 6, 8, 12, and 24-hour intervals. Retention can extend up to 360 days.

For details, see the official [AKS Backup overview](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-backup-overview) and [support matrix](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-backup-support-matrix).

## The Simplified Single-Command Backup Experience

Historically, configuring AKS backup required you to manually:

1. Create a Backup vault
2. Create a backup policy
3. Provision a storage account and blob container
4. Install the Backup extension in the cluster
5. Assign RBAC roles to the extension identity
6. Enable Trusted Access between the vault and cluster
7. Initialize and create a backup instance

That is seven distinct steps, each with its own set of CLI commands and potential for misconfiguration. The new `az dataprotection enable-backup trigger` command collapses all of this into a single invocation:

```bash
# Ensure dataprotection extension is version 1.9.0+
az extension add -n dataprotection --upgrade
az extension show -n dataprotection --query version -o tsv

# Enable backup with a single command
az dataprotection enable-backup trigger \
  --datasource-type AzureKubernetesService \
  --datasource-id /subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.ContainerService/managedClusters/<aks-cluster>
```

When you run this, Azure Backup automatically:

- Validates the AKS cluster state and backup compatibility
- Creates (or reuses) a region-specific backup resource group
- Installs the Backup extension in the cluster if not already present
- Creates (or reuses) required storage resources
- Creates (or reuses) a Backup vault and backup policy
- Enables Trusted Access between the vault and cluster
- Initializes and creates the backup instance

### Predefined Backup Strategies

You can pass a `--backup-strategy` flag to select a predefined retention configuration:

| Strategy | Operational Tier Retention | Vault Tier Retention |
|---|---|---|
| `Week` (default) | 7 days | None |
| `Month` | 30 days | None |
| `DisasterRecovery` | 7 days | 90 days |
| `Custom` | Uses existing vault and policy | Uses existing vault and policy |

For government agencies with COOP (Continuity of Operations) requirements, the `DisasterRecovery` strategy is particularly relevant:

```bash
az dataprotection enable-backup trigger \
  --datasource-type AzureKubernetesService \
  --datasource-id /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.ContainerService/managedClusters/<cluster> \
  --backup-strategy DisasterRecovery
```

You can also pass a JSON configuration file to use existing resources or apply tags:

```json
{
  "tags": {
    "Owner": "platform-team@agency.gov",
    "Environment": "Production",
    "CostCenter": "IT-Infrastructure"
  }
}
```

```bash
az dataprotection enable-backup trigger \
  --datasource-type AzureKubernetesService \
  --datasource-id /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.ContainerService/managedClusters/<cluster> \
  --backup-strategy DisasterRecovery \
  --backup-configuration-file @config.json
```

See the full [CLI backup walkthrough](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-backup-using-cli) for parameter details.

## Infrastructure as Code: Bicep Template for the Backup Vault

While the single CLI command is excellent for rapid onboarding, government agencies with mature DevOps practices will want to define the Backup vault declaratively. Here is a Bicep template that creates a geo-redundant Backup vault with Cross Region Restore enabled, soft delete, and immutability settings appropriate for compliance-sensitive environments:

```bicep
@description('Name of the Backup vault')
param vaultName string

@description('Azure region for the vault')
param location string = resourceGroup().location

@description('Enable Cross Region Restore for DR')
param enableCrossRegionRestore bool = true

resource backupVault 'Microsoft.DataProtection/backupVaults@2026-03-01' = {
  name: vaultName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    storageSettings: [
      {
        datastoreType: 'VaultStore'
        type: 'GeoRedundant'
      }
    ]
    featureSettings: {
      crossRegionRestoreSettings: {
        state: enableCrossRegionRestore ? 'Enabled' : 'Disabled'
      }
      crossSubscriptionRestoreSettings: {
        state: 'Enabled'
      }
    }
    securitySettings: {
      softDeleteSettings: {
        state: 'On'
        retentionDurationInDays: 14
      }
      immutabilitySettings: {
        state: 'Unlocked'
      }
    }
    monitoringSettings: {
      azureMonitorAlertSettings: {
        alertsForAllJobFailures: 'Enabled'
      }
    }
  }
  tags: {
    Environment: 'Production'
    ManagedBy: 'Bicep'
  }
}

output vaultId string = backupVault.id
output vaultPrincipalId string = backupVault.identity.principalId
```

Deploy it with:

```bash
az deployment group create \
  --resource-group rg-backup-prod \
  --template-file backup-vault.bicep \
  --parameters vaultName=bv-aks-prod-eastus
```

For the complete Bicep resource reference, see the [Microsoft.DataProtection/backupVaults template documentation](https://learn.microsoft.com/en-us/azure/templates/microsoft.dataprotection/backupvaults?pivots=deployment-language-bicep).

## Restore Procedures

Backup is only useful if you can actually restore. AKS Backup supports three restore scenarios:

- **Original-Location Recovery (OLR)** - restore to the same cluster that was backed up
- **Alternate-Location Recovery (ALR)** - restore to a different AKS cluster in the same or different subscription
- **Cross-Region Restore** - restore Vault Tier backups to a cluster in the Azure paired region

### Step-by-Step CLI Restore

**1. List available recovery points:**

```bash
az dataprotection recovery-point list \
  --backup-instance-name $BACKUP_INSTANCE \
  --resource-group $VAULT_RG \
  --vault-name $VAULT_NAME
```

**2. Initialize a restore configuration:**

```bash
az dataprotection backup-instance initialize-restoreconfig \
  --datasource-type AzureKubernetesService > restoreconfig.json
```

This generates a JSON file where you can control exactly what gets restored:

```json
{
  "conflict_policy": "Skip",
  "include_cluster_scope_resources": true,
  "included_namespaces": ["production", "monitoring"],
  "excluded_namespaces": null,
  "persistent_volume_restore_mode": "RestoreWithVolumeData",
  "namespace_mappings": null
}
```

Key options include:
- `conflict_policy`: `Skip` (default) or `Patch` (also referred to as Update in some documentation) for handling name collisions
- `included_namespaces` / `excluded_namespaces`: fine-grained namespace selection
- `namespace_mappings`: remap namespaces during restore (e.g., restore `production` as `dr-production`)
- `persistent_volume_restore_mode`: `RestoreWithVolumeData` or `RestoreWithoutVolumeData`

**3. Build and execute the restore request (ALR example):**

```bash
# Prepare restore request for alternate cluster
az dataprotection backup-instance restore initialize-for-data-recovery \
  --datasource-type AzureKubernetesService \
  --restore-location eastus \
  --source-datastore OperationalStore \
  --recovery-point-id $RECOVERY_POINT_ID \
  --restore-configuration restoreconfig.json \
  --target-resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerService/managedClusters/<target-cluster> \
  > restorerequestobject.json

# Validate permissions
az dataprotection backup-instance validate-for-restore \
  --backup-instance-name $BACKUP_INSTANCE \
  --resource-group $VAULT_RG \
  --restore-request-object restorerequestobject.json \
  --vault-name $VAULT_NAME

# Execute restore
az dataprotection backup-instance restore trigger \
  --backup-instance-name $BACKUP_INSTANCE \
  --resource-group $VAULT_RG \
  --vault-name $VAULT_NAME \
  --restore-request-object restorerequestobject.json
```

For cross-region restore (from the Vault Tier to a paired region), add `--use-secondary-region true` and set `--source-datastore VaultStore`. You will also need to provide `staging_resource_group_id` and `staging_storage_account_id` in the restore configuration for data hydration.

The complete restore walkthrough is available at [Restore AKS using CLI](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-restore-using-cli).

## Azure Government Considerations

AKS Backup is available in Azure Government regions (US Gov Arizona, US Gov Texas, US Gov Virginia) with Operational Tier support. Government agencies should be aware of these specifics:

- **Operational Tier only in Azure Government**: As of this writing, the Azure Government regions support the Operational (Snapshot) tier. Vault Tier with geo-redundant cross-region restore is available in commercial Azure regions. Verify the latest regional support in the [support matrix](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-backup-support-matrix).
- **Endpoint differences**: Azure Government uses different service endpoints (e.g., `management.usgovcloudapi.net`, `blob.core.usgovcloudapi.net`). If scripting automation, use `az cloud set --name AzureUSGovernment` before running commands.
- **Resource provider registration**: Ensure `Microsoft.KubernetesConfiguration`, `Microsoft.DataProtection`, and `Microsoft.ContainerService` are registered in your Azure Government subscription.

See [Azure Government vs. global Azure](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) for a comprehensive comparison.

## Why This Matters for Government

Government IT leaders should pay close attention to AKS Backup for several reasons:

**Regulatory compliance**: Federal and state frameworks, including NIST SP 800-53 (CP-9 Information System Backup, CP-10 System Recovery and Reconstitution), require agencies to maintain tested backup and recovery capabilities for critical information systems. AKS Backup provides auditable, policy-driven backup with configurable retention that maps directly to these controls.

**Continuity of Operations (COOP)**: State and local agencies must maintain the ability to recover critical services during disruptions. The `DisasterRecovery` strategy with Vault Tier and Cross Region Restore provides a documented, automated path to restoring Kubernetes workloads in a paired Azure region, reducing Recovery Time Objectives (RTO) from days to hours.

**Operational simplification**: Government IT teams are often resource-constrained. The single-command backup experience eliminates an entire class of configuration errors and reduces the onboarding time for new clusters from a multi-hour process to minutes. This is particularly valuable for agencies managing multiple AKS clusters across development, staging, and production environments.

**Immutability and tamper protection**: The Backup vault supports soft delete, immutability settings, and Multi-user Authorization (MUA) which adds a second layer of approval for critical operations. These features align with zero-trust principles and help protect against ransomware or insider threats targeting backup data.

**Cost predictability**: Operational Tier backups use existing storage resources in your subscription (blob storage and disk snapshots), and the per-namespace protected instance fee is straightforward to budget via standard Azure Cost Management. There are no surprise charges: the pricing model is transparent and easy to forecast across development, staging, and production clusters.

## Application-Consistent Backups with Custom Hooks

For agencies running stateful workloads like databases on AKS, crash-consistent snapshots may not be sufficient. AKS Backup supports custom pre-backup and post-backup hooks that execute commands inside your containers before and after snapshot operations. This enables you to flush database buffers, pause write operations, or run application-specific consistency checks.

Hooks are defined as Kubernetes custom resources and deployed directly to the cluster:

```yaml
apiVersion: clusterbackup.dataprotection.microsoft.com/v1alpha1
kind: BackupHook
metadata:
  name: postgres-hook
  namespace: default
spec:
  backupHook:
  - name: pg-freeze
    includedNamespaces:
    - database
    preHooks:
      - exec:
          container: postgres
          command:
            - /bin/bash
            - -c
            - pg_isready && psql -c "SELECT pg_backup_start('aks-backup');"
          onError: Continue
          timeout: 30s
    postHooks:
      - exec:
          container: postgres
          command:
            - /bin/bash
            - -c
            - psql -c "SELECT pg_backup_stop();"
          onError: Continue
          timeout: 30s
```

## Getting Started Checklist

Here is a quick-reference checklist for agencies ready to enable AKS Backup:

1. **Register resource providers**: `Microsoft.KubernetesConfiguration`, `Microsoft.DataProtection`, `Microsoft.ContainerService`
2. **Verify CLI version**: Azure CLI 2.41+ with `dataprotection` extension 1.9.0+
3. **Ensure CSI drivers are enabled** on your AKS cluster (required for persistent volume backup)
4. **Decide on a strategy**: `Week`, `Month`, `DisasterRecovery`, or `Custom`
5. **Run the single command**: `az dataprotection enable-backup trigger`
6. **Validate**: Trigger an on-demand backup and test a restore to an alternate cluster
7. **Document your DR runbook**: Include restore commands, target cluster details, and RTO/RPO targets

## Conclusion

AKS Backup has evolved from a multi-step manual process into a streamlined, production-ready service that government agencies can deploy in minutes. The combination of a single CLI command for onboarding, Bicep templates for infrastructure-as-code, granular restore controls, and compliance-aligned security features makes it a practical foundation for Kubernetes business continuity. Whether you are running citizen-facing applications or internal microservices, investing the time to configure and test AKS Backup now pays dividends when you need it most.

## References

- [AKS Backup Overview](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-backup-overview)
- [Configure AKS Backup Using CLI](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-backup-using-cli)
- [Restore AKS Using CLI](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-restore-using-cli)
- [AKS Backup Support Matrix](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-backup-support-matrix)
- [Backup Vault Bicep Reference](https://learn.microsoft.com/en-us/azure/templates/microsoft.dataprotection/backupvaults?pivots=deployment-language-bicep)
- [Azure Government vs. Global Azure](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure)

