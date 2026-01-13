# OCM/RHACM Dependency Isolation Strategy

This document provides a detailed analysis of how to isolate or minimize OCM/RHACM dependencies in Ramen.

## Dependency Categories

### Category 1: Critical Dependencies (Cannot Remove Without Major Rewrite)

These dependencies are fundamental to Ramen's architecture:

#### 1.1 ManifestWork (`work.open-cluster-management.io`)

**Current Usage:**
- Deploys VRG, MaintenanceMode, NetworkFence, Namespaces, DRClusterConfig, RBAC to managed clusters
- Primary mechanism for multi-cluster resource deployment

**Isolation Strategy:**
- **Option A**: Replace with direct Kubernetes API client access
  - Pros: No OCM dependency
  - Cons: 
    - Requires managing multiple kubeconfigs
    - Less scalable (no built-in retry, status tracking)
    - Lose OCM's declarative resource management
    - Need to implement own status tracking
  - Effort: High (rewrite all deployment logic)

- **Option B**: Create abstraction layer
  - Pros: Can swap implementations
  - Cons: Still need to implement alternative
  - Effort: Medium (create interface, keep OCM implementation)

**Recommendation**: Keep ManifestWork - it's the standard OCM pattern and provides significant value.

#### 1.2 ManagedClusterView (`view.open-cluster-management.io`)

**Current Usage:**
- Reads VRG, MaintenanceMode, NetworkFence, StorageClass, VolumeSnapshotClass, VolumeReplicationClass, DRClusterConfig from managed clusters
- Primary mechanism for reading managed cluster state

**Isolation Strategy:**
- **Option A**: Replace with direct Kubernetes API client access
  - Pros: No OCM dependency
  - Cons:
    - Requires managing multiple kubeconfigs
    - Less scalable
    - Need to handle cluster connectivity issues manually
  - Effort: High (rewrite all status reading logic)

- **Option B**: Create abstraction layer
  - Pros: Can swap implementations
  - Cons: Still need to implement alternative
  - Effort: Medium (create interface, keep OCM implementation)

**Recommendation**: Keep ManagedClusterView - it provides a clean abstraction for reading remote cluster state.

#### 1.3 Placement API (`cluster.open-cluster-management.io/v1beta1`)

**Current Usage:**
- Determines which cluster to place workloads on
- DRPlacementControl references Placement to get target cluster
- Ramen creates/updates PlacementDecision to control placement

**Isolation Strategy:**
- **Option A**: Replace with custom cluster selection logic
  - Pros: No OCM dependency
  - Cons:
    - Lose OCM placement features (predicates, prioritizers, cluster sets)
    - Need to implement own cluster selection
    - Lose integration with OCM ecosystem
  - Effort: Medium (implement custom selection logic)

- **Option B**: Use Placement but abstract the interface
  - Pros: Can swap implementations later
  - Cons: Still depends on Placement API
  - Effort: Low (create interface wrapper)

**Recommendation**: Keep Placement API but abstract it - allows future flexibility.

#### 1.4 ManagedCluster (`cluster.open-cluster-management.io/v1`)

**Current Usage:**
- Discovers and validates managed clusters
- Reads cluster claims (cluster ID, storage classes, etc.)
- Checks cluster status (joined, available)

**Isolation Strategy:**
- **Option A**: Replace with custom cluster registry
  - Pros: No OCM dependency
  - Cons:
    - Need to maintain cluster metadata separately
    - Lose OCM cluster management features
    - Need to implement own cluster discovery
  - Effort: Medium (implement custom registry)

- **Option B**: Use ManagedCluster but abstract the interface
  - Pros: Can swap implementations later
  - Cons: Still depends on ManagedCluster API
  - Effort: Low (create interface wrapper)

**Recommendation**: Keep ManagedCluster but abstract it - it's a standard way to represent clusters.

### Category 2: Replaceable Dependencies (Can Remove)

These dependencies can be removed or replaced with simpler alternatives:

#### 2.1 PlacementRule (`apps.open-cluster-management.io`)

**Current Usage:**
- Legacy placement support
- Used for backward compatibility with older OCM deployments
- Also used for secret propagation via policies

**Isolation Strategy:**
- **Remove**: Migrate all usage to Placement API
  - Pros: Remove legacy dependency
  - Cons: Lose backward compatibility
  - Effort: Low (update code to use Placement only)

**Recommendation**: Remove PlacementRule support - it's legacy and Placement API is the future.

#### 2.2 Policy APIs (`policy.open-cluster-management.io`)

**Current Usage:**
- Secret propagation via policies
- Used in VolSync secret propagation

**Isolation Strategy:**
- **Replace**: Use ManifestWork to deploy secrets directly
  - Pros: Remove policy dependency
  - Cons: Need to manage secrets via ManifestWork
  - Effort: Low (replace policy-based secret propagation)

**Recommendation**: Remove Policy API dependency - use ManifestWork directly.

## Isolation Implementation Plan

### Phase 1: Create Abstraction Layers (Low Risk)

**Goal**: Abstract OCM dependencies behind interfaces without changing behavior.

**Steps:**
1. Create `ClusterManager` interface to abstract ManagedCluster
2. Create `PlacementManager` interface to abstract Placement/PlacementDecision
3. Create `ResourceDeployer` interface to abstract ManifestWork
4. Create `ResourceReader` interface to abstract ManagedClusterView
5. Implement OCM-based implementations of these interfaces
6. Update controllers to use interfaces instead of direct OCM APIs

**Files to Modify:**
- `internal/controller/util/managedcluster.go` → Add interface
- `internal/controller/util/mw_util.go` → Add interface
- `internal/controller/util/mcv_util.go` → Add interface
- `internal/controller/drplacementcontrol_controller.go` → Use interfaces

**Benefits:**
- No functional changes
- Enables future alternative implementations
- Makes dependencies explicit

### Phase 2: Remove Legacy Dependencies (Medium Risk)

**Goal**: Remove PlacementRule and Policy API dependencies.

**Steps:**
1. Migrate all PlacementRule usage to Placement API
2. Replace policy-based secret propagation with ManifestWork
3. Remove PlacementRule and Policy API imports
4. Update RBAC to remove unused permissions

**Files to Modify:**
- `internal/controller/drplacementcontrol_controller.go` → Remove PlacementRule support
- `internal/controller/volsync/secret_propagator.go` → Use ManifestWork instead of policies
- `config/rbac/role.yaml` → Remove unused permissions

**Benefits:**
- Reduced dependencies
- Simpler codebase
- Focus on modern OCM APIs

### Phase 3: Alternative Implementations (High Risk, Optional)

**Goal**: Implement non-OCM alternatives for critical dependencies.

**Steps:**
1. Implement direct K8s API client for ResourceDeployer
2. Implement direct K8s API client for ResourceReader
3. Implement custom cluster selection for PlacementManager
4. Implement custom cluster registry for ClusterManager
5. Add feature flag to switch between OCM and direct implementations

**Files to Create:**
- `internal/controller/util/direct_deployer.go` → Direct K8s API deployment
- `internal/controller/util/direct_reader.go` → Direct K8s API reading
- `internal/controller/util/custom_placement.go` → Custom placement logic
- `internal/controller/util/custom_cluster_registry.go` → Custom cluster registry

**Benefits:**
- Can run without OCM
- More flexibility

**Drawbacks:**
- Lose OCM benefits (scalability, declarative management)
- More complex codebase
- Need to maintain multiple implementations

## Code Changes Required

### Abstraction Layer Example

```go
// internal/controller/util/interfaces.go

// ClusterManager abstracts cluster discovery and management
type ClusterManager interface {
    GetCluster(ctx context.Context, name string) (*ClusterInfo, error)
    ListClusters(ctx context.Context) ([]ClusterInfo, error)
    GetClusterID(ctx context.Context, name string) (string, error)
    GetClusterClaims(ctx context.Context, name string) (map[string]string, error)
}

// PlacementManager abstracts cluster selection
type PlacementManager interface {
    GetPlacement(ctx context.Context, name, namespace string) (*PlacementInfo, error)
    GetPlacementDecision(ctx context.Context, placementName, namespace string) (*PlacementDecisionInfo, error)
    CreateOrUpdatePlacementDecision(ctx context.Context, placementName, namespace string, clusterName string) error
}

// ResourceDeployer abstracts resource deployment to managed clusters
type ResourceDeployer interface {
    DeployVRG(ctx context.Context, cluster string, vrg *rmn.VolumeReplicationGroup) error
    DeployMaintenanceMode(ctx context.Context, cluster string, mm *rmn.MaintenanceMode) error
    DeployNetworkFence(ctx context.Context, cluster string, nf *csiaddonsv1alpha1.NetworkFence) error
    DeployNamespace(ctx context.Context, cluster string, ns *corev1.Namespace) error
    DeleteResource(ctx context.Context, cluster, resourceType, name, namespace string) error
}

// ResourceReader abstracts reading resources from managed clusters
type ResourceReader interface {
    ReadVRG(ctx context.Context, cluster, name, namespace string) (*rmn.VolumeReplicationGroup, error)
    ReadMaintenanceMode(ctx context.Context, cluster, name string) (*rmn.MaintenanceMode, error)
    ReadNetworkFence(ctx context.Context, cluster, name, namespace string) (*csiaddonsv1alpha1.NetworkFence, error)
    ReadStorageClass(ctx context.Context, cluster, name string) (*storagev1.StorageClass, error)
}
```

### OCM Implementation Example

```go
// internal/controller/util/ocm_implementations.go

type OCMClusterManager struct {
    client client.Client
}

func (o *OCMClusterManager) GetCluster(ctx context.Context, name string) (*ClusterInfo, error) {
    mc := &ocmv1.ManagedCluster{}
    if err := o.client.Get(ctx, types.NamespacedName{Name: name}, mc); err != nil {
        return nil, err
    }
    // Convert to ClusterInfo
    return convertManagedClusterToClusterInfo(mc), nil
}

// Similar implementations for other interfaces...
```

### Direct K8s Implementation Example

```go
// internal/controller/util/direct_implementations.go

type DirectResourceDeployer struct {
    clusterClients map[string]client.Client // Map of cluster name to kubeconfig client
}

func (d *DirectResourceDeployer) DeployVRG(ctx context.Context, cluster string, vrg *rmn.VolumeReplicationGroup) error {
    client, ok := d.clusterClients[cluster]
    if !ok {
        return fmt.Errorf("no client for cluster %s", cluster)
    }
    return client.Create(ctx, vrg)
}

// Similar implementations for other methods...
```

## Migration Path

### Step 1: Add Interfaces (Week 1-2)
- Create interface definitions
- No functional changes
- Low risk

### Step 2: Implement OCM Wrappers (Week 2-3)
- Implement OCM-based implementations of interfaces
- Update controllers to use interfaces
- Test thoroughly
- Medium risk

### Step 3: Remove Legacy (Week 3-4)
- Remove PlacementRule support
- Remove Policy API usage
- Update tests
- Medium risk

### Step 4: Optional - Add Alternatives (Week 4+)
- Implement direct K8s API alternatives
- Add feature flags
- Test both implementations
- High risk, optional

## Testing Strategy

1. **Unit Tests**: Test interfaces with mock implementations
2. **Integration Tests**: Test OCM implementations with real OCM cluster
3. **E2E Tests**: Test full flow with both OCM and direct implementations (if implemented)

## Conclusion

**Recommended Approach:**
1. ✅ Create abstraction layers (Phase 1) - Low risk, high value
2. ✅ Remove legacy dependencies (Phase 2) - Medium risk, medium value
3. ⚠️ Skip alternative implementations (Phase 3) - High risk, low value unless OCM-free operation is required

**Key Insight**: Ramen is fundamentally an OCM extension. While some dependencies can be abstracted or removed, the core architecture assumes OCM's multi-cluster management capabilities. Removing all OCM dependencies would result in a fundamentally different product.
