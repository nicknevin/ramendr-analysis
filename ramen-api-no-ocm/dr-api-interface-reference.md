# Ramen DR API Interface Reference

This document describes the API needed to build an orchestrator for Regional Disaster Recovery (RDR) based on Ramen [TODO: add ref] but with no
dependencies on Open Cluster Management (OCM) or Red Hat Advanced Cluster Management (RHACM).

All configuration and orchestration is done through the **Kubernetes API** using **Custom Resource Definitions (CRDs)**.

## ToDo

- cover DRClusterConfig
- validating StorageClass etc.

## Goals

- Provide an API specification 
- Keep to a minimum the changes required to Ramen.

## Non-goals

- Support for ArgoCD Applications or OCM Subscriptions, i.e. only discovered applications will be supported.
- Cluster creation and inter-cluster communication.
- Setting up the storage mirroring or replication.
- Setting up the S3 compatible object storage.

## Overview of Ramen Regional DR with OCM

### Core Architecture and Concepts

Ramen utilizes a Hub-and-Spoke architecture built on top of OCM.
- Hub Cluster (Control Plane): Runs the Ramen DR Orchestrator. It acts as the central control point where administrators
  define DR policies and manage application placement. It uses ACM's ManifestWork to push configurations down to the
  managed clusters.
- Managed Clusters (Data Plane): The primary and secondary clusters where the actual workloads and storage reside. These
  clusters run the Ramen DR Manager, which executes local storage operations.
- S3 Object Store: Because the Hub cluster only manages application deployment manifests, Ramen requires an S3 object
  store to replicate the dynamically generated Kubernetes Persistent Volume (PV) cluster data from the primary cluster
  to the secondary cluster.
- Asynchronous Storage Replication: For Regional DR, the underlying storage replicates volume data asynchronously based
  on a defined schedule, meaning there is an expected Recovery Point Objective (RPO) and potential data loss during an
  unplanned failover.

### The Building Blocks (Custom Resources)

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

### Orchestration Workflows

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

### Implementation

Ramen is implemented in eight controllers. Three run on the hub cluster and the other five on the managed clusters.
The following sections provide a high-level overview of their
responsibilities. 

Note: Fencing is a Metro DR feature and isn't applicable to Regional DR. References below to
fencing are just for completeness and can be ignored for the purposes of Regional DR.

#### Hub controllers

##### DRPlacementControlReconciler

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

##### DRPolicyReconciler

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

##### DRClusterReconciler

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

#### Managed cluster controllers

##### VolumeReplicationGroupReconciler

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

##### DRClusterConfigReconciler

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

##### ReplicationGroupSourceReconciler

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

##### ReplicationGroupDestinationReconciler

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

###### ProtectedVolumeReplicationGroupListReconciler

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

## Ramen Regional DR without OCM

The core difference between Ramen RDR without OCM and Ramen RDR with OCM is in the communication of state between the
Hub cluster and the managed clusters. In Ramen without OCM a means must be provided to create/update VRGs on the managed
clusters from the hub cluster, and to report VRG state from the managed clusters to the hub cluster.

[ToDo: Mention any other significant differences]

To handle state transfer the following mechanism is proposed.

When Ramen wishes to create or update a VRG on a managed cluster it will create or update a ManifestWork CR,
encapsulating the VRG, in the managed clusters namespace in the hub cluster. The vendor will provide infrastructure
(e.g. a controller) that will watch for ManifestWork changes in the managed cluster namespaces and apply any changes in the
encapsulated VRG to the VRG in the ramen-ops namespace in the corresponding managed cluster.

Similarly the vendor will provide infrastructure (e.g. a controller) that will watch for VRG changes in the managed clusters and
apply changes to a corresponding ManagedClusterView CR in the managed clusters namespace on the hub cluster.

### Alternatives

1. Dispense with the ManifestWork and ManagedClusterView wrappers and use VRGs directly. This will require some mechanism (namespaces) to distinguish between incoming and outgoing VRGs.
1. Dispense with the ManifestWork and ManagedClusterView wrappers and use new Ramen CRs which convey the same core information. This will eliminate use of OCM CRDs but will require the implementation of a translation layer in Ramen.
1. Rather than require the use of CRDs to manage VRG state provide a gRPC API. In this case the vendor would need to provide a server in a sidecar or plugin of some sort to field API calls and effect changes in the VRGs. A server would also be needed for the OCM case which would presumably in turn make use of OCM ManifestWork and ManagedClusterView.

## Initial setup

Before placing a workload under DR protection via a DRPC the following setup is required.

- vendor (third-party) controllers etc
- S3 storage
- storage replication
- a DRCluster CR in the hub cluster for each managed cluster
    Per-managed-cluster DR configuration.
    - The name of the CR must be the cluster name
    - The only field which is relevant for RDR is *s3ProfileName* which references the S3 profile for the cluster
      defined in the DRPolicy CR.

    | Field | Type | Required | Description |
    | :---- | :--- | :------- | :---------- |
    | `s3ProfileName` | string | Yes | S3 profile name from RamenConfig (for PV metadata) |

- a DRPolicy CR in the hub cluster
    Defines the two managed clusters, the replication schedule, and class selectors.

    | Field | Type | Required | Description |
    | :---- | :--- | :------- | :---------- |
    | `drClusters` | []string | Yes | Exactly 2 cluster names (DRCluster names) |
    | `schedulingInterval` | string | Yes | Replication interval, e.g. `5m`, `1h`, `1d` |
    | `replicationClassSelector` | LabelSelector | No | Select VolumeReplicationClass |
    | `volumeSnapshotClassSelector` | LabelSelector | No | Select VolumeSnapshotClass (e.g. VolSync) |
    | `volumeGroupSnapshotClassSelector` | LabelSelector | No | Select VolumeGroupSnapshotClass |

- the RamenConfig in a ConfigMap in each cluster Defines operator configuration (S3 profiles, etc.). On the hub the
    ConfigMap is named `ramen-hub-operator-config` and on the managed clusters it is named
    `ramen-dr-cluster-operator-config`.
    
    To **configure S3** (for DR clusters and policies), you need to read/update this ConfigMap and the structure under
    `s3StoreProfiles`. Each profile has `s3ProfileName`, `s3Bucket`, `s3CompatibleEndpoint`, `s3Region`, `s3SecretRef`,
    and optional `caCertificates`, `veleroNamespaceSecretKeyRef`. Secrets use keys `AWS_ACCESS_KEY_ID` and
    `AWS_SECRET_ACCESS_KEY`.
- StorageClass, VolumeSnapshotClass, VolumeReplicationClass, VolumeGroupReplicationClass, VolumeGroupSnapshotClass

## DR Operations

Workloads are DR protected by creating a DRPC CR which specifies the workload resources to be protected and
the desired state for the resources.

**Orchestration (what to set from your UI):**

- **Enable protection**: Create DRPC with `drPolicyRef`, `pvcSelector`, and `preferredCluster`.
- **Relocate (planned migration)**: Set `spec.action = "Relocate"` and `spec.preferredCluster = <target-cluster>`.
- **Failover (disaster recovery)**: Set `spec.action = "Failover"` and `spec.failoverCluster = <target-cluster>`.
- **Clear action**: After operation completes, you can set `spec.action = ""` and clear `spec.failoverCluster` when applicable.

### 3.2 DRPC Spec (Configuration and Orchestration)

| Field | Type | Required | Description |
| :---- | :--- | :------- | :---------- |
| `drPolicyRef` | ObjectReference | Yes | Reference to DRPolicy (immutable) |
| `pvcSelector` | LabelSelector | Yes | Labels to select PVCs to protect (immutable) |
| `preferredCluster` | string | No | Preferred cluster to run the app (initial or after relocate) |
| `failoverCluster` | string | No | Target cluster for **failover**; set when `action=Failover` |
| `action` | string | No | `Failover` \| `Relocate` – triggers the operation |
| `protectedNamespaces` | []string | No | Additional namespaces to protect (unmanaged resources) |
| `kubeObjectProtection` | object | No | Kube object capture/recovery (recipe, interval, selector) |
| `volSyncSpec` | object | No | VolSync-specific config (mover, TLS, etc.) |


## Status Reporting

### 1.3 DRPolicy Status (For Your Interface)

| Field | Type | Description |
| :---- | :--- | :---------- |
| `conditions` | []Condition | e.g. `Validated` |
| `status.async.peerClasses` | []PeerClass | Resolved async replication peer classes (Regional DR) |
| `status.sync.peerClasses` | []PeerClass | Resolved sync replication peer classes (Metro DR) |

Use **conditions** to show validation/errors; use **peerClasses** to show which storage classes are available for DR.

### 2.3 DRCluster Status (For Your Interface)

| Field | Type | Description |
| :---- | :--- | :---------- |
| `phase` | string | `Available`, `Starting`, `Fencing`, `Fenced`, `Unfencing`, `Unfenced` |
| `conditions` | []Condition | e.g. `Validated`, `Clean`, `Fenced` |
| `maintenanceModes` | []ClusterMaintenanceMode | Storage maintenance mode state per provisioner/target |

Use **phase** and **conditions** for cluster readiness state in the UI.


### 3.3 DRPC Status (Critical for Your Interface)

| Field | Type | Description |
| :---- | :--- | :---------- |
| `phase` | string | High-level state (see below) |
| `progression` | string | Fine-grained step (see below) |
| `preferredDecision` | { clusterName, clusterNamespace } | Current cluster where workload is placed |
| `conditions` | []Condition | Available, PeerReady, Protected (see below) |
| `actionStartTime` | time | When current action started |
| `actionDuration` | duration | Duration of current action |
| `lastUpdateTime` | time | Last status update |
| `lastGroupSyncTime` | time | Last successful sync of all PVCs |
| `lastGroupSyncDuration` | duration | Last sync duration |
| `lastGroupSyncBytes` | int64 | Bytes transferred in last sync |
| `lastKubeObjectProtectionTime` | time | Last kube object capture |
| `resourceConditions` | VRGConditions | Aggregated VRG conditions from managed cluster |

### DRPC Phase (status.phase)

| Value | Meaning |
| :---- | :------ |
| (empty) | Initial / not yet started |
| `WaitForUser` | Waiting for user action (e.g. after hub recovery) |
| `Initiating` | Action is being prepared |
| `Deploying` | Initial deployment in progress |
| `Deployed` | Initial deployment done |
| `FailingOver` | Failover in progress |
| `FailedOver` | Failover completed |
| `Relocating` | Relocation in progress |
| `Relocated` | Relocation completed |
| `Deleting` | DRPC is being deleted |

### DRPC Conditions (status.conditions)

Standard Kubernetes conditions; check `type` and `status` (True/False).

| Type | Meaning |
| :--- | :------ |
| **Available** | Whether the cluster in `preferredDecision` is ready for the workload. Reason can be `Progressing`, `Success`, `Paused`, etc. |
| **PeerReady** | Whether a peer cluster is ready for failover/relocate. Reason e.g. `Success`, `NotStarted`, `Paused`. |
| **Protected** | Whether the workload is protected. Reason: `Protected`, `Progressing`, `Error`, `Unknown`. |[=

### 4.4 VRG Status (For Your Interface)

| Field | Type | Description |
| :---- | :--- | :---------- |
| `state` | string | `Primary` \| `Secondary` \| `Unknown` |
| `conditions` | []Condition | Summary conditions (see VRG conditions below) |
| `protectedPVCs` | []ProtectedPVC | Per-PVC status and conditions |
| `lastGroupSyncTime` | time | Last successful sync of all PVCs |
| `lastGroupSyncDuration` | duration | Last sync duration |
| `lastGroupSyncBytes` | int64 | Bytes in last sync |
| `kubeObjectProtection` | object | Capture to recover from (if kube object protection enabled) |
| `prepareForFinalSyncComplete` | bool | Relocation: final sync prepared |
| `finalSyncComplete` | bool | Relocation: final sync done |

### 4.5 VRG Condition Types (status.conditions and protectedPVCs[].conditions)

| Type | Level | Meaning |
| :--- | :---- | :------ |
| **DataReady** | VRG + PVC | PV data ready (e.g. for failover/relocate) |
| **DataProtected** | VRG + PVC | PV data in sync with peer |
| **ClusterDataReady** | VRG | PV/PVC cluster data restored (S3) |
| **ClusterDataProtected** | VRG | PV cluster data uploaded to S3 |
| **KubeObjectsReady** | VRG | Kube objects restored (if enabled) |
| **NoClusterDataConflict** | VRG | No conflict between clusters |
| **AutoCleanup** | VRG | Post-DR cleanup state |
| **ReplicationSourceSetup** | PVC (VolSync) | VolSync source setup |
| **ReplicationDestinationSetup** | PVC (VolSync) | VolSync destination setup |
| **PVsRestored** | PVC (VolSync) | VolSync PVCs restored |
| **FinalSyncInProgress** | PVC (VolSync) | Final sync in progress |

Each condition has **status** (True/False/Unknown), **reason**, and **message** – use these for UI and automation.

## What you need to implement in order to use ramen for RDR

- Capability to write/read resources on the managed clusters from the hub.

---

## Summary: What You Need for a DR Interface

### To Configure DR

1. **DRPolicy**: Create/update/list – define the two clusters and replication (scheduling interval, class selectors).
2. **DRCluster**: Create/update/list – per-cluster S3 profile, region, fencing.
3. **RamenConfig** (ConfigMap): Read/update S3 profiles and operator settings.
4. **DRPlacementControl**: Create with `placementRef`, `drPolicyRef`, `pvcSelector`, `preferredCluster` to protect an application.

### To Orchestrate DR

1. **DRPlacementControl**:
   - **Relocate**: PATCH `spec.action = "Relocate"`, `spec.preferredCluster = <target>`.
   - **Failover**: PATCH `spec.action = "Failover"`, `spec.failoverCluster = <target>`.
   - **Clear**: PATCH `spec.action = ""`, clear `spec.failoverCluster` when appropriate.

### To Show Status

1. **DRPlacementControl**: Watch/list and display:
   - `status.phase` (high-level state)
   - `status.progression` (current step)
   - `status.preferredDecision.clusterName` (current cluster)
   - `status.conditions` (Available, PeerReady, Protected)
   - `status.actionStartTime`, `status.actionDuration`, `status.lastGroupSyncTime`, etc.
2. **DRPolicy**: `status.conditions`, `status.async.peerClasses`, `status.sync.peerClasses`.
3. **DRCluster**: `status.phase`, `status.conditions`, `status.maintenanceModes`.
4. **VolumeReplicationGroup** (optional, on managed clusters): `status.state`, `status.conditions`, `status.protectedPVCs`, sync times/bytes.

### Suggested Watch Targets

- `DRPlacementControl` in the relevant namespace(s) – for live DR state and progression.
- `DRPolicy` and `DRCluster` – for validation and cluster state (less frequent updates).

This gives you everything needed to implement a UI or orchestrator that configures DR (policies, clusters, applications) and runs relocate/failover while showing accurate status and progression.

## Deletion and Garbage Collection

## Security

## Secondary hub

## Coexistent OCM and Vendor Ramen

Not supported.

## Other considerations

Suppose we wanted to replace the use of S3. What would it take to do this?
