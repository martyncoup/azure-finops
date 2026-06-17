# Azure FinOps - Resource Graph Queries

A collection of Azure Resource Graph queries designed to support FinOps scenarios, cost optimisation, and resource governance.

## Query Categories

| File | Description |
|------|-------------|
| [orphaned-queries.md](orphaned-queries.md) | Identify orphaned resources no longer attached to active workloads |
| [idle-resources-queries.md](idle-resources-queries.md) | Find idle or underutilized resources generating unnecessary cost |
| [right-sizing-queries.md](right-sizing-queries.md) | Detect over-provisioned resources that are candidates for downsizing |
| [compute-optimization-queries.md](compute-optimization-queries.md) | Compute-specific savings including Hybrid Benefit, Spot, and SKU selection |
| [storage-optimization-queries.md](storage-optimization-queries.md) | Storage-related optimisations including tiering, redundancy, and lifecycle |
| [carbon-optimization-queries.md](carbon-optimization-queries.md) | Carbon-aware placement, emissions tracking, energy-efficient compute, and sustainability |
| [container-optimization-queries.md](container-optimization-queries.md) | AKS, Container Instances, Container Registry, and Container Apps cost optimisation |

## Usage

These queries are designed for use with:
- **Azure Resource Graph Explorer** in the Azure Portal
- **Azure Workbooks** for dashboarding and reporting
- **Azure Resource Graph SDK** for programmatic access
- **Azure CLI** via `az graph query`

### Example - Azure CLI

```bash
az graph query -q "resources | where type == 'microsoft.compute/disks' | where properties.diskState == 'Unattached'" --subscriptions <subscription-id>
```

## Prerequisites

- Some queries require **Azure Monitor agent** and **VM Insights** for metrics-based analysis
- Resource Graph queries operate on resource metadata; cross-reference with Azure Monitor metrics for utilisation data
- Appropriate RBAC permissions (Reader role minimum) on target subscriptions

## Contributing

When adding new queries, follow the existing format:
1. Use `####` heading for the resource type
2. Wrap the query in a fenced code block
3. Add a `> **_Note:_**` block for caveats or prerequisites
4. Ensure the query projects `subscriptionId`, resource identifier, `location`, `tags`, and a `Details` column
