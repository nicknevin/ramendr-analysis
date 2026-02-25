# Ramen Without OCM

This directory contains a strawman proposal for using Ramen to orchestrate Reginal Disaster Recovery (RDR) with no
dependencies on Open Cluster management (OCM).

It covers the changes needed to Ramen to facilitate this, the APIs which will be provided for managing DR and tracking status.

The dependencies Ramen currently has on OCM are analyzed in [OCM/RHACM Dependencies Documentation](../docs/OCM-DEPENDENCIES-README.md) and
this provides useful background information.

A brief overview of the Ramen architecture and implementation as it now is with OCM is [here](./ramen-arch-with-ocm.md).

For the purposes of discussion we shall refer to Vendor Ramen.

## Breaking the OCM Dependencies

As covered in [OCM/RHACM Dependencies Documentation](../docs/OCM-DEPENDENCIES-README.md) four core dependencies on OCM which need to be broken are
- ManifestWork - For resource deployment
- ManagedClusterView - For status reading
- Placement/PlacementDecision - For cluster selection
- ManagedCluster - For cluster discovery

They are addressed in the following sections.

### ManagedCluster

There will be no cluster discovery or management in Ramen without OCM, and hence no ManagedCluster CRs.
The managed clusters are each defined in a DRCluster CR on the hub cluster. See [TODO ref].

### Placement and PlacementDecision

In OCM the Placement and PlacementDecision CRs provide sophisticated capabilities for managing the placement of workloads on clusters.
In Ramen without OCM there is just a simple choice of on which managed cluster the workload is to be deployed and this
is simply be specified in the DRPC and there is no need for the Placement and PlacementDecision CRs.

### ManifestWork and ManagedClusterView

In Ramen with OCM, ManifestWork CRs are used to deploy resources (such as a VRG) from the hub to the managed
clusters. To deploy a resource from the hub to a managed cluster, a ManifestWork CR encapsulating the resource is
created on the hub in the namespace of the target managed cluster and OCM takes care of deploying the resource in the
target managed cluster and updating the status of the ManifestWork CR to reflect the deployment status.

Similarly to get the status of resources on a managed cluster a ManagedClusterView CR is created on the hub describing
the resources to get status for and OCM takes care of updating the status ManagedClusterView CR retrieving with the
status of the resources on the managed cluster.

In Ramen without OCM it is proposed to use the same ManifestWork and ManagedClusterView CRs for deploying resources and
getting resource state, with the vendor using Ramen without OCM, providing the infrastructure for communication between
the hub and managed clusters and updating the status of ManifestWork and ManagedClusterView CRs.

The architecture and implementation of the infrastructure for managing ManifestWork and ManagedClusterView is left to
the vendors discretion. OCM manages ManifestWork for example via a pull model where an agent on the managed clusters
watches for changes in the ManifestWork CR specs on the hub, applies changes to the resources in the managed cluster,
while reporting the status of the updates by updating the status of the ManifestWork CR on the hub. Aletrnatively a push model could
also be used where an agent on the hub pushes spec changes to the managed clusters.

Whatever architecture is used it must implement the following APIs for the ManifestWork and ManagedClusterView CRs.

#### ManifestWork API

Ramen without OCM creates the following resources types
- VRG
- Namespace
- DRClusterConfig

Status Conditions:
- WorkApplied - Manifests have been applied
- WorkAvailable - Resources are available/ready
- WorkDegraded - Resources have issues
- WorkProgressing - Application in progres

#### ManagedClusterView API

Ramen without OCM gets the status of the following resources types via ManagedClusterView CRs
- VolumeReplicationGroup
- DRClusterConfig
- StorageClass
- VolumeSnapshotClass
- VolumeReplicationClass
- VolumeGroupSnapshotClass
- VolumeGroupReplicationClass

**Status Condition Structure:**
- **Type:** `ConditionViewProcessing` (always "Processing")
- **Reason:** Can be:
  - `ReasonGetResourceFailed` (GetResourceFailed) - resource query failed
  - `GetResourceProcessing` - currently processing
  - Other OCM-defined reasons for status
- **Status:** `metav1.ConditionTrue` (True/False/Unknown)
- **Message:** Human-readable description or error message
- **LastTransitionTime:** When the condition changed

**Example Status:**
```yaml
status:
  conditions:
  - lastTransitionTime: "2021-06-15T21:05:11Z"
    message: Watching resources successfully
    reason: GetResourceProcessing
    status: "True"
    type: Processing
  result:
    apiVersion: ramendr.openshift.io/v1alpha1
    kind: VolumeReplicationGroup
    metadata:
    # ... actual resource data
```

**Status.Result Field:**
- Contains the raw JSON-encoded resource data (`Raw` field)
- Only populated when status is successful
- Marshaled/Unmarshaled as needed for type conversion

ManagedClusterView is a namespaced resource. To get the status of resources in managed cluster A the ManagedClusterView
CR is created in namespace A in the hub cluster.

#### Alternative Solutions

1. Dispense with the ManifestWork and ManagedClusterView wrappers and use VRGs directly. This will require some mechanism (namespaces) to distinguish between incoming and outgoing VRGs.
1. Dispense with the ManifestWork and ManagedClusterView wrappers and use new Ramen CRs which convey the same core information. This will eliminate use of OCM CRDs but will require the implementation of a translation layer in Ramen.
1. Rather than require the use of CRDs to manage VRG state provide a gRPC API. In this case the vendor would need to provide a server in a sidecar or plugin of some sort to field API calls and effect changes in the VRGs. A server would also be needed for the OCM case which would presumably in turn make use of OCM ManifestWork and ManagedClusterView.

## Requirements and Initial Setup

Before placing a workload under DR protection via a DRPC the following setup is required.

- The third-party controllers vendor (third-party) controllers etc. must be running.
- S3 compatible storage must be created amd configured.
- Storage replication must be enabled.
- A DRCluster CR must be created in the hub cluster for each managed cluster.
    This provides per-managed-cluster DR configuration.
    - The name of the CR must be the cluster name
    - The only field which is relevant for RDR is *s3ProfileName* which references the S3 profile for the cluster
      defined in the DRPolicy CR.

    | Field | Type | Required | Description |
    | :---- | :--- | :------- | :---------- |
    | `s3ProfileName` | string | Yes | S3 profile name from RamenConfig (for PV metadata) |

- A DRPolicy CR must be created in the hub cluster
    This defines the two managed clusters, the replication schedule, and class selectors.

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
- Any of the following required for volume replication must be defined.
    - StorageClass
    - VolumeSnapshotClass
    - VolumeReplicationClass
    - VolumeGroupReplicationClass
    - VolumeGroupSnapshotClass

## DR Operations API

The DRPlacementControl (DRPC) Custom Resource is the API for managing DR Operations. This CR specifies the workload resources to be protected and
the desired state for the resources.

**Orchestration (what to set from your UI):**

- **Enable protection**: Create DRPC with `drPolicyRef`, `pvcSelector`, and `preferredCluster`.
- **Relocate (planned migration)**: Set `spec.action = "Relocate"` and `spec.preferredCluster = <target-cluster>`.
- **Failover (disaster recovery)**: Set `spec.action = "Failover"` and `spec.failoverCluster = <target-cluster>`.
- **Clear action**: After operation completes, you can set `spec.action = ""` and clear `spec.failoverCluster` when applicable.

### DRPC Spec (Configuration and Orchestration)

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

**Note:** `placementRef` is a required field for the OCM version of Ramen. This references one of Placement or PlacementRule, which are OCM objects discussed earlier in this document. 
For the purposes of this work, this can safely be ignored and set to `null` and Ramen will ignore it.

### DRPC Statuses
| Field                          | Type              | Description                                                                                                                             |
| :----------------------------- | :---------------- | :-------------------------------------------------------------------------------------------------------------------------------------- |
| `Phase`                        | string            | The current state of the DRPC, e.g. `WaitForUser`, `Initiating`, `FailingOver`, etc.                                                    |
| `ObservedGeneration`           | int               | Most recently processed generation of DRPC                                                                                              |
| `ActionStartTime`              | Time              | Start time of most recent DR Action                                                                                                     |
| `ActionDuration`               | Duration          | Duration of most recent DR Action                                                                                                       |
| `Progression`                  | ProgressionStatus | Detailed progress for current process, e.g. `EnsuringVolumesAreSecondary`, `WaitingForResourceRestore`, `CheckingFailoverPrerequisites` |
| `PreferredDecision`            | PlacementDecision | The cluster on which the Application should be running                                                                                  |
| `Conditions`                   | []Condition       | e.g. `Available`, `PeerReady`, `Protected`                                                                                              |
| `ResourceConditions`           | VRGConditions     | Current condition of the VRGs ``                                                                                                        |
| `LastUpdateTime`               | Time              |                                                                                                                                         |
| `lastGroupSyncTime`            | Time              | Most recent sync time for all PVCs                                                                                                      |
| `lastGroupSyncDuration`        | Duration          | How long sync took to run                                                                                                               |
| `lastGroupSyncBytes`           | int               | Size of most recent sync                                                                                                                |
| `LastKubeObjectProtectionTime` | Time              | Most recent Kube Object Protection time (recipe, interval, selector)                                                                    |

For more detailed information on the DRPC CRD, examples of CRs, as well as a guide on usage and best practices, [Check here](https://github.com/mulbc/ramen/blob/docs/enhance-documentation/docs/drpc-crd.md)

## Creating a DR Manager or UI

Ramen provides feedback on the state of DR through the status and conditions of the DRPC, DRPolicy, DRCluster and VRG CRs [TODO: are there any other?]

### DRPolicy Status

| Field | Type | Description |
| :---- | :--- | :---------- |
| `conditions` | []Condition | e.g. `Validated` |
| `status.async.peerClasses` | []PeerClass | Resolved async replication peer classes (Regional DR) |

Use **conditions** to show validation/errors; use **peerClasses** to show which storage classes are available for DR.

### DRCluster Status

| Field | Type | Description |
| :---- | :--- | :---------- |
| `phase` | string | `Available`, `Starting`, `Fencing`, `Fenced`, `Unfencing`, `Unfenced` |
| `conditions` | []Condition | e.g. `Validated`, `Clean`, `Fenced` |
| `maintenanceModes` | []ClusterMaintenanceMode | Storage maintenance mode state per provisioner/target |

[TODO: Do the phase and maintenanceModes fields apply?]

Use **phase** and **conditions** for cluster readiness state in the UI.

### DRPC Status

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

#### DRPC Phase (status.phase)

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

#### DRPC Conditions (status.conditions)

Standard Kubernetes conditions; check `type` and `status` (True/False).

| Type | Meaning |
| :--- | :------ |
| **Available** | Whether the cluster in `preferredDecision` is ready for the workload. Reason can be `Progressing`, `Success`, `Paused`, etc. |
| **PeerReady** | Whether a peer cluster is ready for failover/relocate. Reason e.g. `Success`, `NotStarted`, `Paused`. |
| **Protected** | Whether the workload is protected. Reason: `Protected`, `Progressing`, `Error`, `Unknown`. |[=

#### VRG Status

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

#### VRG Condition Types (status.conditions and protectedPVCs[].conditions)

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

## Deletion and Garbage Collection

## Security

## Hub Recovery

Not supported.

## Other considerations

Clusters will run either Ramen with OCM or Ramen without OCM. Running both on a cluster will not be supported.

Suppose we wanted to replace the use of S3. What would it take to do this?
