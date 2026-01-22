# DRPC on Managed Clusters - Detailed Sequence Diagrams

This document contains detailed sequence diagrams for key operations in the ACM-free architecture.

## Failover Operation Sequence

```mermaid
sequenceDiagram
    participant User
    participant DRPC1 as DRPC Controller<br/>(Cluster 1 - Primary)
    participant DRPC2 as DRPC Controller<br/>(Cluster 2 - Secondary)
    participant S3 as S3 Coordination Store
    participant ClusterReg as Cluster Registry
    participant K8sClient as K8s API Client Manager
    participant VRG1 as VRG (Cluster 1)
    participant VRG2 as VRG (Cluster 2)
    participant DRClusterOp2 as DR Cluster Operator<br/>(Cluster 2)

    Note over User,DRClusterOp2: Phase 1: Initiate Failover
    
    User->>DRPC1: Set spec.action = Failover<br/>spec.failoverCluster = cluster-2
    DRPC1->>DRPC1: Validate failoverCluster
    DRPC1->>DRPC1: Check isProtected() condition
    
    DRPC1->>S3: Acquire coordination lock<br/>(DRPC-{name}-lock)
    S3-->>DRPC1: Lock acquired (TTL: 30s)
    
    DRPC1->>ClusterReg: Get cluster information
    ClusterReg-->>DRPC1: Cluster list with kubeconfig secrets
    
    DRPC1->>K8sClient: Get kubeconfig for cluster-2
    K8sClient->>K8sClient: Load kubeconfig from secret
    K8sClient-->>DRPC1: K8s client for cluster-2
    
    Note over User,DRClusterOp2: Phase 2: Read Current State
    
    DRPC1->>K8sClient: Read VRG from cluster-2
    K8sClient->>VRG2: Get VRG (namespace, name)
    VRG2-->>K8sClient: VRG status (Secondary)
    K8sClient-->>DRPC1: VRG status
    
    DRPC1->>K8sClient: Read VRG from cluster-1 (local)
    K8sClient->>VRG1: Get VRG
    VRG1-->>K8sClient: VRG status (Primary)
    K8sClient-->>DRPC1: VRG status
    
    DRPC1->>DRPC1: Validate failover prerequisites
    
    Note over User,DRClusterOp2: Phase 3: Update VRG to Primary
    
    DRPC1->>S3: Write failover state<br/>(DRPC-{name}-state)
    S3-->>DRPC1: State written
    
    DRPC1->>K8sClient: Update VRG2 to Primary<br/>Action = Failover
    K8sClient->>VRG2: Update VRG.Spec<br/>ReplicationState = Primary<br/>Action = Failover
    VRG2-->>K8sClient: VRG updated
    K8sClient-->>DRPC1: Update complete
    
    VRG2->>DRClusterOp2: VRG Reconcile event
    DRClusterOp2->>DRClusterOp2: processAsPrimary()
    DRClusterOp2->>DRClusterOp2: Restore data, promote volumes
    DRClusterOp2->>VRG2: Update VRG status<br/>DataReady = True
    
    Note over User,DRClusterOp2: Phase 4: Verify Readiness
    
    loop Poll until ready
        DRPC1->>K8sClient: Read VRG2 status
        K8sClient->>VRG2: Get VRG
        VRG2-->>K8sClient: VRG status
        K8sClient-->>DRPC1: VRG status
        
        alt VRG not ready
            DRPC1->>DRPC1: Wait and requeue
        end
    end
    
    Note over User,DRClusterOp2: Phase 5: Update Decision and Complete
    
    DRPC1->>S3: Write placement decision<br/>(DRPC-{name}-decision)<br/>cluster = cluster-2
    S3-->>DRPC1: Decision written
    
    DRPC2->>S3: Read placement decision<br/>(polling)
    S3-->>DRPC2: Decision (cluster-2)
    DRPC2->>DRPC2: Update local state
    
    DRPC1->>DRPC1: setDRState(FailedOver)
    DRPC1->>S3: Release lock (optional)
    
    Note over User,DRClusterOp2: Phase 6: Cleanup (Optional)
    
    DRPC1->>K8sClient: Update VRG1 to Secondary<br/>(cleanup old primary)
    K8sClient->>VRG1: Update VRG.Spec<br/>ReplicationState = Secondary
    VRG1-->>K8sClient: VRG updated
```

## Cluster Discovery and Registration

```mermaid
sequenceDiagram
    participant Admin
    participant ClusterReg as Cluster Registry<br/>(CRD)
    participant Secret as Kubeconfig Secret
    participant DRPC1 as DRPC Controller<br/>(Cluster 1)
    participant DRPC2 as DRPC Controller<br/>(Cluster 2)
    participant K8sClient as K8s Client Manager

    Note over Admin,K8sClient: Initial Setup
    
    Admin->>Secret: Create kubeconfig secret<br/>for cluster-2
    Secret-->>Admin: Secret created
    
    Admin->>ClusterReg: Create/Update ClusterRegistry<br/>Add cluster-2 info
    ClusterReg-->>Admin: ClusterRegistry updated
    
    Note over Admin,K8sClient: Discovery and Client Creation
    
    DRPC1->>ClusterReg: List clusters
    ClusterReg-->>DRPC1: Cluster list<br/>(cluster-1, cluster-2)
    
    DRPC1->>ClusterReg: Get cluster-2 details
    ClusterReg-->>DRPC1: Cluster info<br/>(kubeconfigSecret, endpoint)
    
    DRPC1->>K8sClient: Create client for cluster-2
    K8sClient->>Secret: Read kubeconfig secret
    Secret-->>K8sClient: Kubeconfig data
    K8sClient->>K8sClient: Parse kubeconfig<br/>Create client.Client
    K8sClient-->>DRPC1: K8s client ready
    
    Note over Admin,K8sClient: Validation
    
    DRPC1->>K8sClient: Test connection to cluster-2
    K8sClient->>DRPC2: API call (health check)
    DRPC2-->>K8sClient: API response
    K8sClient-->>DRPC1: Connection validated
    
    DRPC1->>ClusterReg: Update cluster-2 status<br/>Status = Available
    ClusterReg-->>DRPC1: Status updated
```

## Leader Election via S3

```mermaid
sequenceDiagram
    participant DRPC1 as DRPC Controller<br/>(Cluster 1)
    participant DRPC2 as DRPC Controller<br/>(Cluster 2)
    participant DRPC3 as DRPC Controller<br/>(Cluster 3)
    participant S3 as S3 Coordination Store

    Note over DRPC1,S3: Initial Leader Election
    
    DRPC1->>S3: Try create lock object<br/>Key: drpc-{name}-lock<br/>If-None-Match: *
    S3-->>DRPC1: Lock created (DRPC1 is leader)
    
    DRPC2->>S3: Try create lock object<br/>Key: drpc-{name}-lock<br/>If-None-Match: *
    S3-->>DRPC2: Error: Already exists (not leader)
    
    DRPC3->>S3: Try create lock object<br/>Key: drpc-{name}-lock<br/>If-None-Match: *
    S3-->>DRPC3: Error: Already exists (not leader)
    
    Note over DRPC1,S3: Lock Renewal (Leader)
    
    loop Every 15 seconds
        DRPC1->>S3: Update lock object<br/>Key: drpc-{name}-lock<br/>If-Match: {etag}<br/>TTL: 30s
        S3-->>DRPC1: Lock renewed
    end
    
    Note over DRPC1,S3: Leader Failure and Re-election
    
    DRPC1->>DRPC1: Controller crash/network failure
    Note over DRPC1: Lock expires after 30s
    
    DRPC2->>S3: Try create lock object<br/>Key: drpc-{name}-lock<br/>If-None-Match: *
    S3-->>DRPC2: Lock created (DRPC2 is new leader)
    
    DRPC3->>S3: Try create lock object<br/>Key: drpc-{name}-lock<br/>If-None-Match: *
    S3-->>DRPC3: Error: Already exists (not leader)
    
    Note over DRPC1,S3: Graceful Leader Release
    
    DRPC2->>DRPC2: Shutdown initiated
    DRPC2->>S3: Delete lock object<br/>Key: drpc-{name}-lock<br/>If-Match: {etag}
    S3-->>DRPC2: Lock deleted
    
    DRPC3->>S3: Try create lock object<br/>Key: drpc-{name}-lock<br/>If-None-Match: *
    S3-->>DRPC3: Lock created (DRPC3 is new leader)
```

## State Coordination via S3

```mermaid
sequenceDiagram
    participant DRPC1 as DRPC Controller<br/>(Cluster 1 - Leader)
    participant DRPC2 as DRPC Controller<br/>(Cluster 2 - Follower)
    participant S3 as S3 Coordination Store

    Note over DRPC1,S3: Write State (Leader)
    
    DRPC1->>DRPC1: State change detected<br/>(e.g., failover initiated)
    DRPC1->>S3: Write state object<br/>Key: drpc-{name}-state<br/>Content: {phase, progression, conditions}
    S3-->>DRPC1: State written (ETag: abc123)
    
    Note over DRPC1,S3: Read State (Follower)
    
    DRPC2->>S3: Read state object<br/>Key: drpc-{name}-state<br/>If-None-Match: xyz789
    alt State changed
        S3-->>DRPC2: State object<br/>ETag: abc123<br/>Content: {new state}
        DRPC2->>DRPC2: Update local DRPC status
        DRPC2->>DRPC2: Reconcile based on new state
    else State unchanged
        S3-->>DRPC2: 304 Not Modified
        DRPC2->>DRPC2: No action needed
    end
    
    Note over DRPC1,S3: Watch State (Alternative)
    
    DRPC2->>S3: Watch state object<br/>Key: drpc-{name}-state<br/>Poll interval: 5s
    loop Polling loop
        DRPC2->>S3: Get state object<br/>If-None-Match: {lastETag}
        alt State changed
            S3-->>DRPC2: New state (ETag: def456)
            DRPC2->>DRPC2: Update and reconcile
        else No change
            S3-->>DRPC2: 304 Not Modified
            DRPC2->>DRPC2: Continue polling
        end
    end
```

## Direct VRG Deployment (Replacing ManifestWork)

```mermaid
sequenceDiagram
    participant DRPC1 as DRPC Controller<br/>(Cluster 1)
    participant K8sClient as K8s API Client
    participant VRG2 as VRG (Cluster 2)
    participant DRClusterOp2 as DR Cluster Operator<br/>(Cluster 2)

    Note over DRPC1,DRClusterOp2: Create New VRG
    
    DRPC1->>DRPC1: Generate VRG spec<br/>(Primary, Failover action)
    DRPC1->>K8sClient: Create VRG in cluster-2<br/>Namespace: ramen-system<br/>Name: {drpc-name}
    
    K8sClient->>VRG2: Create VRG resource
    VRG2-->>K8sClient: VRG created
    
    K8sClient->>K8sClient: Wait for resource creation
    K8sClient-->>DRPC1: VRG created successfully
    
    VRG2->>DRClusterOp2: VRG Reconcile event
    DRClusterOp2->>DRClusterOp2: Process VRG as Primary
    
    Note over DRPC1,DRClusterOp2: Update Existing VRG
    
    DRPC1->>K8sClient: Read current VRG from cluster-2
    K8sClient->>VRG2: Get VRG
    VRG2-->>K8sClient: Current VRG (Secondary)
    K8sClient-->>DRPC1: Current VRG
    
    DRPC1->>DRPC1: Update VRG spec<br/>ReplicationState = Primary
    
    DRPC1->>K8sClient: Update VRG in cluster-2<br/>ResourceVersion: {current}
    K8sClient->>VRG2: Update VRG resource
    VRG2-->>K8sClient: VRG updated (conflict check passed)
    K8sClient-->>DRPC1: VRG updated successfully
    
    alt Conflict (ResourceVersion mismatch)
        VRG2-->>K8sClient: Error: Conflict
        K8sClient-->>DRPC1: Update failed (conflict)
        DRPC1->>K8sClient: Retry: Read and update
    end
```

## Multi-Cluster Coordination During Failover

```mermaid
sequenceDiagram
    participant DRPC1 as DRPC (Cluster 1)<br/>Leader
    participant DRPC2 as DRPC (Cluster 2)<br/>Follower
    participant DRPC3 as DRPC (Cluster 3)<br/>Follower
    participant S3 as S3 Coordination
    participant K8sClient as K8s Client Manager

    Note over DRPC1,K8sClient: Leader Initiates Failover
    
    DRPC1->>S3: Acquire lock (already leader)
    DRPC1->>S3: Write state: FailingOver
    S3->>DRPC2: State change notification (if watching)
    S3->>DRPC3: State change notification (if watching)
    
    DRPC1->>K8sClient: Read VRG from all clusters
    K8sClient->>DRPC1: VRG status from all clusters
    
    DRPC1->>DRPC1: Determine failover target<br/>(cluster-2)
    
    DRPC1->>K8sClient: Update VRG2 to Primary
    K8sClient-->>DRPC1: VRG updated
    
    DRPC1->>S3: Write decision: cluster-2
    S3->>DRPC2: Decision notification
    S3->>DRPC3: Decision notification
    
    Note over DRPC1,K8sClient: Followers React to Decision
    
    DRPC2->>S3: Read decision
    S3-->>DRPC2: Decision: cluster-2 (this cluster)
    DRPC2->>DRPC2: Update local state<br/>Prepare for workload
    
    DRPC3->>S3: Read decision
    S3-->>DRPC3: Decision: cluster-2 (not this cluster)
    DRPC3->>DRPC3: Update local state<br/>Standby mode
    
    Note over DRPC1,K8sClient: Leader Monitors Progress
    
    loop Until ready
        DRPC1->>K8sClient: Read VRG2 status
        K8sClient->>DRPC1: VRG2 status
        alt Not ready
            DRPC1->>DRPC1: Wait and requeue
        end
    end
    
    DRPC1->>S3: Write state: FailedOver
    S3->>DRPC2: State notification
    S3->>DRPC3: State notification
```

## Error Handling and Recovery

```mermaid
sequenceDiagram
    participant DRPC1 as DRPC (Cluster 1)
    participant DRPC2 as DRPC (Cluster 2)
    participant S3 as S3 Coordination
    participant K8sClient as K8s Client
    participant VRG2 as VRG (Cluster 2)

    Note over DRPC1,VRG2: Network Failure Scenario
    
    DRPC1->>K8sClient: Update VRG2 to Primary
    K8sClient->>VRG2: Update request
    VRG2-->>K8sClient: Network timeout
    
    K8sClient-->>DRPC1: Error: Connection failed
    DRPC1->>DRPC1: Retry with backoff
    
    Note over DRPC1,VRG2: Lock Expiration Scenario
    
    DRPC1->>S3: Renew lock
    S3-->>DRPC1: Network timeout
    
    Note over DRPC1: Lock expires after 30s
    
    DRPC2->>S3: Acquire lock
    S3-->>DRPC2: Lock acquired (new leader)
    
    DRPC2->>S3: Read state
    S3-->>DRPC2: Last known state
    
    DRPC2->>DRPC2: Resume coordination<br/>from last known state
    
    Note over DRPC1,VRG2: Conflict Resolution
    
    DRPC1->>K8sClient: Update VRG2
    K8sClient->>VRG2: Update with ResourceVersion
    VRG2-->>K8sClient: Error: Conflict<br/>(ResourceVersion mismatch)
    
    K8sClient-->>DRPC1: Update failed (conflict)
    DRPC1->>K8sClient: Read current VRG2
    K8sClient->>VRG2: Get VRG
    VRG2-->>K8sClient: Current VRG
    K8sClient-->>DRPC1: Current VRG
    
    DRPC1->>DRPC1: Merge changes<br/>with current state
    DRPC1->>K8sClient: Retry update<br/>with new ResourceVersion
    K8sClient->>VRG2: Update VRG
    VRG2-->>K8sClient: VRG updated
    K8sClient-->>DRPC1: Update successful
```
