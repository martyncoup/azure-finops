# Azure Compute Optimization - Queries

Here you can find queries to identify compute-related cost optimization opportunities including hybrid benefit, reserved instances, and configuration improvements.

#### VMs Not Using Azure Hybrid Benefit

```
resources
| where type == "microsoft.compute/virtualmachines"
| where properties.storageProfile.osDisk.osType =~ "Windows"
| where isempty(properties.licenseType) or properties.licenseType != "Windows_Server"
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, vmSize, properties.licenseType, tags, Details
```

> **_Note:_** Windows VMs not leveraging Azure Hybrid Benefit (AHUB) can save up to 40% on compute costs if you have eligible Windows Server licenses with Software Assurance.

#### SQL VMs Not Using Azure Hybrid Benefit

```
resources
| where type == "microsoft.sqlvirtualmachine/sqlvirtualmachines"
| where properties.sqlServerLicenseType != "AHUB"
| extend sqlLicenseType = tostring(properties.sqlServerLicenseType)
| extend sqlEdition = tostring(properties.sqlImageSku)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sqlLicenseType, sqlEdition, tags, Details
```

> **_Note:_** SQL VMs with PAYG licensing can save up to 55% by applying Azure Hybrid Benefit with existing SQL Server licenses.

#### Linux VMs Not Using Azure Hybrid Benefit

```
resources
| where type == "microsoft.compute/virtualmachines"
| where properties.storageProfile.osDisk.osType =~ "Linux"
| where isempty(properties.licenseType) or properties.licenseType !in ("RHEL_BYOS", "SLES_BYOS")
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| extend imagePublisher = tostring(properties.storageProfile.imageReference.publisher)
| where imagePublisher in ("RedHat", "SUSE")
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, vmSize, imagePublisher, properties.licenseType, tags, Details
```

> **_Note:_** RHEL and SLES VMs can leverage Azure Hybrid Benefit with eligible subscriptions.

#### VMs Using Burstable SKUs at High Utilization

```
resources
| where type == "microsoft.compute/virtualmachines"
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| where vmSize startswith "Standard_B"
| extend Details = pack_all()
| project Resource=id, name, resourceGroup, subscriptionId, location, vmSize, tags, Details
```

> **_Note:_** Cross-reference with CPU credit metrics. Burstable VMs consistently depleting credits should be migrated to general-purpose SKUs for better cost-performance.

#### Virtual Machine Scale Sets Without Spot Instances

```
resources
| where type == "microsoft.compute/virtualmachinescalesets"
| where properties.virtualMachineProfile.priority != "Spot"
| extend vmSize = tostring(properties.virtualMachineProfile.hardwareProfile.vmSize)
| extend capacity = toint(sku.capacity)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, vmSize, capacity, tags, Details
```

> **_Note:_** VMSS workloads that tolerate interruptions (batch, dev/test) can save up to 90% using Spot instances.

#### VMs with Premium Disks in Dev/Test Subscriptions

```
resources
| where type == "microsoft.compute/virtualmachines"
| extend osDiskTier = tostring(properties.storageProfile.osDisk.managedDisk.storageAccountType)
| where osDiskTier has "Premium"
| join kind=inner (
    resourcecontainers
    | where type == "microsoft.resources/subscriptions"
    | where tags["environment"] in ("dev", "test", "development", "sandbox")
      or name has "dev" or name has "test" or name has "sandbox"
    | project subscriptionId
) on subscriptionId
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, vmSize, osDiskTier, tags, Details
```

> **_Note:_** Dev/Test workloads rarely need Premium SSD performance. Downgrading to Standard SSD can reduce disk costs by ~50%.

#### Unattached Premium/Ultra Disks (High-Cost Orphans)

```
resources
| where type == "microsoft.compute/disks"
| extend diskState = tostring(properties.diskState)
| where diskState == "Unattached"
| where sku.name has "Premium" or sku.name has "Ultra"
| extend diskSizeGB = toint(properties.diskSizeGB)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sku=sku.name, diskSizeGB, diskState, tags, Details
| order by diskSizeGB desc
```

> **_Note:_** Prioritises high-cost unattached disks (Premium/Ultra) for immediate cost reclamation.

#### VMs in Stopped (Not Deallocated) State

```
resources
| where type == "microsoft.compute/virtualmachines"
| where properties.extended.instanceView.powerState.code == "PowerState/stopped"
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, vmSize, tags, Details
```

> **_Note:_** VMs in "Stopped" state (not deallocated) still incur full compute charges. They should be deallocated or deleted if not needed.
