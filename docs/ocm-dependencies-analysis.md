# OCM/RHACM Dependencies Analysis

This document provides a comprehensive analysis of Ramen's dependencies on Open Cluster Management (OCM) / Red Hat Advanced Cluster Management (RHACM), focusing on orchestration-related components (excluding UI).

## Executive Summary

Ramen is fundamentally built as an OCM placement extension. It relies heavily on OCM APIs for:
- **Cluster Management**: Discovering and managing clusters via `ManagedCluster`
- **Placement**: Using `Placement` and `PlacementDecision` for cluster selection
- **Resource Deployment**: Using `ManifestWork` to deploy resources to managed clusters
- **Resource Reading**: Using `ManagedClusterView` to read resources from managed clusters
- **Legacy Support**: Supporting `PlacementRule` for backward compatibility

## Core OCM Dependencies

### 1. Go Module Dependencies

From `go.mod`:
```go
open-cluster-management.io/api v0.15.0
open-cluster-management.io/config-policy-controller v0.15.0
open-cluster-management.io/governance-policy-propagator v0.16.0
open-cluster-management.io/multicloud-operators-subscription v0.15.0
```

### 2. OCM API Groups Used

#### 2.1 Cluster Management (`cluster.open-cluster-management.io`)

**Resources:**
- `ManagedCluster` - Represents a managed cluster
  - Used to: Get cluster information, check cluster status, read cluster claims
  - Location: `internal/controller/util/managedcluster.go`
  - Key usage: Cluster ID extraction, storage class claims

- `Placement` - Modern placement API for cluster selection
  - Used to: Select target clusters for workload placement
  - Location: `internal/controller/drplacementcontrol_controller.go`
  - Key usage: DRPlacementControl references Placement to determine target cluster

- `PlacementDecision` - Results of placement decisions
  - Used to: Store and retrieve cluster selection decisions
  - Location: `internal/controller/drplacementcontrol_controller.go`
  - Key usage: Ramen creates/updates PlacementDecision to control placement

#### 2.2 Work API (`work.open-cluster-management.io`)

**Resources:**
- `ManifestWork` - Deploy resources to managed clusters
  - Used to: Deploy VRG, MaintenanceMode, NetworkFence, DRClusterConfig, Namespaces, RBAC to managed clusters
  - Location: `internal/controller/util/mw_util.go`
  - Key usage: Primary mechanism for deploying Ramen resources to managed clusters

#### 2.3 View API (`view.open-cluster-management.io`)

**Resources:**
- `ManagedClusterView` - Read resources from managed clusters
  - Used to: Read VRG, MaintenanceMode, NetworkFence, StorageClass, VolumeSnapshotClass, etc. from managed clusters
  - Location: `internal/controller/util/mcv_util.go`
  - Key usage: Hub operator reads managed cluster state without direct API access

#### 2.4 Legacy Placement (`apps.open-cluster-management.io`)

**Resources:**
- `PlacementRule` - Legacy placement mechanism
  - Used to: Support older placement configurations
  - Location: `internal/controller/drplacementcontrol_controller.go`
  - Key usage: Backward compatibility with older OCM deployments

#### 2.5 Policy API (`policy.open-cluster-management.io`)

**Resources:**
- Used for: Policy propagation (limited usage)
  - Location: `internal/controller/volsync/secret_propagator.go`
  - Key usage: Propagating secrets via policies

#### 2.6 Addon API (`addon.open-cluster-management.io`)

**Resources:**
- `ManagedClusterAddon` - Manage addons on clusters
  - Used to: Check/manage addons (limited usage)

## Detailed Dependency Mapping

### Hub Operator Dependencies

The **Ramen Hub Operator** (`ramen-hub-operator`) runs on the OCM hub cluster and has the following OCM dependencies:

#### 1. DRPlacementControl Controller

**OCM Dependencies:**
- **Placement/PlacementRule**: Reads user's Placement/PlacementRule to determine target cluster
- **PlacementDecision**: Creates/updates PlacementDecision to control cluster selection
- **ManagedCluster**: Reads cluster information and claims
- **ManifestWork**: Deploys VRG, MaintenanceMode, NetworkFence, Namespaces to managed clusters
- **ManagedClusterView**: Reads VRG status from managed clusters

**Key Files:**
- `internal/controller/drplacementcontrol_controller.go`
- `internal/controller/drplacementcontrol.go`
- `internal/controller/util/mw_util.go`
- `internal/controller/util/mcv_util.go`

#### 2. DRPolicy Controller

**OCM Dependencies:**
- **ManagedCluster**: Validates clusters referenced in DRPolicy
- **PlacementRule**: Creates PlacementRules for secret propagation (legacy)

**Key Files:**
- `internal/controller/drpolicy_controller.go`

#### 3. DRCluster Controller

**OCM Dependencies:**
- **ManifestWork**: Deploys DRClusterConfig, RBAC resources to managed clusters
- **ManagedCluster**: Validates cluster existence and status

**Key Files:**
- `internal/controller/drcluster_controller.go`
- `internal/controller/drclusters.go`

### DR Cluster Operator Dependencies

The **Ramen DR Cluster Operator** (`ramen-dr-cluster-operator`) runs on managed clusters and has **minimal** OCM dependencies:

- **No direct OCM API usage** - The DR cluster operator primarily manages VolumeReplicationGroup resources locally
- Resources are deployed to it via ManifestWork from the hub

## Resource Flow

### Deployment Flow (Hub → Managed Cluster)

```
DRPlacementControl (Hub)
  ↓
Placement/PlacementDecision (OCM) → Selects target cluster
  ↓
ManifestWork (OCM) → Deploys VRG to managed cluster
  ↓
VRG (Managed Cluster) → Managed by DR Cluster Operator
```

### Status Reading Flow (Managed Cluster → Hub)

```
VRG (Managed Cluster)
  ↓
ManagedClusterView (OCM) → Reads VRG status
  ↓
DRPlacementControl (Hub) → Updates status based on MCV results
```

## Critical OCM Dependencies

### Must-Have (Cannot be easily replaced)

1. **ManifestWork** - Core mechanism for deploying resources to managed clusters
   - **Impact**: Without this, Ramen cannot deploy VRG, MaintenanceMode, or other resources to managed clusters
   - **Alternative**: Would require direct API access to managed clusters (not scalable)

2. **ManagedClusterView** - Core mechanism for reading resources from managed clusters
   - **Impact**: Without this, Ramen cannot read VRG status or other resources from managed clusters
   - **Alternative**: Would require direct API access to managed clusters (not scalable)

3. **Placement/PlacementDecision** - Core mechanism for cluster selection
   - **Impact**: Without this, Ramen cannot determine which cluster to place workloads on
   - **Alternative**: Could be replaced with custom cluster selection logic, but would lose OCM integration

4. **ManagedCluster** - Core mechanism for cluster discovery
   - **Impact**: Without this, Ramen cannot discover or validate managed clusters
   - **Alternative**: Could be replaced with custom cluster registry

### Nice-to-Have (Could be replaced)

1. **PlacementRule** - Legacy placement support
   - **Impact**: Loss of backward compatibility with older OCM deployments
   - **Alternative**: Migration to Placement API

2. **Policy APIs** - Used for secret propagation
   - **Impact**: Loss of policy-based secret propagation
   - **Alternative**: Direct secret creation via ManifestWork

## Isolation Strategy

### What Could Be Isolated

1. **Placement Logic**: The core DR orchestration logic (failover, relocation) could potentially work with a different cluster selection mechanism
2. **Resource Deployment**: Could use direct API access instead of ManifestWork (but loses scalability)
3. **Status Reading**: Could use direct API access instead of ManagedClusterView (but loses scalability)

### What Cannot Be Isolated

1. **OCM Hub Architecture**: Ramen is fundamentally designed as an OCM hub operator
2. **Multi-Cluster Management**: The entire architecture assumes OCM-managed clusters
3. **ManifestWork Pattern**: This is the standard OCM pattern for multi-cluster resource deployment

## Recommendations

### For OCM-Independent Operation

To make Ramen work without OCM, you would need to:

1. **Replace ManifestWork**: Implement direct Kubernetes API client access to managed clusters
   - Pros: No OCM dependency
   - Cons: Requires managing multiple kubeconfigs, less scalable, loses OCM benefits

2. **Replace ManagedClusterView**: Implement direct Kubernetes API client access to read resources
   - Pros: No OCM dependency
   - Cons: Requires managing multiple kubeconfigs, less scalable

3. **Replace Placement/PlacementDecision**: Implement custom cluster selection logic
   - Pros: No OCM dependency
   - Cons: Lose OCM placement features (predicates, prioritizers)

4. **Replace ManagedCluster**: Implement custom cluster registry
   - Pros: No OCM dependency
   - Cons: Need to maintain cluster metadata separately

### For Minimal OCM Dependency

To minimize OCM dependencies while keeping benefits:

1. **Keep ManifestWork**: Essential for scalable multi-cluster deployment
2. **Keep ManagedClusterView**: Essential for scalable multi-cluster status reading
3. **Simplify Placement**: Use basic Placement API only, avoid advanced features
4. **Remove Legacy**: Remove PlacementRule support
5. **Remove Policy Dependencies**: Replace policy-based secret propagation with direct ManifestWork

## Code Locations

### Core OCM Integration Points

1. **ManifestWork Operations**: `internal/controller/util/mw_util.go`
2. **ManagedClusterView Operations**: `internal/controller/util/mcv_util.go`
3. **ManagedCluster Operations**: `internal/controller/util/managedcluster.go`
4. **Placement Operations**: `internal/controller/drplacementcontrol_controller.go`
5. **PlacementRule Operations**: `internal/controller/drplacementcontrol_controller.go`

### RBAC Requirements

See `config/rbac/role.yaml` and `config/hub/rbac/role.yaml` for complete OCM API permissions required.

## Conclusion

Ramen is **deeply integrated** with OCM/RHACM. While some components could theoretically be replaced, the core architecture assumes OCM's multi-cluster management capabilities. The most critical dependencies are:

1. **ManifestWork** - For resource deployment
2. **ManagedClusterView** - For status reading
3. **Placement/PlacementDecision** - For cluster selection
4. **ManagedCluster** - For cluster discovery

Removing these would require significant architectural changes and would result in a fundamentally different product.
