# Ramen Without OCM

This directory contains a straw-man proposal for using Ramen to orchestrate Regional Disaster Recovery with no
dependencies on Open Cluster management (OCM). We shall refer to this as _Ramen without OCM_.

The dependencies Ramen currently has on OCM are analyzed in [OCM/RHACM Dependencies Documentation](../docs/OCM-DEPENDENCIES-README.md) and
this provides useful background information.

## Breaking the OCM Dependencies

As covered in [OCM/RHACM Dependencies Documentation](../docs/OCM-DEPENDENCIES-README.md) four core dependencies on OCM which need to be broken out are
- ManifestWork - For resource deployment
- ManagedClusterView - For status reading
- Placement/PlacementDecision - For cluster selection
- ManagedCluster - For cluster discovery

They are addressed in the following sections.

### ManagedCluster

There will be no cluster discovery or management in Ramen without OCM, and hence no ManagedCluster CRs.
The managed clusters are each defined in a DRCluster CR on the hub cluster. See [DRCluster CRD](./drplacementcontrol.crd.md) for more information.

### Placement and PlacementDecision

In OCM the Placement and PlacementDecision CRs provide sophisticated capabilities for managing the placement of workloads on clusters.  In Ramen without OCM there is just a simple choice of on which managed cluster the workload is to be deployed. This can be specified in the DRPC and there is no need for the Placement and PlacementDecision CRs.

### ManifestWork and ManagedClusterView

In Ramen with OCM, ManifestWork CRs are used to deploy resources (such as a VRG) from the hub to the managed clusters. To deploy a resource from the hub to a managed cluster, a ManifestWork CR encapsulating the resource is
created on the hub in the namespace of the target managed cluster and OCM takes care of deploying the resource in the
target managed cluster and updating the status of the ManifestWork CR to reflect the deployment status.

Similarly, in order to get the status of resources on a managed cluster, a ManagedClusterView CR is created on the hub. While this CR describes the resources, it is OCM's responsibility to retrieve the status of these resources from the managed cluster as well as to update the status of the ManagedClusterViewCR accordingly.

In Ramen without OCM it is proposed to use the same ManifestWork and ManagedClusterView CRs for deploying resources and
getting resource state, but with the vendor using Ramen without OCM providing the infrastructure to
- in the case of ManifestWork, create, update or delete resources on the managed clusters and update the ManifestWork status on the hub
- in the case of ManagedClusterView, get the status of resources on managed clusters and update the status of ManagedClusterView on the hub

The architecture and implementation of this infrastructure is left to the vendor integrating with Ramen without OCM.  OCM for example manages ManifestWork with a pull model where an agent on the managed clusters watches for changes in ManifestWork CR specs on the hub and applies them to the resources in the managed cluster, while reporting the status of the updates by updating the status of the ManifestWork CR on the hub. Alternatively a push model might be used where an agent on the hub pushes resource changes to the managed clusters.

For purposes of discussion in this document, this vendor provided infrastructure shall be called the Object Transport System (OTS).

The OTS must implement the following APIs for the ManifestWork and ManagedClusterView CRs.

#### ManifestWork API

The ManifestWork Custom Resource is defined [here](./manifestwork.crd.md) and
OCM documentation explaining the ManifestWork concepts can be found
[here](https://open-cluster-management.io/docs/concepts/work-distribution/manifestwork/).

Ramen creates the following resources types via ManifestWork and these must be supported
in the OTS.
- VolumeReplicationGroup
- Namespace
- DRClusterConfig

Many of the fields in the ManifestWork spec are not used by Ramen and support for them is not required in the OTS. These fields are:
- deleteOption
- executor
- manifestConfigs:items:conditionRules
- manifestConfigs:items:feedbackRules
- manifestConfigs:items:updateStrategy

The OTS must set the following status Conditions in the ManifestWork CR according to the state of the resource(s) in the managed cluster.
- WorkApplied - manifests have been applied
- WorkAvailable - resources are available/ready
- WorkDegraded - resources have issues
- WorkProgressing - application in progress

#### ManagedClusterView API

The ManagedClusterView Custom Resource is defined [here](./managedclusterview.crd.md).  ManagedClusterView is a
namespaced resource. To get the status of resources in managed cluster X the ManagedClusterView CR is created in
namespace X in the hub cluster.

The OTS must watch for changes in the specs of ManagedClusterView CRs in the managed cluster namespace on the hub and
then query the required resources on the managed clusters and update the ManagedClusterView CR status.

Ramen gets the status of the following resources types via ManagedClusterView CRs and the OTS must support these.
- VolumeReplicationGroup
- DRClusterConfig
- StorageClass
- VolumeSnapshotClass
- VolumeReplicationClass
- VolumeGroupSnapshotClass
- VolumeGroupReplicationClass

[Here](./mcv-vrg-example.md) is an example of a successfully updated ManagedClusterView CR for a VolumeReplicationGroup CR on a managed cluster.

#### Alternative Solutions

A couple of alternatives to the above for dealing with the ManifestWork and ManagedClusterView dependency are:
1. Dispense with the ManifestWork and ManagedClusterView wrappers and use new Ramen "wrapper" CRs which convey the same
   core information. This will eliminate use of OCM CRDs but will require the implementation of a translation layer in
   Ramen.
2. Rather than use of CRDs to manage resource state provide a gRPC API. In this case the vendor would need to provide a
   server in a sidecar or plugin of some sort to field API calls and effect changes in the managed resources. A server
   would also be needed for the OCM case which would presumably could in turn make use of OCM ManifestWork and
   ManagedClusterView.

## Initial Setup Requirements

Before using Ramen without OCM to protect workloads the same setup as for Ramen with OCM is required with some
exceptions and additions. Namely
- The ManifestWork and ManagedClusterView CRDs from OCM must be installed in the hub cluster. There is no requirement for
  anything else from OCM/RHACM.
- The OTS must be installed and operating.
- Storage replication must be enabled.
- S3 compatible storage must be created and configured.
- A [DRCluster](./drcluster.crd.md) CR must be created in the hub cluster for each managed cluster.
    This provides per-managed-cluster DR configuration.
    - The name of the CR must be the cluster name
    - The only field which is relevant for Regional DR is *s3ProfileName* which references the S3 profile for the cluster
      defined in the DRPolicy CR.
- A [DRPolicy](./drpolicy.crd.md) CR must be created in the hub cluster.
    This defines the two managed clusters, the replication schedule, and class selectors.
- The RamenConfig which defines operator configuration (S3 profiles, etc.) must be defined in a ConfigMap in each cluster.
    On the hub the ConfigMap is named `ramen-hub-operator-config` and on the managed clusters it is named
    `ramen-dr-cluster-operator-config`. For Ramen without OCM the ClusterManagementDisabled field must be set to `true`.
- Any of the following required for Volume Replication must be defined.
    - StorageClass
    - VolumeSnapshotClass
    - VolumeReplicationClass
    - VolumeGroupReplicationClass
    - VolumeGroupSnapshotClass

See the Ramen documentation for the finer details.

## DR Operations API

The [DRPlacementControl](./drplacementcontrol.crd.md) (DRPC) Custom Resource is the API for managing DR Operations. This
CR specifies the workload resources to be protected and the desired state for the resources.

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
| `LastUpdateTime`               | Time              | Most recent update time a condition or the overall status were updated                                                                  |
| `lastGroupSyncTime`            | Time              | Most recent sync time for all PVCs                                                                                                      |
| `lastGroupSyncDuration`        | Duration          | How long sync took to run                                                                                                               |
| `lastGroupSyncBytes`           | int               | Size of most recent sync                                                                                                                |
| `LastKubeObjectProtectionTime` | Time              | Most recent Kube Object Protection time (recipe, interval, selector)                                                                    |

For more detailed information on the DRPC CRD, examples of CRs, as well as a guide on usage and best practices, [Check here](https://github.com/mulbc/ramen/blob/docs/enhance-documentation/docs/drpc-crd.md)

## Creating a DR Manager or UI

Ramen provides feedback on the state of DR through the status and conditions of the DRPC, DRPolicy, DRCluster and VRG CRs.

### DRPolicy Status

| Field | Type | Description |
| :---- | :--- | :---------- |
| `conditions` | []Condition | e.g. `Validated` |
| `status.async.peerClasses` | []PeerClass | Resolved async replication peer classes (Regional DR) |

Use **conditions** to show validation/errors; use **peerClasses** to show which storage classes are available for DR.

### DRCluster Status

| Field | Type | Description |
| :---- | :--- | :---------- |
| `phase` | string | `Available`, `Starting` |
| `conditions` | []Condition | e.g. `Validated` |

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
| **Protected** | Whether the workload is protected. Reason: `Protected`, `Progressing`, `Error`, `Unknown`. |

## Deletion and Garbage Collection

When a workload is removed from data protection Ramen will clean up resources it created on the hub and managed
clusters. It will be the responsibility of the OTS to delete the resources it created.
