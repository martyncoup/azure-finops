# Azure Carbon Optimization - Queries

Here you can find queries to support carbon-aware workload placement, emissions tracking, and sustainability optimisation using Azure Resource Graph.

#### Resources in High Carbon Intensity Regions

```
resources
| extend regionLower = tolower(location)
| where regionLower in ("australiacentral", "australiacentral2", "australiaeast", "australiasoutheast", "centralindia", "southindia", "westindia", "eastasia", "southafricanorth", "southafricawest")
| summarize resourceCount=count(), types=make_set(type) by location, subscriptionId
| order by resourceCount desc
```

> **_Note:_** Regions powered primarily by fossil fuels have higher carbon intensity. Consider migrating workloads to regions with cleaner energy grids (e.g., Sweden Central, Norway East, France Central). Refer to [Azure sustainability regions](https://datacenters.microsoft.com/globe/fact-sheets) for current carbon intensity data.

#### Compute Resources Eligible for Region Migration (Carbon Reduction)

```
resources
| where type in ("microsoft.compute/virtualmachines", "microsoft.compute/virtualmachinescalesets", "microsoft.web/serverfarms", "microsoft.containerservice/managedclusters")
| extend regionLower = tolower(location)
| where regionLower !in ("swedencentral", "norwayeast", "francecentral", "canadacentral", "uksouth", "ukwest", "northeurope", "westeurope", "switzerlandnorth")
| extend vmSize = coalesce(
    tostring(properties.hardwareProfile.vmSize),
    tostring(properties.virtualMachineProfile.hardwareProfile.vmSize),
    tostring(sku.name)
)
| extend Details = pack_all()
| project Resource=id, type, resourceGroup, subscriptionId, location, vmSize, tags, Details
```

> **_Note:_** The "low carbon" region list is illustrative and should be updated based on current [Electricity Maps](https://app.electricitymaps.com) or Microsoft sustainability data. Workload migration feasibility depends on latency, compliance, and data residency requirements.

#### Virtual Machines Running 24/7 (Auto-Shutdown Candidates)

```
resources
| where type == "microsoft.compute/virtualmachines"
| where properties.extended.instanceView.powerState.code == "PowerState/running"
| join kind=leftouter (
    resources
    | where type == "microsoft.devtestlab/schedules"
    | where properties.taskType == "ComputeVmShutdownTask"
    | extend targetVmId = tostring(properties.targetResourceId)
    | project targetVmId, shutdownTime=properties.dailyRecurrence.time
) on $left.id == $right.targetVmId
| where isempty(targetVmId)
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, vmSize, tags, Details
```

> **_Note:_** Running VMs without auto-shutdown schedules consume energy continuously. Enabling schedules for non-production workloads reduces both cost and carbon emissions (typically 65%+ reduction for dev/test).

#### Over-provisioned Resources Contributing to Carbon Waste

```
resources
| where type == "microsoft.compute/virtualmachines"
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| where vmSize has_any ("Standard_E", "Standard_M", "Standard_L", "Standard_G")
| join kind=inner (
    resourcecontainers
    | where type == "microsoft.resources/subscriptions"
    | where tags["environment"] in ("dev", "test", "development", "sandbox")
      or name has "dev" or name has "test" or name has "sandbox"
    | project subscriptionId
) on subscriptionId
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, vmSize, tags, Details
```

> **_Note:_** Memory-optimised and high-performance VM SKUs in non-production environments consume disproportionate energy. Right-sizing these reduces both cost and carbon footprint.

#### Storage Redundancy Carbon Impact (Multi-Region Replication)

```
resources
| where type == "microsoft.storage/storageaccounts"
| where sku.name has "GRS" or sku.name has "GZRS" or sku.name has "RA"
| extend replication = tostring(sku.name)
| extend accessTier = tostring(properties.accessTier)
| summarize accountCount=count(), totalByReplication=count() by replication, location
| order by totalByReplication desc
```

> **_Note:_** Geo-redundant replication doubles storage carbon footprint by maintaining copies across regions. Evaluate whether LRS/ZRS is sufficient for workloads where geo-redundancy is not required.

#### Idle Resources with Carbon Cost (Summary)

```
resources
| where type in (
    "microsoft.compute/virtualmachines",
    "microsoft.network/azurefirewalls",
    "microsoft.network/applicationgateways",
    "microsoft.network/virtualnetworkgateways",
    "microsoft.apimanagement/service"
)
| extend powerState = tostring(properties.extended.instanceView.powerState.code)
| extend isIdle = case(
    type == "microsoft.compute/virtualmachines" and powerState == "PowerState/deallocated", true,
    type == "microsoft.network/azurefirewalls" and (properties.ipConfigurations == "[]" or isnull(properties.ipConfigurations)), true,
    type == "microsoft.network/applicationgateways" and properties.backendAddressPools == "[]", true,
    false
)
| where isIdle == true
| summarize idleCount=count() by type, location, subscriptionId
| order by idleCount desc
```

> **_Note:_** Idle resources consume rack space, cooling, and baseline energy. Decommissioning reduces Scope 2 and Scope 3 emissions.

#### ARM64-Eligible Workloads (Energy Efficient Compute)

```
resources
| where type == "microsoft.compute/virtualmachines"
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| where vmSize startswith "Standard_D" or vmSize startswith "Standard_E"
| where vmSize !has "p"
| extend osType = tostring(properties.storageProfile.osDisk.osType)
| where osType =~ "Linux"
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, vmSize, osType, tags, Details
```

> **_Note:_** Azure Arm-based VMs (Dpsv5, Epsv5 series) deliver up to 50% better price-performance and consume significantly less energy per vCPU. Linux workloads on D/E series are prime candidates for migration to Arm equivalents.

#### Carbon Emissions Data (Microsoft Cloud for Sustainability)

```
resources
| where type == "microsoft.carbon/carbonemissionreports"
| extend reportDate = tostring(properties.reportDate)
| extend totalEmissions = todouble(properties.totalCarbonEmissions)
| extend scope1 = todouble(properties.scope1Emissions)
| extend scope2 = todouble(properties.scope2Emissions)
| extend scope3 = todouble(properties.scope3Emissions)
| extend Details = pack_all()
| project Resource=id, subscriptionId, reportDate, totalEmissions, scope1, scope2, scope3, Details
| order by reportDate desc
```

> **_Note:_** Requires the Microsoft Cloud for Sustainability APIs / Carbon Optimization preview. This query retrieves emission reports if the carbon resource provider is registered on the subscription.

#### Resource Distribution by Region (Carbon Intensity Heatmap Data)

```
resources
| summarize resourceCount=count() by location, subscriptionId
| order by resourceCount desc
```

> **_Note:_** Use this as input to map resource density against regional carbon intensity data. High resource counts in high-intensity regions represent the greatest opportunity for carbon reduction through migration.
