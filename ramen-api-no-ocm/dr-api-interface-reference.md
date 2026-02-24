# Ramen DR API Interface Reference

This document describes the API needed to build an orchestrator for Regional Disaster Recovery (RDR) based on Ramen [TODO: add ref] but with no
dependencies on Open Cluster Management (OCM) or Red Hat Advanced Cluster Management (RHACM).

All configuration and orchestration is done through the **Kubernetes API** using **Custom Resource Definitions (CRDs)**.

## Overview of Ramen Regional DR with OCM

Ramen RDR is supported in a three cluster configuration. 
- There is _hub_ cluster which runs the Ramen DR Orchestrator. It acts as the central control point where administrators
  define disaster recovery relationships using DRPolicy resources and orchestrate workload placement using
  DRPlacementControl (DRPC) resources. It uses ACM's ManifestWork to push configurations down to the managed clusters.
- There are two _managed_ clusters which are the primary and secondary clusters on which the workloads run. These
  clusters run the Ramen DR Manager, which executes local storage operations.


When the hub pushes a VolumeReplicationGroup (VRG) to a managed cluster, the local DR Manager creates individual
VolumeReplication (VR) resources for each Persistent Volume Claim (PVC). This directly commands the storage layer to
promote, demote, or resync the underlying volumes.

The managed cluster that is currently acting as the active (primary) is responsible for capturing the Kubernetes
Persistent Volume (PV) metadata and pushing it to a central S3 object store. During a failover or relocation, the target
(secondary) managed cluster downloads this metadata from S3 to restore the volumes before the application starts.

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

#### The Building Blocks (Custom Resources)

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

#### Orchestration Workflows
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

## Goals

- Minimum of changes needed in Ramen.
- Apart from inter-cluster resource management no changes to Ramen orchestration.
- Discovered workloads only, i.e. no support for ArgoCD Applications or OCM Subscriptions.

## Out of scope

- Cluster creation and inter-cluster communication.
- Setting up the storage mirroring or replication.
- Setting up the S3 compatibile object storage.

## Requirements

- 1 hub and 2 managed clusters
- S3 Storage and secrets set up (auto propagation?) - managed clusters must have access.
- Capability to write/read resources on the managed clusters from the hub.

## Ramen Regional DR without OCM

The core difference between Ramen RDR without OCM and Ramen RDR with OCM is in the communication of state between the
Hub cluster and the managed clusters. In Ramen without OCM a means must be provided to create/update VRGs on the managed
clusters from the hub cluster, and to report VRG state from the managed clusters to the hub cluster.

Other differences are
-
-

To handle state transfer the following mechanism is proposed.

When Ramen wishes to create or update a VRG on a managed cluster it will create or update a ManifestWork CR,
encapsulating the VRG, in the managed clusters namespace in the hub cluster. The vendor will provide infrastructure
(e.g. a controller) that will watch for ManifestWork changes in the managed cluster namespaces and apply any changes in the
encapsulated VRG to the VRG in the ramen-ops namespace in the corresponding managed cluster.

Similarly the vendor will provide infrastructure (e.g. a controller) that will watch for VRG changes in the managed clusters and
apply changes to a corresponding ManagedClusterView CR in the managed clusters namespace on the hub cluster.

### Alternatives

1. Dispense with the ManifestWork and ManagedClusterView wrappers and use VRGs directly. This will require some mechanism (namespaces) to distingguish between incoming and outgoing VRGs.
1. Dispense with the ManifestWork and ManagedClusterView wrappers and use new Ramen CRs which convey the same core information. This will eliminate use of OCM CRDs but will require the implementation of a translation layer in Ramen.
1. Rather than require the use of CRDs to manage VRG state provide a gRPC API. In this case the vendor would need to provide a server in a sidecar or plugin of some sort to field API calls and effect changes in the VRGs. A server would also be needed for the OCM case which would presumably in turn make use of OCM ManifestWork and ManagedClusterView.

## Resources Overview

| Resource | Scope | Where it lives | Purpose |
| :------- | :---- | :------------- | :------ |
| **DRPolicy** | Cluster | Hub | DR topology: clusters, replication, S3 |
| **DRCluster** | Cluster | Hub | Per-cluster DR config (S3 profile, fencing, region) |
| **DRPlacementControl** | Namespaced | Hub | Per-application DR: protect, relocate, failover |
| **VolumeReplicationGroup** | Namespaced | Managed clusters | Volume replication state (created by Ramen from DRPC) |
| **RamenConfig** | ConfigMap | Hub / each cluster | Operator config (S3 profiles, etc.); not a CRD |

For a **DR configuration and orchestration** interface, you primarily need **DRPolicy**, **DRCluster**, and **DRPlacementControl**. **VolumeReplicationGroup** is needed for detailed per-workload and per-PVC status if you show it (often on managed clusters).

---

## 1. DRPolicy

**Purpose**: Define DR topology (two clusters), replication schedule, and class selectors.

### 1.1 DRPolicy Endpoints (Kubernetes API)

- List: `GET /apis/ramendr.openshift.io/v1alpha1/drpolicies`
- Get: `GET /apis/ramendr.openshift.io/v1alpha1/drpolicies/{name}`
- Create: `POST /apis/ramendr.openshift.io/v1alpha1/drpolicies`
- Update: `PUT` or `PATCH` on the same URL as Get
- Delete: `DELETE` (same as Get)

### 1.2 DRPolicy Spec (Configuration)

| Field | Type | Required | Description |
| :---- | :--- | :------- | :---------- |
| `drClusters` | []string | Yes | Exactly 2 cluster names (DRCluster names) |
| `schedulingInterval` | string | Yes | Replication interval, e.g. `5m`, `1h`, `1d` |
| `replicationClassSelector` | LabelSelector | No | Select VolumeReplicationClass |
| `volumeSnapshotClassSelector` | LabelSelector | No | Select VolumeSnapshotClass (e.g. VolSync) |
| `volumeGroupSnapshotClassSelector` | LabelSelector | No | Select VolumeGroupSnapshotClass |

### 1.3 DRPolicy Status (For Your Interface)

| Field | Type | Description |
| :---- | :--- | :---------- |
| `conditions` | []Condition | e.g. `Validated` |
| `status.async.peerClasses` | []PeerClass | Resolved async replication peer classes (Regional DR) |
| `status.sync.peerClasses` | []PeerClass | Resolved sync replication peer classes (Metro DR) |

Use **conditions** to show validation/errors; use **peerClasses** to show which storage classes are available for DR.

---

## 2. DRCluster

**Purpose**: Per-managed-cluster DR configuration (S3 profile, region, fencing).

### 2.1 DRCluster Endpoints

- List: `GET /apis/ramendr.openshift.io/v1alpha1/drclusters`
- Get: `GET /apis/ramendr.openshift.io/v1alpha1/drclusters/{name}`
- Create / Update / Delete: same patterns as DRPolicy.

### 2.2 DRCluster Spec (Configuration)

| Field | Type | Required | Description |
| :---- | :--- | :------- | :---------- |
| `s3ProfileName` | string | Yes | S3 profile name from RamenConfig (for PV metadata) |
| `region` | string | No | Region; clusters in same region are Metro DR peers |
| `clusterFence` | string | No | `Unfenced` \| `Fenced` \| `ManuallyFenced` \| `ManuallyUnfenced` |
| `cidrs` | []string | No | Node CIDRs used for fencing |

### 2.3 DRCluster Status (For Your Interface)

| Field | Type | Description |
| :---- | :--- | :---------- |
| `phase` | string | `Available`, `Starting`, `Fencing`, `Fenced`, `Unfencing`, `Unfenced` |
| `conditions` | []Condition | e.g. `Validated`, `Clean`, `Fenced` |
| `maintenanceModes` | []ClusterMaintenanceMode | Storage maintenance mode state per provisioner/target |

Use **phase** and **conditions** for cluster readiness and fencing state in the UI.

---

## 3. DRPlacementControl (DRPC)

**Purpose**: Per-application DR – enable protection, relocate, or failover. This is the main resource your interface will **configure** and **orchestrate**.

### 3.1 DRPC Endpoints

- List: `GET /apis/ramendr.openshift.io/v1alpha1/namespaces/{namespace}/drplacementcontrols`
- Get: `GET /apis/ramendr.openshift.io/v1alpha1/namespaces/{namespace}/drplacementcontrols/{name}`
- Create: `POST /apis/ramendr.openshift.io/v1alpha1/namespaces/{namespace}/drplacementcontrols`
- Update: `PUT` or `PATCH` (use PATCH for partial updates, e.g. only `spec.action` and `spec.failoverCluster`)
- Delete: `DELETE` (same as Get)
- Watch: `GET .../drplacementcontrols?watch=1` (for live status)

### 3.2 DRPC Spec (Configuration and Orchestration)

| Field | Type | Required | Description |
| :---- | :--- | :------- | :---------- |
| `placementRef` | ObjectReference | Yes | Reference to Placement or PlacementRule (immutable) |
| `drPolicyRef` | ObjectReference | Yes | Reference to DRPolicy (immutable) |
| `pvcSelector` | LabelSelector | Yes | Labels to select PVCs to protect (immutable) |
| `preferredCluster` | string | No | Preferred cluster to run the app (initial or after relocate) |
| `failoverCluster` | string | No | Target cluster for **failover**; set when `action=Failover` |
| `action` | string | No | `Failover` \| `Relocate` – triggers the operation |
| `protectedNamespaces` | []string | No | Additional namespaces to protect (unmanaged resources) |
| `kubeObjectProtection` | object | No | Kube object capture/recovery (recipe, interval, selector) |
| `volSyncSpec` | object | No | VolSync-specific config (mover, TLS, etc.) |

**Orchestration (what to set from your UI):**

- **Enable protection**: Create DRPC with `placementRef`, `drPolicyRef`, `pvcSelector`, and optionally `preferredCluster`.
- **Relocate (planned migration)**: Set `spec.action = "Relocate"` and `spec.preferredCluster = <target-cluster>`.
- **Failover (disaster recovery)**: Set `spec.action = "Failover"` and `spec.failoverCluster = <target-cluster>`.
- **Clear action**: After operation completes, you can set `spec.action = ""` and clear `spec.failoverCluster` when applicable.

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

### DRPC Progression (status.progression)

Use for progress bars or step indicators. Examples:

- `CreatingMW`, `WaitForReadiness`, `CheckingFailoverPrerequisites`, `FailingOverToCluster`,
  `WaitingForResourceRestore`, `UpdatedPlacement`, `EnsuringVolSyncSetup`, `Completed`, `Paused`, `Deleted`, etc.

Full set is in `api/v1alpha1/drplacementcontrol_types.go` (ProgressionStatus constants).

### DRPC Conditions (status.conditions)

Standard Kubernetes conditions; check `type` and `status` (True/False).

| Type | Meaning |
| :--- | :------ |
| **Available** | Whether the cluster in `preferredDecision` is ready for the workload. Reason can be `Progressing`, `Success`, `Paused`, etc. |
| **PeerReady** | Whether a peer cluster is ready for failover/relocate. Reason e.g. `Success`, `NotStarted`, `Paused`. |
| **Protected** | Whether the workload is protected. Reason: `Protected`, `Progressing`, `Error`, `Unknown`. |[=
Use **Message** for user-facing text; use **Reason** for programmatic handling.

---

## 4. VolumeReplicationGroup (VRG)

**Purpose**: Per-cluster volume replication state. Usually **created and updated by Ramen** from the DRPC; your
interface may **read** it for detailed status (e.g. per-PVC sync status).

### 4.1 Where VRG Lives

VRGs are on **managed clusters**, not the hub. To show VRG status from a hub-based UI you need either OCM
ManagedClusterView (current Ramen design) or direct API access to managed clusters.

### 4.2 VRG Endpoints (on each managed cluster)

- List: `GET /apis/ramendr.openshift.io/v1alpha1/namespaces/{namespace}/volumereplicationgroups`
- Get: `GET /apis/ramendr.openshift.io/v1alpha1/namespaces/{namespace}/volumereplicationgroups/{name}`

### 4.3 VRG Spec (Reference Only – Usually Managed by Ramen)

- `pvcSelector`, `replicationState` (primary/secondary), `s3Profiles`, `async`/`sync`/`volSync`, etc.

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

---

## 5. RamenConfig (S3 and Operator Config)

RamenConfig is stored in a **ConfigMap**, not a CRD. S3 profiles are inside it.

- **Hub**: ConfigMap name `ramen-hub-operator-config`, key `ramen_manager_config.yaml`.
- **DR cluster**: ConfigMap name `ramen-dr-cluster-operator-config`, key `ramen_manager_config.yaml`.

To **configure S3** (for DR clusters and policies), you need to read/update this ConfigMap and the structure under `s3StoreProfiles`. Each profile has `s3ProfileName`, `s3Bucket`, `s3CompatibleEndpoint`, `s3Region`, `s3SecretRef`, and optional `caCertificates`, `veleroNamespaceSecretKeyRef`. Secrets use keys `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

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

## Questions

In the Vendor case does Ramen create ManifestWork CRs for the Vendor impl. to distribute and read ManagedClusterView CRs
created/updated by the Vendir impl, or do we write/read some generic CRs which the Ramen orchestrator will convert to/from ManifestWork/ManagedClusterView?

Are we expecting vendor to use the non-OCM pieces of Ramen, e.g the VRG controller? I would expect so otherwise the ymay
as well just write everything themselves.
