# Failover Operations Sequence Diagrams

This document contains sequence diagrams for failover operations in Ramen, with special focus on VolumeReplicationGroup (VRG) objects.

## Overview

Failover operations in Ramen involve:

1. **DRPlacementControl** orchestrating the failover
2. **VolumeReplicationGroup (VRG)** transitioning from Secondary to Primary
3. **ManifestWork** deploying VRG to managed clusters
4. **ManagedClusterView** reading VRG status
5. **Placement/PlacementDecision** controlling cluster selection
6. **DR Cluster Operator** processing VRG on managed clusters

## Main Failover Sequence

```mermaid
sequenceDiagram
    participant User
    participant DRPC as DRPlacementControl<br/>(Hub)
    participant Placement as Placement API<br/>(OCM)
    participant MW as ManifestWork<br/>(OCM)
    participant MCV as ManagedClusterView<br/>(OCM)
    participant VRG_Hub as VRG (Hub View)
    participant VRG_MC as VRG (Managed Cluster)
    participant DRClusterOp as DR Cluster Operator
    participant Storage as Storage Backend
    participant App as Application Workload

    User->>DRPC: Set spec.action = Failover<br/>spec.failoverCluster = target-cluster
    DRPC->>DRPC: Validate failoverCluster
    DRPC->>DRPC: Check isProtected() condition
    
    alt Workload not protected
        DRPC-->>User: Error: Cannot failover<br/>workload not protected
    end
    
    DRPC->>DRPC: setDRState(FailingOver)
    DRPC->>Placement: Get current cluster decision
    Placement-->>DRPC: Current cluster (primary-cluster)
    
    DRPC->>DRPC: checkFailoverPrerequisites(primary-cluster)
    Note over DRPC: Check Metro/Regional prerequisites<br/>Check MaintenanceMode activations
    
    DRPC->>Placement: retainClusterDecisionAsFailover(primary-cluster)
    Note over Placement: Mark primary cluster decision<br/>as retained for failover
    
    DRPC->>DRPC: switchToCluster(failover-cluster)
    DRPC->>DRPC: createVRGManifestWorkAsPrimary(failover-cluster)
    
    alt VRG exists but not Primary
        DRPC->>MW: Update VRG to Primary state
        MW->>VRG_MC: Update VRG.Spec.ReplicationState = Primary
    else VRG does not exist
        DRPC->>DRPC: newVRG(failover-cluster, Primary, VRGActionFailover)
        DRPC->>MW: Create ManifestWork with VRG (Primary)
        MW->>VRG_MC: Deploy VRG to managed cluster
    end
    
    VRG_MC->>DRClusterOp: VRG Reconcile (Primary, Failover)
    DRClusterOp->>DRClusterOp: processAsPrimary()
    
    Note over DRClusterOp: Restore cluster data (PVs/PVCs)<br/>from S3 if needed
    
    DRClusterOp->>Storage: Promote volumes to Primary<br/>(VolumeReplication)
    Storage-->>DRClusterOp: Volumes promoted
    
    DRClusterOp->>DRClusterOp: Restore PVCs from S3 metadata
    DRClusterOp->>DRClusterOp: Restore kube objects (if enabled)
    DRClusterOp->>VRG_MC: Update VRG status conditions
    
    DRPC->>MCV: Read VRG status from failover-cluster
    MCV->>VRG_MC: Query VRG
    VRG_MC-->>MCV: VRG status (DataReady, etc.)
    MCV-->>DRPC: VRG status
    
    DRPC->>DRPC: checkReadiness(failover-cluster)
    
    alt VRG not ready
        DRPC->>DRPC: Wait and requeue
    end
    
    DRPC->>Placement: Update PlacementDecision<br/>to failover-cluster
    Placement->>App: Relocate workload to failover-cluster
    
    DRPC->>DRPC: updatePreferredDecision()
    DRPC->>DRPC: setDRState(FailedOver)
    
    Note over DRPC: Failover complete<br/>Cleanup old primary cluster
```

## Detailed VRG Failover Sequence

```mermaid
sequenceDiagram
    participant DRPC as DRPlacementControl<br/>(Hub Operator)
    participant MW as ManifestWork<br/>(OCM)
    participant MCV as ManagedClusterView<br/>(OCM)
    participant VRG_MW as VRG in ManifestWork<br/>(Hub)
    participant VRG_MC as VRG on Managed Cluster
    participant VRGController as VRG Controller<br/>(DR Cluster Operator)
    participant VolRep as VolumeReplication<br/>(CSI Addons)
    participant VolSync as VolSync<br/>(if async)
    participant S3 as S3 Store
    participant PVC as PersistentVolumeClaim
    participant PV as PersistentVolume

    Note over DRPC,PV: Phase 1: Initiate Failover
    
    DRPC->>DRPC: RunFailover()
    DRPC->>DRPC: isValidFailoverTarget(failover-cluster)
    DRPC->>MCV: Get VRG from failover-cluster
    MCV->>VRG_MC: Query VRG status
    VRG_MC-->>MCV: VRG (Secondary or not found)
    MCV-->>DRPC: VRG status
    
    DRPC->>DRPC: isProtected()
    Note over DRPC: Check ConditionProtected = True
    
    DRPC->>DRPC: switchToFailoverCluster()
    DRPC->>DRPC: checkFailoverPrerequisites()
    
    Note over DRPC,PV: Phase 2: Create/Update VRG as Primary
    
    DRPC->>DRPC: switchToCluster(failover-cluster)
    DRPC->>DRPC: createVRGManifestWorkAsPrimary()
    DRPC->>MW: Find ManifestWork (VRG type)
    
    alt VRG ManifestWork exists
        MW-->>DRPC: ManifestWork found
        DRPC->>MW: Extract VRG from ManifestWork
        MW-->>DRPC: VRG (Secondary state)
        DRPC->>DRPC: updateVRGState(Primary)
        DRPC->>MW: Update ManifestWork with VRG (Primary)
        MW->>VRG_MW: Update VRG.Spec.ReplicationState = Primary<br/>VRG.Spec.Action = Failover
    else VRG ManifestWork not found
        DRPC->>DRPC: newVRG(failover-cluster, Primary, Failover)
        Note over DRPC: Create VRG with:<br/>- ReplicationState: Primary<br/>- Action: Failover<br/>- Sync/Async spec from DRPolicy
        DRPC->>MW: Create ManifestWork with VRG
        MW->>VRG_MW: Create VRG in ManifestWork
    end
    
    MW->>VRG_MC: Deploy VRG to managed cluster
    VRG_MC->>VRGController: VRG Reconcile Event
    
    Note over DRPC,PV: Phase 3: VRG Processing as Primary (Failover)
    
    VRGController->>VRGController: processAsPrimary()
    
    alt Cluster data needs restore
        VRGController->>VRGController: shouldRestoreClusterData()
        VRGController->>S3: Read PV/PVC metadata
        S3-->>VRGController: PV/PVC metadata
        VRGController->>PV: Restore PV from metadata
        VRGController->>PVC: Restore PVC from metadata
        VRGController->>VRG_MC: Update VRG status
    end
    
    VRGController->>VRGController: reconcileAsPrimary()
    
    alt Sync DR (Metro)
        VRGController->>VolRep: Promote VolumeReplication<br/>to Primary
        VolRep->>Storage: Promote replication relationship
        Storage-->>VolRep: Primary established
        VolRep-->>VRGController: VolumeReplication ready
    else Async DR (Regional)
        VRGController->>VolSync: Ensure ReplicationDestination<br/>promoted (if failover action)
        VolSync->>Storage: Restore from snapshot
        Storage-->>VolSync: Data restored
        VolSync-->>VRGController: ReplicationDestination ready
    end
    
    VRGController->>VRGController: updateVRGDataReadyCondition()
    VRGController->>VRG_MC: Update VRG.Status.Conditions<br/>DataReady = True
    
    alt Kube objects protection enabled
        VRGController->>VRGController: shouldRestoreKubeObjects()
        VRGController->>S3: Read kube objects from Velero backup
        S3-->>VRGController: Kube objects
        VRGController->>VRGController: kubeObjectsRecover()
        VRGController->>VRG_MC: Update VRG.Status.Conditions<br/>KubeObjectsReady = True
    end
    
    VRGController->>VRG_MC: Update VRG final status
    
    Note over DRPC,PV: Phase 4: Verify Readiness
    
    DRPC->>MCV: Read VRG from failover-cluster
    MCV->>VRG_MC: Query VRG
    VRG_MC-->>MCV: VRG with DataReady = True
    MCV-->>DRPC: VRG status
    
    DRPC->>DRPC: checkReadiness(failover-cluster)
    Note over DRPC: Verify:<br/>- VRG DataReady condition<br/>- VolSync setup (if async)
    
    alt VRG ready
        DRPC->>Placement: Update PlacementDecision
        DRPC->>DRPC: ensureFailoverActionCompleted()
        DRPC->>DRPC: setDRState(FailedOver)
    else VRG not ready
        DRPC->>DRPC: Wait and requeue
    end
```

## VolumeReplicationGroup State Transitions During Failover

```mermaid
stateDiagram-v2
    [*] --> Secondary: Initial Protection
    Secondary --> ValidatingFailover: User sets<br/>DRPC.action = Failover
    
    ValidatingFailover --> CheckingPrerequisites: Validate<br/>failoverCluster
    
    CheckingPrerequisites --> CreatingPrimaryVRG: Prerequisites met<br/>(MaintenanceMode, etc.)
    CheckingPrerequisites --> Waiting: Prerequisites not met
    
    CreatingPrimaryVRG --> DeployingVRG: Create/Update<br/>VRG ManifestWork
    DeployingVRG --> VRGDeployed: ManifestWork<br/>applied
    
    VRGDeployed --> ProcessingAsPrimary: VRG Controller<br/>reconciles
    ProcessingAsPrimary --> RestoringData: Restore PVs/PVCs<br/>from S3 (if needed)
    
    RestoringData --> PromotingVolumes: Data restored
    PromotingVolumes --> VolumesPromoted: VolumeReplication<br/>promoted to Primary
    
    VolumesPromoted --> RestoringKubeObjects: Restore kube objects<br/>(if enabled)
    RestoringKubeObjects --> KubeObjectsRestored: Kube objects restored
    
    KubeObjectsRestored --> DataReady: All conditions met
    DataReady --> UpdatingPlacement: VRG DataReady = True
    
    UpdatingPlacement --> FailedOver: PlacementDecision<br/>updated
    FailedOver --> [*]: Failover complete
    
    Waiting --> CheckingPrerequisites: Retry
    ProcessingAsPrimary --> WaitingForReadiness: Not ready yet
    WaitingForReadiness --> ProcessingAsPrimary: Retry
```

## Failover with VolumeGroupReplication (CephFS Consistency Groups)

```mermaid
sequenceDiagram
    participant DRPC as DRPlacementControl
    participant MW as ManifestWork
    participant VRG as VolumeReplicationGroup
    participant VRGController as VRG Controller
    participant RGS as ReplicationGroupSource<br/>(CephFS CG)
    participant RGD as ReplicationGroupDestination<br/>(CephFS CG)
    participant CephFS as CephFS Storage

    Note over DRPC,CephFS: Failover with VolumeGroupReplication<br/>(CephFS Consistency Groups)

    DRPC->>MW: Create/Update VRG (Primary, Failover)
    MW->>VRG: Deploy VRG to managed cluster
    VRG->>VRGController: Reconcile event
    
    VRGController->>VRGController: processAsPrimary()
    VRGController->>VRGController: reconcileAsPrimary()
    
    alt VolumeGroupReplication enabled
        VRGController->>RGD: Ensure ReplicationGroupDestination<br/>exists and ready
        RGD->>CephFS: Promote consistency group<br/>to Primary
        CephFS-->>RGD: CG promoted
        RGD-->>VRGController: RGD ready
        
        VRGController->>VRGController: Ensure PVCs from RGD
        Note over VRGController: Create PVCs from<br/>ReplicationGroupDestination
    end
    
    VRGController->>VRG: Update VRG status<br/>DataReady = True
    VRG-->>DRPC: VRG ready via MCV
    DRPC->>DRPC: Failover complete
```

## Failover Prerequisites Check Sequence

```mermaid
sequenceDiagram
    participant DRPC as DRPlacementControl
    participant MCV as ManagedClusterView
    participant VRG as VolumeReplicationGroup
    participant DRCluster as DRCluster
    participant MMode as MaintenanceMode
    participant Storage as Storage Backend

    DRPC->>DRPC: checkFailoverPrerequisites(curHomeCluster)
    
    alt Metro DR (Sync)
        DRPC->>DRPC: checkMetroFailoverPrerequisites()
        DRPC->>MCV: Read VRG from failover-cluster
        MCV->>VRG: Query VRG
        VRG-->>MCV: VRG with protected PVCs
        MCV-->>DRPC: VRG status
        
        DRPC->>DRPC: Check protected PVCs<br/>for MaintenanceMode requirements
        
        loop For each protected PVC
            DRPC->>DRCluster: Check MaintenanceMode<br/>for storage identifier
            DRCluster->>MMode: Query MaintenanceMode
            MMode->>Storage: Check FailoverActivated condition
            Storage-->>MMode: Activation status
            MMode-->>DRCluster: MaintenanceMode status
            DRCluster-->>DRPC: MaintenanceMode activated
        end
    else Regional DR (Async)
        DRPC->>DRPC: checkRegionalFailoverPrerequisites()
        DRPC->>DRCluster: Get DRCluster for failover-cluster
        DRCluster-->>DRPC: DRCluster config
        
        DRPC->>DRPC: requiresRegionalFailoverPrerequisites()
        Note over DRPC: Check if any protected PVCs<br/>require MaintenanceMode
        
        alt MaintenanceMode required
            DRPC->>DRCluster: Check MaintenanceMode activations
            DRCluster->>MMode: Query MaintenanceMode
            MMode->>Storage: Check FailoverActivated
            Storage-->>MMode: Activation status
            MMode-->>DRCluster: Status
            DRCluster-->>DRPC: All activations met
        end
    end
    
    alt Prerequisites met
        DRPC->>DRPC: Return true
    else Prerequisites not met
        DRPC->>DRPC: Return false, wait
    end
```

## Error Handling During Failover

```mermaid
sequenceDiagram
    participant DRPC as DRPlacementControl
    participant VRG as VolumeReplicationGroup
    participant MCV as ManagedClusterView
    participant MW as ManifestWork

    Note over DRPC,MW: Error Scenarios During Failover

    alt Invalid failover target
        DRPC->>DRPC: isValidFailoverTarget()
        DRPC->>MCV: Read VRG from target cluster
        MCV-->>DRPC: VRG not found or invalid state
        DRPC->>DRPC: Set condition: Available = False
        DRPC-->>User: Error: Invalid failover target
    end
    
    alt Workload not protected
        DRPC->>DRPC: isProtected()
        DRPC->>DRPC: ConditionProtected != True
        DRPC->>DRPC: Set condition: Available = False
        DRPC-->>User: Error: Cannot failover,<br/>workload not protected
    end
    
    alt Prerequisites not met
        DRPC->>DRPC: checkFailoverPrerequisites()
        DRPC->>DRPC: MaintenanceMode not activated
        DRPC->>DRPC: Set progression:<br/>CheckingFailoverPrerequisites
        DRPC->>DRPC: Requeue and retry
    end
    
    alt VRG deployment failure
        DRPC->>MW: Create/Update VRG ManifestWork
        MW-->>DRPC: Error deploying VRG
        DRPC->>DRPC: Set progression:<br/>FailingOverToCluster
        DRPC->>DRPC: Requeue and retry
    end
    
    alt VRG not ready
        DRPC->>MCV: Read VRG status
        MCV->>VRG: Query VRG
        VRG-->>MCV: VRG DataReady = False
        MCV-->>DRPC: VRG not ready
        DRPC->>DRPC: Set progression:<br/>WaitingForReadiness
        DRPC->>DRPC: Requeue and retry
    end
    
    alt VRG processing error
        VRG->>VRG: processAsPrimary() fails
        VRG->>VRG: Set error conditions
        VRG-->>DRPC: VRG status with errors
        DRPC->>DRPC: Detect error in checkReadiness()
        DRPC->>DRPC: Requeue and retry
    end
```
