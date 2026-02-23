# CSI-Addon Based DR: Architecture and Operation Workflows

This document describes the **basic architecture** and **operation workflows** when customers use standard **CSI-addon based DR** with Ramen. It maps the six typical storage DR operations to how Ramen and the CSI-addons VolumeReplication API work.

## Standard Storage DR Operations Covered

1. **Start data replication** – Enable protection and start replicating volumes to the peer cluster.
2. **Pause data sync** – Temporarily stop or pause replication (storage/VR state).
3. **Data re-sync** – Resume or re-synchronize data (e.g. before planned failover or after pause).
4. **Cluster-down failover** – Unplanned failover when the primary cluster is lost.
5. **Planned failover (migration)** – Planned relocation to the peer cluster.
6. **Failback** – Return the workload to the original primary cluster after recovery.

---

## Basic Architecture (CSI-Addon Based DR)

Ramen uses the [CSI-addons replication specification](https://github.com/csi-addons/spec/tree/main/replication) and the **VolumeReplication** (and optionally **VolumeGroupReplication**) APIs. The following diagram shows the main components.

```mermaid
graph TB
    subgraph "OCM Hub Cluster"
        User[User / Admin]
        DRPC[DRPlacementControl<br/>DRPC]
        DRPolicy[DRPolicy]
        Placement[Placement /<br/>PlacementDecision]
        MW[ManifestWork]
        MCV[ManagedClusterView]
    end

    subgraph "Managed Cluster A (e.g. Primary)"
        VRG_A[VolumeReplicationGroup<br/>VRG]
        VR_A[VolumeReplication<br/>VR - per PVC]
        DRClusterOp_A[DR Cluster Operator]
        CSI_A[CSI Driver +<br/>Replication Controller]
        App_A[Application<br/>Pods + PVCs]
        Storage_A[Storage Backend A]
    end

    subgraph "Managed Cluster B (e.g. Secondary)"
        VRG_B[VolumeReplicationGroup<br/>VRG]
        VR_B[VolumeReplication<br/>VR - per PVC]
        DRClusterOp_B[DR Cluster Operator]
        CSI_B[CSI Driver +<br/>Replication Controller]
        App_B[Application<br/>Pods + PVCs]
        Storage_B[Storage Backend B]
    end

    subgraph "Shared"
        S3[S3 Store<br/>PV/PVC metadata]
    end

    User --> DRPC
    DRPC --> DRPolicy
    DRPC --> Placement
    DRPC --> MW
    DRPC --> MCV
    MW -.deploys.-> VRG_A
    MW -.deploys.-> VRG_B
    MCV -.reads status.-> VRG_A
    MCV -.reads status.-> VRG_B

    VRG_A --> VR_A
    VRG_A --> S3
    VRG_B --> VR_B
    VRG_B --> S3
    DRClusterOp_A --> VRG_A
    DRClusterOp_B --> VRG_B
    VRG_A --> DRClusterOp_A
    VRG_B --> DRClusterOp_B

    VR_A --> CSI_A
    VR_B --> CSI_B
    CSI_A --> Storage_A
    CSI_B --> Storage_B
    Storage_A <-.replication.-> Storage_B
    App_A --> VR_A
    App_B --> VR_B
```

**Component roles:**

- **DRPC**: User-facing DR control; references DRPolicy and Placement; triggers relocate/failover.
- **DRPolicy**: Defines the two DR clusters and replication (e.g. scheduling interval, class selectors).
- **VRG**: One per application (per cluster); manages lifecycle of **VolumeReplication** (VR) resources and PV/PVC metadata in **S3**.
- **VolumeReplication (VR)**: CSI-addon CR per PVC; drives **start/pause/resync** and **primary/secondary** at the storage layer.
- **CSI Driver + Replication Controller**: Implements the replication and state transitions (primary/secondary, sync, resync).
- **S3**: Stores PV (and optionally PVC) metadata so the peer cluster can restore bindings after failover/relocate.

---

## Operation Workflows

### 1. Start Data Replication

User enables DR for an application; Ramen and the CSI-addon start replication for the selected PVCs.

```mermaid
sequenceDiagram
    participant User
    participant DRPC as DRPC (Hub)
    participant MW as ManifestWork
    participant VRG_P as VRG (Cluster A)
    participant VR as VolumeReplication
    participant CSI as CSI Replication
    participant Storage as Storage
    participant S3 as S3
    participant VRG_S as VRG (Cluster B)

    User->>DRPC: Create DRPC (placementRef, drPolicyRef, pvcSelector, preferredCluster)
    DRPC->>DRPC: Resolve Placement → Cluster A
    DRPC->>MW: Create ManifestWork: VRG (Primary) for Cluster A
    MW->>VRG_P: Deploy VRG spec (ReplicationState: primary)
    VRG_P->>VRG_P: Reconcile as Primary
    VRG_P->>VR: Create VolumeReplication (state: primary) per PVC
    VR->>CSI: Create replication / set primary
    CSI->>Storage: Enable replication (primary)
    VRG_P->>S3: Upload PV/PVC metadata

    DRPC->>MW: Create ManifestWork: VRG (Secondary) for Cluster B
    MW->>VRG_S: Deploy VRG spec (ReplicationState: secondary)
    VRG_S->>VRG_S: Reconcile as Secondary
    VRG_S->>VR: Create VolumeReplication (state: secondary) per PVC
    VR->>CSI: Set secondary
    CSI->>Storage: Enable replication (secondary)
    Note over Storage: Data replication starts (primary → secondary)
```

**Result:** Primary on Cluster A, Secondary on Cluster B; PV metadata in S3; replication ongoing.

---

### 2. Pause Data Sync

Replication is paused or suspended. In CSI-addon terms this is typically achieved by the **secondary** no longer actively accepting sync (e.g. VR set to secondary and driver or storage pauses sync), or by storage-specific semantics.

```mermaid
sequenceDiagram
    participant User
    participant DRPC as DRPC (Hub)
    participant VRG_P as VRG Primary
    participant VR as VolumeReplication
    participant CSI as CSI / Storage

    Note over User,CSI: Pause = stop ongoing sync (storage/VR semantics)
    User->>DRPC: (Optional) Trigger or annotate pause<br/>or use storage-specific mechanism
    DRPC->>VRG_P: No spec change for "pause" in Ramen today
    Note over VRG_P: Ramen does not expose a dedicated "pause" API.<br/>Pause may be done via storage vendor API or<br/>by moving VRG to Secondary on primary side (stops new writes).
    VRG_P->>VR: Keep Primary or adjust per vendor
    VR->>CSI: Driver may support pause/suspend replication
    CSI->>CSI: Replication paused (no new sync)
```

**Note:** Ramen does not currently expose a dedicated **pause** API. Pause is often implemented by:

- Storage vendor APIs or features (e.g. “suspend replication”), or
- Keeping VR as **Secondary** on the former primary (so no new writes are replicated) until resync.

Your interface can show “Pause” as a storage-level or vendor-specific step; the diagram above reflects that.

---

### 3. Data Re-Sync

Re-sync brings the secondary back in sync with the primary (e.g. after pause or before planned failover). With CSI-addons, **AutoResync** and VR state transitions drive this.

```mermaid
sequenceDiagram
    participant User
    participant VRG_S as VRG (Secondary)
    participant VR as VolumeReplication
    participant CSI as CSI Replication
    participant Storage as Storage

    Note over User,Storage: Re-sync = secondary catches up with primary
    VRG_S->>VRG_S: Reconcile (ReplicationState: secondary)
    VRG_S->>VR: Ensure VR spec: state=secondary, AutoResync=true (e.g. after failover)
    VR->>CSI: Update VR (secondary, AutoResync)
    CSI->>Storage: Trigger resync (secondary ← primary)
    Storage->>Storage: Re-sync in progress
    CSI->>VR: Status: Resyncing / Replicated
    VR->>VRG_S: Status update
    VRG_S->>VRG_S: Update DataProtected / DataReady when resync complete
```

**Ramen behavior:** For **failover** action, VRG sets **AutoResync=true** on the VR when the VR is Secondary (so the new secondary can resync from the new primary). Re-sync is also implicit when both sides are healthy and VR is Primary on one side and Secondary on the other; the driver performs sync/resync.

---

### 4. Cluster-Down Failover (Unplanned)

Primary cluster is lost; user fails over to the surviving (secondary) cluster.

```mermaid
sequenceDiagram
    participant User
    participant DRPC as DRPC (Hub)
    participant MW as ManifestWork
    participant VRG_S as VRG (Surviving Cluster)
    participant VR as VolumeReplication
    participant CSI as CSI
    participant Storage as Storage
    participant S3 as S3
    participant App as Application

    User->>DRPC: Set spec.action=Failover, spec.failoverCluster=SurvivingCluster
    DRPC->>DRPC: Validate and retain placement for failed cluster
    DRPC->>MW: Create or Update ManifestWork VRG Primary on SurvivingCluster
    MW->>VRG_S: Deploy VRG ReplicationState primary Action Failover
    VRG_S->>VRG_S: processAsPrimary()
    VRG_S->>S3: Restore PV/PVC metadata (cluster data)
    S3->>VRG_S: PV/PVC metadata
    VRG_S->>VRG_S: Restore PVs/PVCs from metadata
    VRG_S->>VR: Update VolumeReplication → Primary (AutoResync=true for peer when it returns)
    VR->>CSI: Promote to primary
    CSI->>Storage: Promote volume to primary (RPO: last sync)
    Storage->>VR: Primary
    VRG_S->>VRG_S: Set DataReady / ClusterDataReady
    DRPC->>DRPC: Update PlacementDecision → SurvivingCluster
    DRPC->>App: Workload scheduled to SurvivingCluster (via Placement)
    DRPC->>DRPC: status.phase = FailedOver
```

**Result:** Workload runs on the surviving cluster; volumes are primary there; PV/PVC restored from S3.

---

### 5. Planned Failover (Migration / Relocate)

Planned move to the peer cluster with optional final sync. Ramen calls this **Relocate**.

```mermaid
sequenceDiagram
    participant User
    participant DRPC as DRPC (Hub)
    participant VRG_P as VRG Primary (Source)
    participant VRG_S as VRG Secondary (Target)
    participant VR_P as VR (Source)
    participant VR_S as VR (Target)
    participant CSI as CSI
    participant S3 as S3
    participant MW as ManifestWork

    User->>DRPC: Set spec.action Relocate spec.preferredCluster TargetCluster
    DRPC->>DRPC: setupRelocation ensure VRs secondary on source
    DRPC->>VRG_P: VRG Primary move to Secondary final sync if supported
    VRG_P->>VR_P: PrepareForFinalSync or RunFinalSync or storage final sync
    VR_P->>CSI: Final sync primary to secondary
    CSI->>CSI: Data synced
    VRG_P->>VR_P: Update VR to Secondary
    VRG_P->>S3: Upload latest PV PVC metadata
    DRPC->>MW: Create or Update VRG Primary on TargetCluster
    MW->>VRG_S: Deploy VRG primary Action Relocate
    VRG_S->>S3: Restore PV PVC metadata
    VRG_S->>VRG_S: Restore PVs PVCs and processAsPrimary
    VRG_S->>VR_S: Update VolumeReplication to Primary
    VR_S->>CSI: Promote to primary
    DRPC->>DRPC: Update PlacementDecision to TargetCluster
    DRPC->>DRPC: status phase Relocated
```

**Result:** Workload relocated to target cluster with controlled final sync and promotion.

---

### 6. Failback

Return workload to the original primary cluster after it has recovered. In Ramen this is another **Relocate** (or **Failover** if the current primary is down).

```mermaid
sequenceDiagram
    participant User
    participant DRPC as DRPC (Hub)
    participant VRG_Cur as VRG (Current Primary - was DR target)
    participant VRG_Orig as VRG (Original - recovered cluster)
    participant VR as VolumeReplication
    participant CSI as CSI
    participant S3 as S3
    participant MW as ManifestWork

    Note over User,MW: Failback is Relocate back to original primary
    User->>DRPC: Set spec.action Relocate spec.preferredCluster OriginalPrimaryCluster
    DRPC->>DRPC: Ensure OriginalPrimaryCluster is healthy
    DRPC->>VRG_Cur: Move VRG to Secondary
    VRG_Cur->>VR: VR to Secondary
    VR->>CSI: Demote to secondary replication to original
    CSI->>CSI: Re-sync to original cluster
    VRG_Cur->>S3: Upload PV PVC metadata
    DRPC->>MW: Create or Update VRG Primary on OriginalPrimaryCluster
    MW->>VRG_Orig: Deploy VRG primary
    VRG_Orig->>S3: Restore PV PVC metadata
    VRG_Orig->>VRG_Orig: processAsPrimary and restore PVs PVCs
    VRG_Orig->>VR: VolumeReplication to Primary
    VR->>CSI: Promote on original cluster
    DRPC->>DRPC: Update PlacementDecision to OriginalPrimaryCluster
    DRPC->>DRPC: status phase Relocated failback complete
```

**Result:** Workload and volume primary are back on the original cluster; replication can run original → current (secondary) again.

---

## Combined Workflow Overview (All Six Operations)

```mermaid
stateDiagram-v2
    [*] --> StartReplication: 1. Start data replication
    StartReplication --> Replicating: VRG Primary + Secondary<br/>VR Primary + Secondary

    Replicating --> Paused: 2. Pause data sync<br/>(storage/vendor or VR state)
    Paused --> Replicating: 3. Data re-sync<br/>(AutoResync / VR update)

    Replicating --> ClusterDownFailover: 4. Cluster down failover<br/>action=Failover
    ClusterDownFailover --> FailedOver: VRG/VRA primary on surviving cluster<br/>Restore from S3, promote volumes

    Replicating --> PlannedFailover: 5. Planned failover (migration)<br/>action=Relocate
    PlannedFailover --> Relocated: Final sync, VRG primary on target<br/>Placement updated

    FailedOver --> Failback: 6. Failback<br/>action=Relocate to original
    Relocated --> Failback: 6. Failback<br/>action=Relocate to original
    Failback --> Replicating: VRG primary on original cluster<br/>Replication restarted
```

---

## Graphviz Versions (Alternative Rendering)

### Architecture (Graphviz)

```graphviz
digraph CSIAddonDRArchitecture {
    rankdir=TB;
    node [shape=box, style=rounded];
    compound=true;

    subgraph cluster_hub {
        label="OCM Hub Cluster";
        User; DRPC; DRPolicy; Placement; MW; MCV;
    }
    subgraph cluster_mc_a {
        label="Managed Cluster A (Primary)";
        VRG_A; VR_A; CSI_A; App_A; Storage_A;
    }
    subgraph cluster_mc_b {
        label="Managed Cluster B (Secondary)";
        VRG_B; VR_B; CSI_B; App_B; Storage_B;
    }
    S3 [label="S3 Store"];

    User -> DRPC -> DRPolicy; DRPC -> Placement; DRPC -> MW; DRPC -> MCV;
    MW -> VRG_A [style=dashed]; MW -> VRG_B [style=dashed];
    MCV -> VRG_A [style=dashed]; MCV -> VRG_B [style=dashed];
    VRG_A -> VR_A; VRG_A -> S3; VRG_B -> VR_B; VRG_B -> S3;
    VR_A -> CSI_A; VR_B -> CSI_B;
    CSI_A -> Storage_A; CSI_B -> Storage_B;
    Storage_A -> Storage_B [label="replication", style=dashed];
    App_A -> VR_A; App_B -> VR_B;
}
```

### Operation Flow Summary (Graphviz)

```graphviz
digraph DROperations {
    rankdir=LR;
    node [shape=box, style=rounded];
    1 [label="1. Start\nreplication"];
    2 [label="2. Pause\nsync"];
    3 [label="3. Re-sync"];
    4 [label="4. Cluster-down\nfailover"];
    5 [label="5. Planned\nfailover"];
    6 [label="6. Failback"];
    1 -> 2 -> 3 -> 1;
    1 -> 4; 4 -> 6;
    1 -> 5; 5 -> 6;
    6 -> 1;
}
```

---

## Summary Table

| #   | Operation                        | Ramen / User action | CSI-addon / storage |
| --- | -------------------------------- | ------------------- | ------------------- |
| 1   | **Start data replication**       | Create DRPC (preferredCluster, placementRef, drPolicyRef, pvcSelector). Ramen creates VRG Primary on preferred cluster, VRG Secondary on peer; VRs created Primary/Secondary. | VR created per PVC; replication enabled (primary ↔ secondary). |
| 2   | **Pause data sync**              | No dedicated Ramen API; optional storage/vendor or custom flow. | Storage or driver “pause” / VR state (e.g. secondary only) to stop sync. |
| 3   | **Data re-sync**                 | VRG reconciliation; after failover, AutoResync=true on VR (Secondary). | VR Secondary + AutoResync; driver/storage performs resync. |
| 4   | **Cluster-down failover**        | Set DRPC `action=Failover`, `failoverCluster=surviving`. Ramen deploys VRG Primary on surviving cluster, restores from S3, promotes VR. | VR promoted to primary on surviving cluster; volume promoted. |
| 5   | **Planned failover (migration)** | Set DRPC `action=Relocate`, `preferredCluster=target`. Ramen runs final sync, moves VRG to Secondary on source, Primary on target. | Final sync; VR primary on target, secondary on source. |
| 6   | **Failback**                     | Set DRPC `action=Relocate`, `preferredCluster=original`. Ramen relocates back to original primary. | VR primary on original cluster; replication original → current. |

This gives you the **basic architecture** and **operation workflows** for standard CSI-addon based DR with Ramen, including the six storage operations you listed.
