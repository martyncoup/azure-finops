# Azure Container Optimization - Queries

Here you can find queries to identify cost optimisation opportunities across AKS, Container Instances, Container Registries, and Container Apps.

## AKS Cluster Cost Management

#### AKS Clusters Without Autoscaler Enabled

```
resources
| where type == "microsoft.containerservice/managedclusters"
| mv-expand pool = properties.agentPoolProfiles
| extend poolName = tostring(pool.name)
| extend enableAutoScaling = tobool(pool.enableAutoScaling)
| extend vmSize = tostring(pool.vmSize)
| extend nodeCount = toint(pool.['count'])
| where enableAutoScaling == false
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, poolName, vmSize, nodeCount, tags, Details
```

> **_Note:_** Node pools without autoscaler pay for fixed capacity regardless of workload demand. Enable the cluster autoscaler to scale down during low-usage periods.

#### AKS Clusters with Over-provisioned System Node Pools

```
resources
| where type == "microsoft.containerservice/managedclusters"
| mv-expand pool = properties.agentPoolProfiles
| where tostring(pool.mode) == "System"
| extend poolName = tostring(pool.name)
| extend vmSize = tostring(pool.vmSize)
| extend nodeCount = toint(pool.['count'])
| where vmSize has_any ("Standard_D8", "Standard_D16", "Standard_D32", "Standard_D48", "Standard_D64", "Standard_E", "Standard_M")
  or nodeCount > 3
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, poolName, vmSize, nodeCount, tags, Details
```

> **_Note:_** System node pools run kube-system components only. They rarely need more than 3 nodes or large SKUs. Consider Standard_D2s_v5 or Standard_D4s_v5 for system pools.

#### AKS Clusters Using Premium SKUs Without GPU Workloads

```
resources
| where type == "microsoft.containerservice/managedclusters"
| mv-expand pool = properties.agentPoolProfiles
| extend vmSize = tostring(pool.vmSize)
| where vmSize has_any ("Standard_NC", "Standard_ND", "Standard_NV")
| extend poolName = tostring(pool.name)
| extend nodeCount = toint(pool.['count'])
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, poolName, vmSize, nodeCount, tags, Details
```

> **_Note:_** Cross-reference with GPU utilisation metrics in Container Insights. GPU node pools with consistently low GPU usage should be resized or removed.

#### AKS Clusters Without Virtual Nodes (ACI Burst)

```
resources
| where type == "microsoft.containerservice/managedclusters"
| where properties.addonProfiles.aciConnectorLinux.enabled != true
| extend nodePoolCount = array_length(properties.agentPoolProfiles)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, nodePoolCount, tags, Details
```

> **_Note:_** Virtual nodes enable serverless burst to ACI for spiky workloads without maintaining idle capacity. Particularly beneficial for batch and event-driven workloads.

#### AKS Clusters in Same Region (Consolidation Candidates)

```
resources
| where type == "microsoft.containerservice/managedclusters"
| summarize clusterCount=count(), clusters=make_list(id) by location, subscriptionId
| where clusterCount > 1
| project subscriptionId, location, clusterCount, clusters
```

> **_Note:_** Multiple AKS clusters in the same region and subscription may be candidates for consolidation using namespaces, reducing control plane and per-node overhead costs.

#### AKS Clusters Without Spot Node Pools

```
resources
| where type == "microsoft.containerservice/managedclusters"
| where properties.agentPoolProfiles !has "Spot"
| mv-expand pool = properties.agentPoolProfiles
| where tostring(pool.mode) == "User"
| extend poolName = tostring(pool.name)
| extend vmSize = tostring(pool.vmSize)
| extend nodeCount = toint(pool.['count'])
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, poolName, vmSize, nodeCount, tags, Details
```

> **_Note:_** AKS user node pools running interruptible workloads (batch processing, CI/CD agents, dev/test) can save up to 90% using Spot instances.

## Container Instances (ACI)

#### Long-Running Container Groups (AKS Crossover Candidates)

```
resources
| where type == "microsoft.containerinstances/containergroups"
| where properties.restartPolicy == "Always"
| extend containerCount = array_length(properties.containers)
| extend os = tostring(properties.osType)
| extend cpu = todouble(properties.containers[0].properties.resources.requests.cpu)
| extend memoryGB = todouble(properties.containers[0].properties.resources.requests.memoryInGB)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, os, containerCount, cpu, memoryGB, tags, Details
```

> **_Note:_** Container groups with `restartPolicy: Always` are long-running services. At approximately 3+ vCPU continuously, AKS becomes more cost-effective than ACI.

#### ACI Groups with GPU Allocated

```
resources
| where type == "microsoft.containerinstances/containergroups"
| mv-expand container = properties.containers
| where isnotempty(container.properties.resources.requests.gpu)
| extend gpuCount = toint(container.properties.resources.requests.gpu.count)
| extend gpuSku = tostring(container.properties.resources.requests.gpu.sku)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, gpuSku, gpuCount, tags, Details
```

> **_Note:_** GPU container instances are expensive. Verify these are actively processing workloads and not sitting idle.

#### Idle Container Groups (Running with No Ingress)

```
resources
| where type == "microsoft.containerinstances/containergroups"
| where properties.instanceView.state == "Running"
| where isnull(properties.ipAddress) or properties.ipAddress.ports == "[]"
| extend containerCount = array_length(properties.containers)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, containerCount, tags, Details
```

## Container Registry (ACR)

#### Premium ACRs Potentially Over-tiered

```
resources
| where type == "microsoft.containerregistry/registries"
| where sku.name == "Premium"
| extend geoReplications = tobool(properties.networkRuleSet)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sku=sku.name, tags, Details
```

> **_Note:_** Premium ACR is required for geo-replication, private endpoints, and content trust. If none of these features are used, Standard tier provides equivalent functionality at lower cost.

#### ACR Without Retention Policies (Unbounded Storage Growth)

```
resources
| where type == "microsoft.containerregistry/registries"
| where isnull(properties.policies.retentionPolicy) or properties.policies.retentionPolicy.status != "enabled"
| extend sku = tostring(sku.name)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sku, tags, Details
```

> **_Note:_** Without retention policies, untagged manifests accumulate indefinitely. Enable retention policies or run `az acr purge` regularly to control storage costs.

#### Geo-Replicated ACRs in Non-Production

```
resources
| where type == "microsoft.containerregistry/registries"
| where array_length(properties.replicationLocations) > 1 or sku.name == "Premium"
| join kind=inner (
    resourcecontainers
    | where type == "microsoft.resources/subscriptions"
    | where tags["environment"] in ("dev", "test", "development", "sandbox")
      or name has "dev" or name has "test" or name has "sandbox"
    | project subscriptionId
) on subscriptionId
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sku=sku.name, tags, Details
```

> **_Note:_** Geo-replication in dev/test is rarely justified. Each replica incurs Premium tier costs per region.

## Container Apps

#### Container Apps Environments with No Active Apps

```
resources
| where type == "microsoft.app/managedenvironments"
| join kind=leftouter (
    resources
    | where type == "microsoft.app/containerapps"
    | extend envId = tostring(properties.managedEnvironmentId)
    | summarize appCount=count() by envId
) on $left.id == $right.envId
| where isnull(appCount) or appCount == 0
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, appCount, tags, Details
```

> **_Note:_** Empty Container Apps environments still incur charges for the underlying infrastructure (particularly with dedicated workload profiles).

#### Container Apps with Min Replicas > 0 (Scale-to-Zero Candidates)

```
resources
| where type == "microsoft.app/containerapps"
| mv-expand rule = properties.template.scale.rules
| extend minReplicas = toint(properties.template.scale.minReplicas)
| where minReplicas > 0
| extend maxReplicas = toint(properties.template.scale.maxReplicas)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, minReplicas, maxReplicas, tags, Details
```

> **_Note:_** Container Apps with `minReplicas > 0` never scale to zero. For event-driven or low-traffic workloads, setting minReplicas to 0 eliminates idle compute costs.

#### Container Apps Not Using Consumption Workload Profile

```
resources
| where type == "microsoft.app/containerapps"
| extend environmentId = tostring(properties.managedEnvironmentId)
| join kind=inner (
    resources
    | where type == "microsoft.app/managedenvironments"
    | where properties.workloadProfiles !has "Consumption"
    | project envId=id
) on $left.environmentId == $right.envId
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, environmentId, tags, Details
```

> **_Note:_** Dedicated workload profiles provide reserved compute. If apps don't need guaranteed capacity or specific hardware, Consumption profile offers pay-per-use with scale-to-zero.

## AKS Networking Costs

#### AKS Clusters with Azure CNI and Large Subnet Allocation

```
resources
| where type == "microsoft.containerservice/managedclusters"
| where properties.networkProfile.networkPlugin == "azure"
| extend subnetId = tostring(properties.agentPoolProfiles[0].vnetSubnetID)
| extend maxPods = toint(properties.agentPoolProfiles[0].maxPods)
| extend nodeCount = toint(properties.agentPoolProfiles[0].['count'])
| extend estimatedIPs = nodeCount * maxPods
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, networkPlugin="Azure CNI", nodeCount, maxPods, estimatedIPs, subnetId, tags, Details
```

> **_Note:_** Azure CNI pre-allocates IPs for all potential pods. Large maxPods values on large clusters reserve excessive subnet space and may force larger VNETs. Consider Azure CNI Overlay or kubenet for IP-efficient networking.

## AKS Storage Costs

#### AKS Persistent Volumes Using Premium Storage

```
resources
| where type == "microsoft.compute/disks"
| where sku.name has "Premium"
| where properties.diskState == "Attached"
| where tostring(managedBy) has "aks"
| extend diskSizeGB = toint(properties.diskSizeGB)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, sku=sku.name, diskSizeGB, managedBy, tags, Details
| order by diskSizeGB desc
```

> **_Note:_** Not all AKS persistent workloads require Premium SSD performance. Evaluate if StandardSSD_LRS is sufficient for non-latency-sensitive persistent volumes.

#### Orphaned Disks from Deleted AKS Pods

```
resources
| where type == "microsoft.compute/disks"
| where properties.diskState == "Unattached"
| where tags has "kubernetes.io" or name startswith "kubernetes-dynamic-pvc-"
| extend diskSizeGB = toint(properties.diskSizeGB)
| extend skuName = tostring(sku.name)
| extend Details = pack_all()
| project Resource=id, resourceGroup, subscriptionId, location, skuName, diskSizeGB, tags, Details
| order by diskSizeGB desc
```

> **_Note:_** When AKS pods or namespaces are deleted with `reclaimPolicy: Retain`, their PersistentVolume disks remain. These orphaned Kubernetes disks should be reviewed and deleted if no longer needed.
