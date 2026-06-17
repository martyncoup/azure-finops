# Azure Storage Optimization - Queries

Here you can find queries to identify storage-related cost optimization opportunities including access tier mismatches, redundancy over-provisioning, and lifecycle management.

#### Storage Accounts Without Lifecycle Management

```
resources
| where type == "microsoft.storage/storageaccounts"
| where kind == "StorageV2" or kind == "BlobStorage"
| join kind=leftouter (
    resources
    | where type == "microsoft.storage/storageaccounts/managementpolicies"
    | extend storageAccountId = tostring(split(id, "/managementPolicies")[0])
    | project storageAccountId
) on $left.id == $right.storageAccountId
| where isempty(storageAccountId)
| extend accessTier = tostring(properties.accessTier)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, kind, accessTier, tags, Details
```

> **_Note:_** Storage accounts without lifecycle management policies miss automatic tiering opportunities. Implementing policies to transition blobs from Hot to Cool to Archive can yield 50-80% savings.

#### Storage Accounts with GRS/RAGRS in Non-Production

```
resources
| where type == "microsoft.storage/storageaccounts"
| where sku.name has "GRS" or sku.name has "GZRS"
| join kind=inner (
    resourcecontainers
    | where type == "microsoft.resources/subscriptions"
    | where tags["environment"] in ("dev", "test", "development", "sandbox")
      or name has "dev" or name has "test" or name has "sandbox"
    | project subscriptionId
) on subscriptionId
| extend replication = tostring(sku.name)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, replication, tags, Details
```

> **_Note:_** Geo-redundant storage in dev/test environments is typically unnecessary. Switching to LRS can reduce storage costs by ~50%.

#### Premium Storage Accounts with Low Transaction Counts

```
resources
| where type == "microsoft.storage/storageaccounts"
| where sku.tier == "Premium"
| extend replication = tostring(sku.name)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, replication, kind, tags, Details
```

> **_Note:_** Cross-reference with transaction metrics in Azure Monitor. Premium storage accounts with low IOPS may be better served by Standard tier.

#### Storage Accounts Using Classic (v1)

```
resources
| where type == "microsoft.storage/storageaccounts"
| where kind == "Storage"
| extend replication = tostring(sku.name)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, kind, replication, tags, Details
```

> **_Note:_** Classic (v1) storage accounts lack access tiering, lifecycle management, and other cost optimisation features available in GPv2.

#### Large Unattached Managed Disks

```
resources
| where type == "microsoft.compute/disks"
| extend diskState = tostring(properties.diskState)
| where diskState == "Unattached"
| extend diskSizeGB = toint(properties.diskSizeGB)
| where diskSizeGB >= 128
| extend skuName = tostring(sku.name)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, skuName, diskSizeGB, diskState, tags, Details
| order by diskSizeGB desc
```

#### Disks with Excessive Provisioned Size vs Used

```
resources
| where type == "microsoft.compute/disks"
| extend diskState = tostring(properties.diskState)
| where diskState == "Attached"
| extend diskSizeGB = toint(properties.diskSizeGB)
| where diskSizeGB >= 512
| extend skuName = tostring(sku.name)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, skuName, diskSizeGB, managedBy, tags, Details
| order by diskSizeGB desc
```

> **_Note:_** Large attached disks should be cross-referenced with OS-level disk usage metrics to identify over-provisioned storage.

#### Recovery Services Vaults with Excessive Retention

```
resources
| where type == "microsoft.recoveryservices/vaults"
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sku=sku.name, tags, Details
```

> **_Note:_** Cross-reference with backup policies to identify overly generous retention settings. Reducing retention from 99 years to business-appropriate periods significantly reduces costs.

#### Key Vaults with Soft Delete Extended Retention

```
resources
| where type == "microsoft.keyvault/vaults"
| extend softDeleteRetentionDays = toint(properties.softDeleteRetentionInDays)
| where softDeleteRetentionDays > 7
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, softDeleteRetentionDays, tags, Details
```

> **_Note:_** While soft delete is important for protection, extended retention periods (default 90 days) result in continued billing for deleted secrets, keys, and certificates.
