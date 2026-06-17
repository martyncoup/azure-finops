# Azure Right-Sizing - Queries

Here you can find queries to identify resources that may be over-provisioned and candidates for right-sizing.

#### Virtual Machines - Low CPU Usage

```
resources
| where type == "microsoft.compute/virtualmachines"
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| project id, name, resourceGroup, subscriptionId, location, vmSize, tags
| join kind=inner (
    InsightsMetrics
    | where Namespace == "Processor" and Name == "UtilizationPercentage"
    | where TimeGenerated > ago(30d)
    | summarize avgCpu=avg(Val), maxCpu=max(Val), p95Cpu=percentile(Val, 95) by _ResourceId
) on $left.id == $right._ResourceId
| where p95Cpu < 20
| project id, name, resourceGroup, subscriptionId, location, vmSize, avgCpu, maxCpu, p95Cpu, tags
| order by p95Cpu asc
```

> **_Note:_** Requires Azure Monitor agent and VM Insights enabled. VMs with P95 CPU below 20% over 30 days are flagged.

#### Virtual Machines - Low Memory Usage

```
resources
| where type == "microsoft.compute/virtualmachines"
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| project id, name, resourceGroup, subscriptionId, location, vmSize, tags
| join kind=inner (
    InsightsMetrics
    | where Namespace == "Memory" and Name == "AvailableMB"
    | where TimeGenerated > ago(30d)
    | extend totalMemMB = toreal(Tags["vm.azm.ms/memorySizeMB"])
    | extend usedPercent = (1 - (Val / totalMemMB)) * 100
    | summarize avgMem=avg(usedPercent), maxMem=max(usedPercent), p95Mem=percentile(usedPercent, 95) by _ResourceId
) on $left.id == $right._ResourceId
| where p95Mem < 30
| project id, name, resourceGroup, subscriptionId, location, vmSize, avgMem, maxMem, p95Mem, tags
| order by p95Mem asc
```

> **_Note:_** Requires Azure Monitor agent and VM Insights enabled. VMs with P95 memory usage below 30% are flagged.

#### App Service Plans - Over-provisioned

```
resources
| where type =~ "microsoft.web/serverfarms"
| where properties.numberOfSites > 0
| extend workerCount = toint(properties.numberOfWorkers)
| extend currentSku = tostring(sku.name)
| extend currentTier = tostring(sku.tier)
| where workerCount > 1
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, currentSku, currentTier, numberOfSites=properties.numberOfSites, workerCount, tags, Details
```

> **_Note:_** App Service Plans with multiple workers but few sites may be over-provisioned for their workload.

#### SQL Databases - DTU/vCore Over-provisioned

```
resources
| where type == "microsoft.sql/servers/databases"
| where sku.name != "ElasticPool"
| where name != "master"
| extend currentSku = tostring(sku.name)
| extend currentTier = tostring(sku.tier)
| extend maxSizeGB = todouble(properties.maxSizeBytes) / 1073741824
| extend currentSizeGB = todouble(properties.currentSizeBytes) / 1073741824
| where maxSizeGB > 0
| extend usagePercent = (currentSizeGB / maxSizeGB) * 100
| where usagePercent < 25
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, currentSku, currentTier, currentSizeGB, maxSizeGB, usagePercent, tags, Details
| order by usagePercent asc
```

> **_Note:_** SQL Databases using less than 25% of provisioned storage capacity are flagged as candidates for tier reduction.

#### Azure Kubernetes Service - Node Pool Sizing

```
resources
| where type == "microsoft.containerservice/managedclusters"
| mv-expand pool = properties.agentPoolProfiles
| extend poolName = tostring(pool.name)
| extend vmSize = tostring(pool.vmSize)
| extend nodeCount = toint(pool.['count'])
| extend minCount = toint(pool.minCount)
| extend maxCount = toint(pool.maxCount)
| extend enableAutoScaling = tobool(pool.enableAutoScaling)
| where enableAutoScaling == false or nodeCount == maxCount
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, poolName, vmSize, nodeCount, minCount, maxCount, enableAutoScaling, tags, Details
```

> **_Note:_** AKS node pools without autoscaling enabled, or running at max capacity, may benefit from autoscaler configuration or right-sizing.

#### Azure Database for PostgreSQL/MySQL - Over-provisioned vCores

```
resources
| where type has "microsoft.dbforpostgresql/flexibleservers" or type has "microsoft.dbformysql/flexibleservers"
| extend skuName = tostring(sku.name)
| extend skuTier = tostring(sku.tier)
| extend storageSizeGB = toint(properties.storage.storageSizeGB)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, type, skuName, skuTier, storageSizeGB, location, tags, Details
```

> **_Note:_** Cross-reference with Azure Monitor metrics (cpu_percent, memory_percent) to determine if the provisioned tier is appropriate.
