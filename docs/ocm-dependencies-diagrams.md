# OCM/RHACM Dependencies - Diagrams

This document contains visual diagrams showing Ramen's dependencies on OCM/RHACM.

## Architecture Overview

```mermaid
graph TB
    subgraph "OCM Hub Cluster"
        HubOperator[Ramen Hub Operator]
        DRPC[DRPlacementControl]
        DRPolicy[DRPolicy]
        DRCluster[DRCluster]
        Placement[Placement/PlacementRule]
        PlacementDecision[PlacementDecision]
        MW[ManifestWork]
        MCV[ManagedClusterView]
        MC[ManagedCluster]
    end
    
    subgraph "OCM Managed Cluster 1"
        DRClusterOp1[DR Cluster Operator]
        VRG1[VolumeReplicationGroup]
        App1[Application Workload]
    end
    
    subgraph "OCM Managed Cluster 2"
        DRClusterOp2[DR Cluster Operator]
        VRG2[VolumeReplicationGroup]
        App2[Application Workload]
    end
    
    HubOperator --> DRPC
    HubOperator --> DRPolicy
    HubOperator --> DRCluster
    DRPC --> Placement
    DRPC --> PlacementDecision
    DRPC --> MW
    DRPC --> MCV
    DRPC --> MC
    MW -.deploys.-> VRG1
    MW -.deploys.-> VRG2
    MCV -.reads.-> VRG1
    MCV -.reads.-> VRG2
    MC -.discovery.-> DRClusterOp1
    MC -.discovery.-> DRClusterOp2
    VRG1 --> App1
    VRG2 --> App2
    
    style HubOperator fill:#e1f5ff
    style DRPC fill:#fff4e1
    style MW fill:#ffcccc
    style MCV fill:#ffcccc
    style Placement fill:#ffcccc
    style MC fill:#ffcccc
```

## OCM API Dependency Graph

```mermaid
graph LR
    subgraph "Ramen Components"
        DRPC[DRPlacementControl<br/>Controller]
        DRPolicy[DRPolicy<br/>Controller]
        DRCluster[DRCluster<br/>Controller]
        MWUtil[ManifestWork<br/>Utility]
        MCVUtil[ManagedClusterView<br/>Utility]
        MCUtil[ManagedCluster<br/>Utility]
    end
    
    subgraph "OCM Core APIs"
        MC[ManagedCluster<br/>cluster.open-cluster-management.io]
        Placement[Placement<br/>cluster.open-cluster-management.io]
        PD[PlacementDecision<br/>cluster.open-cluster-management.io]
        MW[ManifestWork<br/>work.open-cluster-management.io]
        MCV[ManagedClusterView<br/>view.open-cluster-management.io]
        PR[PlacementRule<br/>apps.open-cluster-management.io]
    end
    
    DRPC --> Placement
    DRPC --> PD
    DRPC --> MW
    DRPC --> MCV
    DRPC --> PR
    DRPC --> MC
    DRPolicy --> MC
    DRPolicy --> PR
    DRCluster --> MC
    DRCluster --> MW
    MWUtil --> MW
    MCVUtil --> MCV
    MCUtil --> MC
    
    style MC fill:#ff9999
    style Placement fill:#ff9999
    style PD fill:#ff9999
    style MW fill:#ff9999
    style MCV fill:#ff9999
    style PR fill:#ffcccc
```

## Resource Flow Diagram

```mermaid
sequenceDiagram
    participant User
    participant DRPC as DRPlacementControl
    participant Placement as Placement API
    participant MW as ManifestWork
    participant MCV as ManagedClusterView
    participant MC as ManagedCluster
    participant VRG as VolumeReplicationGroup
    participant App as Application
    
    User->>DRPC: Create DRPlacementControl
    DRPC->>Placement: Read Placement/PlacementRule
    Placement-->>DRPC: Target cluster decision
    DRPC->>MC: Validate cluster exists
    MC-->>DRPC: Cluster status
    DRPC->>MW: Create ManifestWork (VRG)
    MW->>VRG: Deploy VRG to managed cluster
    VRG->>App: Protect application
    DRPC->>MCV: Read VRG status
    MCV->>VRG: Query VRG
    VRG-->>MCV: VRG status
    MCV-->>DRPC: VRG status
    DRPC->>Placement: Update PlacementDecision
```

## Component Dependency Matrix

```mermaid
graph TB
    subgraph "Hub Operator Components"
        A[DRPlacementControl Controller]
        B[DRPolicy Controller]
        C[DRCluster Controller]
    end
    
    subgraph "OCM APIs Used"
        D[ManagedCluster]
        E[Placement]
        F[PlacementDecision]
        G[ManifestWork]
        H[ManagedClusterView]
        I[PlacementRule]
    end
    
    A -->|Reads/Updates| D
    A -->|Reads| E
    A -->|Creates/Updates| F
    A -->|Creates/Updates| G
    A -->|Creates/Reads| H
    A -->|Reads| I
    
    B -->|Reads| D
    B -->|Creates| I
    
    C -->|Reads| D
    C -->|Creates/Updates| G
    
    style A fill:#e1f5ff
    style B fill:#e1f5ff
    style C fill:#e1f5ff
    style D fill:#ffcccc
    style E fill:#ffcccc
    style F fill:#ffcccc
    style G fill:#ffcccc
    style H fill:#ffcccc
    style I fill:#ffcccc
```

## ManifestWork Usage

```mermaid
graph LR
    subgraph "Hub Cluster"
        DRPC[DRPlacementControl]
        DRCluster[DRCluster]
    end
    
    subgraph "ManifestWork Types"
        VRGMW[VRG ManifestWork]
        MMMW[MaintenanceMode ManifestWork]
        NFMW[NetworkFence ManifestWork]
        NSMW[Namespace ManifestWork]
        DRCConfigMW[DRClusterConfig ManifestWork]
        RBACMW[RBAC ManifestWork]
    end
    
    subgraph "Managed Cluster Resources"
        VRG[VolumeReplicationGroup]
        MM[MaintenanceMode]
        NF[NetworkFence]
        NS[Namespace]
        DRCConfig[DRClusterConfig]
        RBAC[ClusterRoles]
    end
    
    DRPC --> VRGMW
    DRPC --> MMMW
    DRPC --> NFMW
    DRPC --> NSMW
    DRCluster --> DRCConfigMW
    DRCluster --> RBACMW
    
    VRGMW --> VRG
    MMMW --> MM
    NFMW --> NF
    NSMW --> NS
    DRCConfigMW --> DRCConfig
    RBACMW --> RBAC
    
    style DRPC fill:#e1f5ff
    style DRCluster fill:#e1f5ff
    style VRGMW fill:#ffcccc
    style MMMW fill:#ffcccc
    style NFMW fill:#ffcccc
    style NSMW fill:#ffcccc
    style DRCConfigMW fill:#ffcccc
    style RBACMW fill:#ffcccc
```

## ManagedClusterView Usage

```mermaid
graph LR
    subgraph "Hub Cluster"
        DRPC[DRPlacementControl]
        DRCluster[DRCluster]
    end
    
    subgraph "ManagedClusterView Types"
        VRGMCV[VRG MCV]
        MMMCV[MaintenanceMode MCV]
        NFMCV[NetworkFence MCV]
        SCMCV[StorageClass MCV]
        VSCMCV[VolumeSnapshotClass MCV]
        VRCMCV[VolumeReplicationClass MCV]
        DRCConfigMCV[DRClusterConfig MCV]
    end
    
    subgraph "Managed Cluster Resources"
        VRG[VolumeReplicationGroup]
        MM[MaintenanceMode]
        NF[NetworkFence]
        SC[StorageClass]
        VSC[VolumeSnapshotClass]
        VRC[VolumeReplicationClass]
        DRCConfig[DRClusterConfig]
    end
    
    DRPC --> VRGMCV
    DRPC --> MMMCV
    DRPC --> NFMCV
    DRPC --> SCMCV
    DRPC --> VSCMCV
    DRPC --> VRCMCV
    DRCluster --> DRCConfigMCV
    
    VRGMCV -.reads.-> VRG
    MMMCV -.reads.-> MM
    NFMCV -.reads.-> NF
    SCMCV -.reads.-> SC
    VSCMCV -.reads.-> VSC
    VRCMCV -.reads.-> VRC
    DRCConfigMCV -.reads.-> DRCConfig
    
    style DRPC fill:#e1f5ff
    style DRCluster fill:#e1f5ff
    style VRGMCV fill:#ffcccc
    style MMMCV fill:#ffcccc
    style NFMCV fill:#ffcccc
    style SCMCV fill:#ffcccc
    style VSCMCV fill:#ffcccc
    style VRCMCV fill:#ffcccc
    style DRCConfigMCV fill:#ffcccc
```

## Dependency Isolation Analysis

```mermaid
graph TB
    subgraph "Critical Dependencies"
        MW[ManifestWork<br/>🔴 Cannot Remove]
        MCV[ManagedClusterView<br/>🔴 Cannot Remove]
        Placement[Placement API<br/>🔴 Cannot Remove]
        MC[ManagedCluster<br/>🔴 Cannot Remove]
    end
    
    subgraph "Replaceable Dependencies"
        PR[PlacementRule<br/>🟡 Can Remove<br/>Legacy Support]
        Policy[Policy APIs<br/>🟡 Can Remove<br/>Secret Propagation]
    end
    
    subgraph "Alternative Approaches"
        DirectAPI[Direct K8s API<br/>Requires kubeconfigs]
        CustomPlacement[Custom Placement Logic<br/>Lose OCM features]
        CustomRegistry[Custom Cluster Registry<br/>Lose OCM integration]
    end
    
    MW -.could replace with.-> DirectAPI
    MCV -.could replace with.-> DirectAPI
    Placement -.could replace with.-> CustomPlacement
    MC -.could replace with.-> CustomRegistry
    PR -.remove.-> Placement
    Policy -.remove.-> DirectAPI
    
    style MW fill:#ff9999
    style MCV fill:#ff9999
    style Placement fill:#ff9999
    style MC fill:#ff9999
    style PR fill:#ffe699
    style Policy fill:#ffe699
```

## Code Organization

```mermaid
graph TB
    subgraph "Ramen Codebase"
        A[internal/controller/drplacementcontrol.go<br/>DRPlacementControl Logic]
        B[internal/controller/drplacementcontrol_controller.go<br/>DRPlacementControl Controller]
        C[internal/controller/util/mw_util.go<br/>ManifestWork Utilities]
        D[internal/controller/util/mcv_util.go<br/>ManagedClusterView Utilities]
        E[internal/controller/util/managedcluster.go<br/>ManagedCluster Utilities]
        F[internal/controller/drpolicy_controller.go<br/>DRPolicy Controller]
        G[internal/controller/drcluster_controller.go<br/>DRCluster Controller]
    end
    
    subgraph "OCM API Usage"
        H[ManifestWork API]
        I[ManagedClusterView API]
        J[Placement API]
        K[ManagedCluster API]
        L[PlacementRule API]
    end
    
    A --> C
    A --> D
    A --> E
    B --> J
    B --> L
    C --> H
    D --> I
    E --> K
    F --> K
    F --> L
    G --> K
    G --> H
    
    style A fill:#e1f5ff
    style B fill:#e1f5ff
    style C fill:#fff4e1
    style D fill:#fff4e1
    style E fill:#fff4e1
    style F fill:#e1f5ff
    style G fill:#e1f5ff
    style H fill:#ffcccc
    style I fill:#ffcccc
    style J fill:#ffcccc
    style K fill:#ffcccc
    style L fill:#ffcccc
```
