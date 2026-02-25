# Overview of Ramen Regional DR with OCM

In Ramen all configuration and orchestration is done through the **Kubernetes API** using **Custom Resource Definitions (CRDs)**.

## Core Architecture and Concepts

Ramen utilizes a Hub-and-Spoke architecture built on top of OCM.
- Hub Cluster (Control Plane): Runs the Ramen DR Orchestrator. It acts as the central control point where administrators
  define DR policies and manage application placement. It uses OCM's ManifestWork to push configurations down to the
  managed clusters and ManagedClusterView to view the status of resources on the managed clusters.
- Managed Clusters (Data Plane): The primary and secondary clusters where the actual workloads and storage reside. These
  clusters run the Ramen DR Manager, which executes local storage operations.
- S3 Object Store: Because the Hub cluster only manages application deployment manifests, Ramen requires an S3 object
  store to replicate the dynamically generated Kubernetes Persistent Volume (PV) cluster data from the primary cluster
  to the secondary cluster.

## The Building Blocks (Custom Resources)

An orchestrator needs declarative APIs to manage state. Ramen uses the following hierarchy of Custom Resources (CRs):
- DRPolicy (Hub-scoped): Created by an administrator to define the DR relationship. It specifies the two paired managed
  clusters, specifies the schedulingInterval (the RPO target for storage sync), and defines the s3ProfileName used to
  store PV metadata.
- DRPlacementControl (DRPC) (Hub-scoped): The core workload-level orchestrator. The DRPC specifies:
    - pvcSelector: Identifies which PVCs in the application namespace need DR protection.
    - action: The current desired DR state, either empty (deploy), Relocate, or Failover.
    - preferredCluster / failoverCluster: The target destinations for the workload.
- VolumeReplicationGroup (VRG) (Managed Cluster-scoped): Created by the Hub operator on the managed clusters. It groups
  all protected PVCs for a specific application. The VRG is responsible for uploading/downloading the PV metadata
  to/from the S3 bucket and acting as the parent resource for individual volume replication.
- VolumeReplication (VR) (Managed Cluster-scoped): Created by the VRG for each individual PVC. It directly interfaces
  with the underlying storage's csi-addons to execute storage-level commands like promote, demote, and resync. This
  custom resource is defined in the CSI Addons for Volume Replication.

## Orchestration Workflows

- Initial Deployment & Steady State
    - Protection: When an application is deployed and assigned a DRPlacementControl on the Hub, Ramen intercepts the
      placement. It creates a VolumeReplicationGroup (VRG) in the Primary state on the active cluster, and a
      corresponding VRG in the Secondary state on the peer cluster.
    - Data Sync: The storage layer begins asynchronously mirroring the actual block/file data based on the DRPolicy
      schedule.
    - Metadata Backup: The VRG running on the primary cluster continuously captures the Kubernetes PV/PVC metadata and
      uploads it as compressed JSON to the configured S3 bucket.
- Relocate (Planned Migration / Disaster Avoidance)
    A Relocate action incurs no data loss but requires application downtime during the transition.
    - Initiation: The user updates the DRPC action to Relocate.
    - Quiesce and Final Sync: Ramen stops the application on the primary cluster (scaling it down to zero via ACM) to
      prevent new data writes. It then triggers and waits for a final data sync of the storage volumes to ensure the
      secondary cluster is 100% up to date.
    - Demote: The VRG on the old primary is updated to Secondary, which demotes the underlying storage volumes.
    - Promote and Restore: The VRG on the target cluster is updated to Primary. Ramen downloads the PV metadata from
      S3, restores the PVs into the cluster, and promotes the target storage volumes to make them writable.
    - App Start: Once the VRG reports ClusterDataReady, ACM deploys the application pods on the new primary cluster,
      which seamlessly bind to the restored PVs.
- Failover (Unplanned Disaster Recovery)
    A Failover is executed when the primary cluster is unreachable. It accepts data loss based on the last successful
    asynchronous storage sync (the RPO).
    - Initiation: The user updates the DRPC action to Failover and sets the failoverCluster. Ramen validates if the
      target cluster is alive and was an established secondary (checked via the PeerReady condition).
    - Force Promote & Metadata Restore: Because the primary is dead, no final sync or graceful shutdown can occur. The
      Hub immediately creates/updates the VRG on the surviving cluster to Primary. The local VRG downloads the latest
      available PV metadata from S3, recreates the PVs, and issues a "force promote" to the storage layer via the
      VolumeReplication CR.
    - App Start: Once the PVs are restored and the volumes are promoted, ACM deploys the application pods to the
      surviving cluster.
    - Healing (Post-Disaster): When the dead cluster eventually comes back online, its local volumes will be in an
      invalid, "split-brain" state. Ramen must fence or demote the old primary and force a resync so its storage
      discards divergent data and cleanly becomes the new secondary.

## Implementation

Ramen is implemented in eight controllers. Three run on the hub cluster and the other five on the managed clusters.
The following sections provide a high-level overview of their
responsibilities.

Note: Fencing is a Metro DR feature and isn't applicable to Regional DR. References below to
fencing are only for completeness and can be ignored for the purposes of Regional DR.

### Hub controllers

#### DRPlacementControlReconciler

Resources Managed:
- Primary: DRPlacementControl (RamenDR CRD)

Watches: ManifestWork, ManagedClusterView, PlacementRule, Placement, DRCluster, DRPolicy, Secret, PlacementDecision

Control Flow:
    User creates DRPC → Validates DRPolicy/DRClusters → Creates PlacementDecision
    → Deploys VRG via ManifestWork → Monitors VRG status via ManagedClusterView
    → Updates DRPC status → Handles failover/relocate actions

Key Responsibilities:
- Central orchestrator for DR workload placement
- Manages failover/relocation between clusters
- Monitors VolumeReplicationGroup (VRG) status on managed clusters
- Handles both async (replication) and sync (metro) DR
- Integrates with VolSync for asynchronous replication

#### DRPolicyReconciler

Resources Managed:
- Primary: DRPolicy (RamenDR CRD)
- Watches: ConfigMap, Secret, DRCluster, ManagedCluster, ManagedClusterView

Control Flow:
    DRPolicy created → Validates DRClusters exist → Validates peer replication classes
    → Creates S3 profiles in RamenOpsNamespace → Creates OCM policies for fencing
    → Updates DRPolicy conditions → Reports validation status

Key Responsibilities:
- Policy validation across DRClusters
- S3 storage profile creation
- Peer class (replication class) validation
- OCM policy creation for cluster fencing/unfencing
- Secret management for storage credentials

Conditions: ValidationFailed, DRClusterNotFound, DRClustersUnavailable, Succeeded

#### DRClusterReconciler

Resources Managed:
- Primary: DRCluster (RamenDR CRD)
- Watches: DRPlacementControl, DRPolicy, ManifestWork, ManagedClusterView, ConfigMap, Secret

Control Flow:
    DRCluster created → Initializing → Validates prerequisites (storage, network fencing)
    → Ready for use → On failover: Fencing → Fenced → Cleaned
    → On recovery: Unfencing → Unfenced → Ready

State Machine:
Initializing → Validating → Validated (Succeeded)
           ↓
        Fencing → Fenced → Cleaning → Cleaned
           ↓
      Unfencing → Unfenced → Validated

Key Responsibilities:
- Cluster registration and validation
- Fencing/unfencing orchestration for failover
- Storage class and replication class validation
- Network fence capability checks
- ManifestWork coordination

### Managed cluster controllers

#### VolumeReplicationGroupReconciler

Resources Managed:
- Primary: VolumeReplicationGroup (RamenDR CRD)
- Watches: PersistentVolumeClaim, VolumeReplication, VolumeGroupReplication, ConfigMap
- Owns: VolumeReplication, VolumeGroupReplication, VolSync resources, Velero backups

Control Flow:
    VRG created → Discovers PVCs (via labels/annotations) → Selects replication class
    → PRIMARY ROLE:
      ├─ Creates VolumeReplication/VolumeGroupReplication
      ├─ Creates snapshots for consistency
      ├─ Uploads PV/PVC metadata to S3
      └─ Protects Kubernetes objects via Velero

    → SECONDARY ROLE:
      ├─ Downloads PV/PVC from S3
      ├─ Restores PVs/PVCs on secondary cluster
      ├─ Enables secondary replication
      └─ Restores Kubernetes objects via Velero

Key Responsibilities:
- Manages volume replication (sync via CSI-Addons, async via VolSync)
- PVC discovery and filtering
- Snapshot creation and management
- Kubernetes object protection (Velero/OADP integration)
- Recipe execution (pre/post backup/restore hooks)
- Primary ↔ Secondary role transitions

#### DRClusterConfigReconciler

Resources Managed:
- Primary: DRClusterConfig (RamenDR CRD)
- Watches: StorageClass, VolumeSnapshotClass, VolumeReplicationClass, VolumeGroupReplicationClass, VolumeGroupSnapshotClass, NetworkFenceClass, ClusterClaim

Control Flow:
    DRClusterConfig created → Discovers storage drivers → Validates replication classes
    → Checks S3 reachability → Creates ClusterClaims → Updates status with capabilities

Key Responsibilities:
- Cluster capability discovery
- Storage driver validation (Ceph, etc.)
- Replication/snapshot class discovery
- S3 backend connectivity validation
- ClusterClaim creation for capabilities
- Ensures one DRClusterConfig per cluster

Conditions: Initializing, ConfigurationProcessed, ConfigurationFailed, S3 Reachable/Unreachable

#### ReplicationGroupSourceReconciler

Resources Managed:
- Primary: ReplicationGroupSource (RamenDR CRD)
- Watches: VolumeGroupSnapshot, PersistentVolumeClaim, ReplicationSource, VolumeSnapshot

Control Flow:
    ReplicationGroupSource created → Creates VolumeGroupSnapshot (consistency group)
    → Creates temporary PVCs from snapshots → Creates ReplicationSource per PVC
    → Syncs to remote via VolSync → Tracks sync status → Prepares for final sync

Key Responsibilities:
- Consistency group snapshots (Ceph CephFS)
- Source-side group replication orchestration
- VolSync integration for async replication
- Sync scheduling (manual and cron-based)
- Final sync preparation for failover

#### ReplicationGroupDestinationReconciler

Resources Managed:
- Primary: ReplicationGroupDestination (RamenDR CRD)
- Watches: ReplicationDestination, VolumeSnapshot

Control Flow:
    ReplicationGroupDestination created → Creates ReplicationDestination per PVC
    → Receives snapshots via VolSync → Recovers PVCs from snapshots
    → Tracks sync status → Can be paused via Spec.Paused

Key Responsibilities:
- Destination-side consistency group recovery
- ReplicationDestination resource management
- PVC recovery from replicated snapshots
- VolSync coordination
- Sync status tracking

#### ProtectedVolumeReplicationGroupListReconciler

Resources Managed:
- Primary: ProtectedVolumeReplicationGroupList (RamenDR CRD)
- Watches: None (one-shot discovery)

Control Flow:
    PVRGLIST created → Queries S3 using S3ProfileName → Parses S3 prefix hierarchy
    → Extracts namespace/VRG pairs → Downloads VRG metadata from S3
    → Populates status with discovered VRGs → Reconciliation complete

Key Responsibilities:
    Discovery of protected VRGs from S3 replica store
    S3 metadata querying and parsing
    VRG catalog generation for DR reporting
    One-shot operation (no continuous reconciliation)
