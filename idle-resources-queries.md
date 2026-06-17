# Azure Idle & Underutilized Resources - Queries

Here you can find queries to identify resources that are idle or significantly underutilized, representing cost optimization opportunities.

#### Stopped (Deallocated) Virtual Machines

```
resources
| where type == "microsoft.compute/virtualmachines"
| where properties.extended.instanceView.powerState.code == "PowerState/deallocated"
| extend lastStatusChange = tostring(properties.extended.instanceView.powerState.displayStatus)
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, vmSize, lastStatusChange, tags, Details
```

> **_Note:_** Deallocated VMs still incur costs for associated disks, public IPs, and other attached resources.

#### Idle Azure SQL Elastic Pools

```
resources
| where type == "microsoft.sql/servers/elasticpools"
| extend dtu = toint(properties.dtu)
| extend currentDtu = tostring(sku.name)
| extend databaseCount = toint(properties.perDatabaseSettings)
| join kind=leftouter (
    resources
    | where type == "microsoft.sql/servers/databases"
    | where sku.name == "ElasticPool"
    | summarize dbCount=count() by elasticPoolId=tostring(properties.elasticPoolId)
) on $left.id == $right.elasticPoolId
| where isnull(dbCount) or dbCount == 0
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, currentDtu, dbCount, tags, Details
```

> **_Note:_** Elastic pools with no databases assigned are generating cost with no workload benefit.

#### Idle Application Gateways (No Backend Targets)

```
resources
| where type == "microsoft.network/applicationgateways"
| mv-expand pool = properties.backendAddressPools
| extend backendCount = array_length(pool.properties.backendAddresses)
| summarize totalBackends=sum(backendCount) by id, resourceGroup, subscriptionId, location, tostring(tags)
| where totalBackends == 0
| project Resource=id, resourceGroup, subscriptionId, location, totalBackends, tags
```

#### Idle Azure Firewall

```
resources
| where type == "microsoft.network/azurefirewalls"
| where properties.ipConfigurations == "[]" or isnull(properties.ipConfigurations)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sku=properties.sku.name, tags, Details
```

> **_Note:_** Azure Firewalls without IP configurations are not routing traffic but still incur hourly charges.

#### Idle ExpressRoute Circuits

```
resources
| where type == "microsoft.network/expressroutecircuits"
| where properties.circuitProvisioningState == "Enabled"
| where properties.serviceProviderProvisioningState == "NotProvisioned"
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sku=sku.name, tier=sku.tier, tags, Details
```

> **_Note:_** ExpressRoute circuits in NotProvisioned state are billable but not carrying traffic.

#### Idle Virtual Network Gateways (VPN/ExpressRoute)

```
resources
| where type == "microsoft.network/virtualnetworkgateways"
| where properties.gatewayType == "Vpn"
| join kind=leftouter (
    resources
    | where type == "microsoft.network/connections"
    | where properties.connectionStatus == "Connected"
    | mv-expand gw = properties.virtualNetworkGateway1
    | extend gwId = tostring(gw.id)
    | summarize activeConnections=count() by gwId
) on $left.id == $right.gwId
| where isnull(activeConnections) or activeConnections == 0
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, properties.gatewayType, sku=properties.sku.name, tags, Details
```

#### Idle API Management Instances

```
resources
| where type == "microsoft.apimanagement/service"
| extend skuName = tostring(sku.name)
| where skuName != "Consumption"
| join kind=leftouter (
    resources
    | where type == "microsoft.apimanagement/service/apis"
    | extend serviceId = tostring(properties.serviceUrl)
    | summarize apiCount=count() by serviceId=tostring(split(id, "/apis/")[0])
) on $left.id == $right.serviceId
| where isnull(apiCount) or apiCount == 0
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, skuName, apiCount, tags, Details
```

> **_Note:_** Non-consumption tier APIM instances with no APIs are generating significant idle cost.

#### Idle Log Analytics Workspaces

```
resources
| where type == "microsoft.operationalinsights/workspaces"
| extend sku = tostring(properties.sku.name)
| extend retentionDays = toint(properties.retentionInDays)
| extend dailyCapGB = tostring(properties.workspaceCapping.dailyQuotaGb)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sku, retentionDays, dailyCapGB, tags, Details
```

> **_Note:_** Cross-reference with ingestion metrics to identify workspaces with minimal or no data ingestion that still retain costly configurations.

#### Idle Redis Cache

```
resources
| where type == "microsoft.cache/redis"
| extend skuName = tostring(properties.sku.name)
| extend skuFamily = tostring(properties.sku.family)
| extend skuCapacity = toint(properties.sku.capacity)
| extend connectedClients = toint(properties.accessKeys)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, skuName, skuFamily, skuCapacity, tags, Details
```

> **_Note:_** Cross-reference with Azure Monitor connected_clients metric. Redis caches with consistently zero connected clients are idle.
