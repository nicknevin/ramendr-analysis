# Failover Operations Sequence Diagrams

This directory contains comprehensive sequence diagrams for failover operations in Ramen, with special focus on VolumeReplicationGroup (VRG) objects.

## Documents

### 1. [failover-sequence-diagrams.md](./failover-sequence-diagrams.md)

**Mermaid sequence diagrams** showing:

- Main failover sequence (complete flow)
- Detailed VRG failover sequence (4 phases)
- VRG state transitions during failover
- Failover with VolumeGroupReplication (CephFS Consistency Groups)
- Failover prerequisites check sequence
- Error handling during failover

### 2. [failover-sequence-diagrams-graphviz.dot](./failover-sequence-diagrams-graphviz.dot)

**Graphviz diagrams** for detailed visualization. Generate images with:

```bash
# Generate SVG (recommended for web)
dot -Tsvg failover-sequence-diagrams-graphviz.dot -o failover-sequence-diagrams-graphviz.svg

# Generate PNG (for documents)
dot -Tpng failover-sequence-diagrams-graphviz.dot -o failover-sequence-diagrams-graphviz.png

# Generate PDF (for printing)
dot -Tpdf failover-sequence-diagrams-graphviz.dot -o failover-sequence-diagrams-graphviz.pdf
```

## Key Components in Failover Operations

### Hub Cluster Components

- **DRPlacementControl**: Orchestrates the failover operation
- **ManifestWork (OCM)**: Deploys VRG to managed clusters
- **ManagedClusterView (OCM)**: Reads VRG status from managed clusters
- **Placement/PlacementDecision (OCM)**: Controls cluster selection

### Managed Cluster Components

- **VolumeReplicationGroup (VRG)**: Manages replication state and data protection
- **DR Cluster Operator**: Processes VRG on managed clusters
- **VolumeReplication (CSI Addons)**: Handles volume replication for sync DR
- **VolSync**: Handles async replication via snapshots
- **ReplicationGroupSource/Destination**: Handles CephFS consistency groups

## Failover Flow Overview

### Phase 1: Initiation

1. User sets `DRPC.spec.action = Failover` and `spec.failoverCluster = target-cluster`
2. DRPlacementControl validates the failover target
3. Checks if workload is protected (`ConditionProtected = True`)
4. Validates failover prerequisites (MaintenanceMode activations, etc.)

### Phase 2: VRG Creation/Update

1. Creates or updates VRG ManifestWork with `ReplicationState = Primary` and `Action = Failover`
2. ManifestWork deploys VRG to the target managed cluster
3. VRG Controller on managed cluster receives reconcile event

### Phase 3: VRG Processing as Primary

1. **Restore Cluster Data**: Restores PVs/PVCs from S3 metadata if needed
2. **Promote Volumes**:
   - For Sync DR: Promotes VolumeReplication to Primary
   - For Async DR: Promotes ReplicationDestination (VolSync)
3. **Restore Kube Objects**: Restores application resources from Velero backup (if enabled)
4. **Update VRG Status**: Sets `DataReady = True` condition

### Phase 4: Verification and Completion

1. Hub operator reads VRG status via ManagedClusterView
2. Verifies VRG readiness (`DataReady = True`, VolSync setup complete)
3. Updates PlacementDecision to target cluster
4. Workload relocates to target cluster
5. Sets DRPC state to `FailedOver`

## Key VRG Operations During Failover

### VRG State Transitions

```text
Secondary → ValidatingFailover → CheckingPrerequisites →
CreatingPrimaryVRG → DeployingVRG → VRGDeployed →
ProcessingAsPrimary → RestoringData → PromotingVolumes →
VolumesPromoted → RestoringKubeObjects → DataReady →
UpdatingPlacement → FailedOver
```

### VRG Spec Changes

- `Spec.ReplicationState`: `Secondary` → `Primary`
- `Spec.Action`: Set to `VRGActionFailover`
- `Spec.Sync` or `Spec.Async`: Updated based on DRPolicy
- Annotations: Updated with destination cluster, DRPC UID, etc.

### VRG Status Conditions

- `DataReady`: Indicates volumes are ready
- `ClusterDataReady`: Indicates PVs/PVCs restored
- `KubeObjectsReady`: Indicates kube objects restored (if enabled)
- `DataProtected`: Indicates data protection status

## Prerequisites for Failover

### Metro DR (Synchronous)

- MaintenanceMode must be activated for protected PVCs
- Storage backend must support failover activation
- Both clusters must be reachable (for validation)

### Regional DR (Asynchronous)

- MaintenanceMode may be required for certain storage backends
- ReplicationDestination must be ready (for VolSync)
- S3 store must be accessible for metadata restore

## Error Scenarios

### Common Errors

1. **Invalid failover target**: Target cluster is not a valid secondary
2. **Workload not protected**: `ConditionProtected != True`
3. **Prerequisites not met**: MaintenanceMode not activated
4. **VRG deployment failure**: ManifestWork fails to deploy
5. **VRG not ready**: DataReady condition not met
6. **VRG processing error**: Controller fails to process as Primary

### Error Handling

- Errors set appropriate conditions on DRPC
- Operations are requeued and retried
- Progression status tracks current phase
- User is notified via events and conditions

## Viewing Diagrams

### Mermaid Diagrams

The Mermaid diagrams in `failover-sequence-diagrams.md` can be viewed:

- In GitHub (renders automatically)
- In Markdown viewers that support Mermaid (VS Code, Obsidian, etc.)
- Online at [Mermaid Live Editor](https://mermaid.live)

### Graphviz Diagrams

Generate images from `failover-sequence-diagrams-graphviz.dot`:

```bash
# Install Graphviz (if needed)
# Ubuntu/Debian: sudo apt-get install graphviz
# macOS: brew install graphviz
# Fedora: sudo dnf install graphviz

# Generate SVG (recommended for web)
dot -Tsvg failover-sequence-diagrams-graphviz.dot -o failover-sequence-diagrams-graphviz.svg

# Generate PNG (for documents)
dot -Tpng failover-sequence-diagrams-graphviz.dot -o failover-sequence-diagrams-graphviz.png

# Generate PDF (for printing)
dot -Tpdf failover-sequence-diagrams-graphviz.dot -o failover-sequence-diagrams-graphviz.pdf
```

## Code References

### Key Files

- **Failover Logic**: `internal/controller/drplacementcontrol.go`
  - `RunFailover()`: Main failover orchestration
  - `switchToFailoverCluster()`: Initiates failover
  - `switchToCluster()`: Switches to target cluster
  - `createVRGManifestWorkAsPrimary()`: Creates/updates VRG as Primary

- **VRG Processing**: `internal/controller/volumereplicationgroup_controller.go`
  - `processAsPrimary()`: Processes VRG as Primary
  - `reconcileAsPrimary()`: Reconciles Primary state
  - `clusterDataRestore()`: Restores PVs/PVCs from S3

- **Volume Replication**: `internal/controller/vrg_volrep.go`
  - Handles VolumeReplication promotion for sync DR

- **VolSync**: `internal/controller/vrg_volsync.go`
  - Handles VolSync ReplicationDestination for async DR

- **VolumeGroupReplication**: `internal/controller/vrg_volgrouprep.go`
  - Handles CephFS consistency groups

## Related Documentation

- [OCM Dependencies Analysis](./ocm-dependencies-analysis.md)
- [OCM Dependencies Diagrams](./ocm-dependencies-diagrams.md)
- [OCM Dependencies Isolation](./ocm-dependencies-isolation.md)

## Summary

Failover operations in Ramen are orchestrated by DRPlacementControl on the hub cluster, which:

1. **Validates** the failover target and prerequisites
2. **Creates/Updates** VRG as Primary via ManifestWork
3. **Monitors** VRG processing via ManagedClusterView
4. **Verifies** VRG readiness before completing failover
5. **Updates** PlacementDecision to relocate workload

The VolumeReplicationGroup is central to failover operations, transitioning from Secondary to Primary state and orchestrating:

- Volume replication promotion
- Data restoration from S3
- Kube object restoration (if enabled)
- Application workload relocation

All operations are coordinated through OCM APIs (ManifestWork, ManagedClusterView, Placement) to provide scalable multi-cluster disaster recovery.
