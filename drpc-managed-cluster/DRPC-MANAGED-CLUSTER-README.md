# DRPC on Managed Clusters - ACM Elimination Strawman

This directory contains a strawman proposal for moving DRPlacementControl (DRPC) to managed clusters and eliminating all ACM (Open Cluster Management) dependencies.

## Documents

### 1. [drpc-managed-cluster-strawman.md](./drpc-managed-cluster-strawman.md)

**Main proposal document** containing:

- Executive summary
- Current architecture (with ACM)
- Proposed architecture (without ACM)
- Key design decisions
- Component designs
- Migration path
- Benefits and challenges
- Open questions

### 2. [drpc-managed-cluster-strawman-sequences.md](./drpc-managed-cluster-strawman-sequences.md)

**Detailed sequence diagrams** showing:

- Failover operation sequence
- Cluster discovery and registration
- Leader election via S3
- State coordination via S3
- Direct VRG deployment (replacing ManifestWork)
- Multi-cluster coordination
- Error handling and recovery

### 3. [drpc-managed-cluster-strawman-graphviz.dot](./drpc-managed-cluster-strawman-graphviz.dot)

**Graphviz diagrams** for detailed visualization. Generate images with:

```bash
# Generate SVG (recommended for web)
dot -Tsvg drpc-managed-cluster-strawman-graphviz.dot -o drpc-managed-cluster-strawman-graphviz.svg

# Generate PNG (for documents)
dot -Tpng drpc-managed-cluster-strawman-graphviz.dot -o drpc-managed-cluster-strawman-graphviz.png

# Generate PDF (for printing)
dot -Tpdf drpc-managed-cluster-strawman-graphviz.dot -o drpc-managed-cluster-strawman-graphviz.pdf
```

## Key Changes

### Architecture Shift

**From**: Hub-based with ACM APIs

**To**: Managed cluster-based with direct K8s API access

### Component Replacements

| Current (ACM) | Proposed (Direct) |
| :------------- | :---------------- |
| ManifestWork | Direct K8s API Client |
| ManagedClusterView | Direct K8s API Client |
| Placement/PlacementDecision | Custom Placement Logic + S3 |
| ManagedCluster | ClusterRegistry (CRD/ConfigMap) |
| Hub Operator | Managed Cluster Operator |

### New Components

1. **ClusterRegistry**: Custom resource for cluster discovery
2. **K8s Client Manager**: Manages kubeconfigs and API clients
3. **S3 Coordinator**: Handles state coordination and leader election
4. **Direct Resource Deployer**: Deploys resources via direct API
5. **Direct Resource Reader**: Reads resources via direct API

## Design Decisions

### 1. DRPC Deployment Model

**Chosen**: Active-Active

- DRPC runs on all managed clusters
- Leader election determines active coordinator
- All DRPCs can read local VRG status
- Only leader coordinates cross-cluster operations

**Benefits**: Resilience, no single point of failure

### 2. Coordination Mechanism

**Chosen**: S3-Based Coordination

- Use S3 as coordination store
- Store DRPC state, decisions, locks
- Leader election via S3 atomic operations

**Benefits**: Reliable, already used by Ramen, no additional infrastructure

### 3. Cluster Discovery

**Chosen**: Custom ClusterRegistry

- ConfigMap or CRD for cluster information
- Manual configuration or auto-discovery
- Stores kubeconfig secrets, endpoints, status

**Benefits**: Simple, flexible, no external dependency

## Migration Path

### Phase 1: Add Abstraction Layers

- Create interfaces for cluster management, resource deployment, resource reading
- Implement both ACM and direct K8s API versions
- Add feature flag to switch between implementations
- **Risk**: Low
- **Effort**: Medium

### Phase 2: Implement Direct K8s Components

- Implement ClusterRegistry CRD
- Implement DirectResourceDeployer
- Implement DirectResourceReader
- Implement S3-based coordination
- Test in parallel with ACM implementation
- **Risk**: Medium
- **Effort**: High

### Phase 3: Move DRPC to Managed Clusters

- Update DRPC to be namespaced (if not already)
- Deploy DRPC controller to managed clusters
- Remove hub operator deployment
- Update RBAC for managed cluster access
- **Risk**: High
- **Effort**: High

### Phase 4: Remove ACM Dependencies

- Remove ManifestWork usage
- Remove ManagedClusterView usage
- Remove Placement/PlacementDecision usage
- Remove ManagedCluster usage
- Remove ACM imports from go.mod
- **Risk**: Medium
- **Effort**: Medium

## Benefits

1. **No ACM Dependency**: Eliminates all OCM/RHACM dependencies
2. **Simpler Architecture**: Direct API access is more straightforward
3. **Better Resilience**: DRPC on each cluster (no single point of failure)
4. **Lower Latency**: Direct API calls vs. going through hub
5. **More Control**: Full control over cluster communication

## Challenges

1. **Kubeconfig Management**: Need to manage kubeconfigs for all peer clusters
2. **Security**: Mutual authentication between clusters
3. **Coordination Complexity**: Need robust coordination mechanism
4. **Network Requirements**: Direct network access between clusters
5. **State Management**: Need reliable state coordination (S3)
6. **Leader Election**: Need robust leader election mechanism

## Open Questions

1. **Network Connectivity**: Do clusters have direct network access to each other?
2. **Kubeconfig Rotation**: How to handle kubeconfig rotation/expiration?
3. **Multi-Cluster Failover**: How to handle DRPC failover if cluster hosting DRPC fails?
4. **Conflict Resolution**: How to resolve conflicts if multiple DRPCs try to coordinate?
5. **Performance**: Is S3 coordination fast enough for real-time operations?
6. **Scalability**: How does this scale to many clusters?

## Viewing Diagrams

### Mermaid Diagrams

The Mermaid diagrams in `drpc-managed-cluster-strawman.md` and `drpc-managed-cluster-strawman-sequences.md` can be viewed:

- In GitHub (renders automatically)
- In Markdown viewers that support Mermaid (VS Code, Obsidian, etc.)
- Online at [Mermaid Live Editor](https://mermaid.live)

### Graphviz Diagrams

Generate images from `drpc-managed-cluster-strawman-graphviz.dot`:

```bash
# Install Graphviz (if needed)
# Ubuntu/Debian: sudo apt-get install graphviz
# macOS: brew install graphviz
# Fedora: sudo dnf install graphviz

# Generate SVG (recommended for web)
dot -Tsvg drpc-managed-cluster-strawman-graphviz.dot -o drpc-managed-cluster-strawman-graphviz.svg

# Generate PNG (for documents)
dot -Tpng drpc-managed-cluster-strawman-graphviz.dot -o drpc-managed-cluster-strawman-graphviz.png

# Generate PDF (for printing)
dot -Tpdf drpc-managed-cluster-strawman-graphviz.dot -o drpc-managed-cluster-strawman-graphviz.pdf
```

## Related Documentation

- [OCM Dependencies Analysis](./ocm-dependencies-analysis.md)
- [OCM Dependencies Diagrams](./ocm-dependencies-diagrams.md)
- [OCM Dependencies Isolation](./ocm-dependencies-isolation.md)
- [Failover Sequence Diagrams](./failover-sequence-diagrams.md)

## Summary

This strawman proposes a significant architectural change to move DRPC from the OCM hub to managed clusters and eliminate all ACM dependencies. The proposal uses:

- **Direct K8s API access** instead of ManifestWork/ManagedClusterView
- **Custom ClusterRegistry** instead of ManagedCluster
- **S3-based coordination** for state and leader election
- **Active-Active DRPC deployment** for resilience

The migration path is designed to be gradual, with abstraction layers allowing both implementations to coexist during the transition period.
