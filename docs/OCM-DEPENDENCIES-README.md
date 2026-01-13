# OCM/RHACM Dependencies Documentation

This directory contains comprehensive documentation and diagrams analyzing Ramen's dependencies on Open Cluster Management (OCM) / Red Hat Advanced Cluster Management (RHACM).

## Documents

### 1. [ocm-dependencies-analysis.md](./ocm-dependencies-analysis.md)
**Comprehensive analysis** of all OCM/RHACM dependencies in Ramen, including:
- Executive summary
- Core OCM dependencies breakdown
- Detailed dependency mapping by component
- Resource flow descriptions
- Critical vs. replaceable dependencies
- Code locations

### 2. [ocm-dependencies-diagrams.md](./ocm-dependencies-diagrams.md)
**Mermaid diagrams** showing:
- Architecture overview
- OCM API dependency graph
- Resource flow sequences
- Component dependency matrix
- ManifestWork usage patterns
- ManagedClusterView usage patterns
- Dependency isolation analysis
- Code organization

### 3. [ocm-dependencies-graphviz.dot](./ocm-dependencies-graphviz.dot)
**Graphviz diagrams** for detailed dependency visualization. Generate images with:
```bash
# Generate all diagrams
dot -Tsvg ocm-dependencies-graphviz.dot -o ocm-dependencies-graphviz.svg
dot -Tpng ocm-dependencies-graphviz.dot -o ocm-dependencies-graphviz.png
dot -Tpdf ocm-dependencies-graphviz.dot -o ocm-dependencies-graphviz.pdf
```

### 4. [ocm-dependencies-isolation.md](./ocm-dependencies-isolation.md)
**Isolation strategy** for minimizing or removing OCM dependencies:
- Dependency categories (critical vs. replaceable)
- Isolation strategies for each dependency
- Implementation plan with phases
- Code examples for abstraction layers
- Migration path and testing strategy

## Quick Reference

### Critical OCM Dependencies

| API | Purpose | Impact if Removed |
|-----|---------|-------------------|
| `ManifestWork` | Deploy resources to managed clusters | Cannot deploy VRG, MaintenanceMode, etc. |
| `ManagedClusterView` | Read resources from managed clusters | Cannot read VRG status, cluster state |
| `Placement` | Cluster selection for workloads | Cannot determine target cluster |
| `PlacementDecision` | Store placement decisions | Cannot control cluster selection |
| `ManagedCluster` | Cluster discovery and validation | Cannot discover or validate clusters |

### Replaceable Dependencies

| API | Purpose | Replacement Strategy |
|-----|---------|----------------------|
| `PlacementRule` | Legacy placement | Migrate to Placement API |
| `Policy APIs` | Secret propagation | Use ManifestWork directly |

## Key Findings

### Architecture
- Ramen is **fundamentally built as an OCM placement extension**
- Hub operator runs on OCM hub cluster
- DR cluster operator has minimal OCM dependencies
- Core orchestration relies on OCM's multi-cluster management

### Dependencies
- **4 critical OCM APIs** that cannot be easily removed
- **2 replaceable dependencies** (legacy support)
- **Deep integration** with OCM's resource management patterns

### Isolation Strategy
- **Phase 1**: Create abstraction layers (low risk, high value)
- **Phase 2**: Remove legacy dependencies (medium risk, medium value)
- **Phase 3**: Alternative implementations (high risk, optional)

## Viewing Diagrams

### Mermaid Diagrams
The Mermaid diagrams in `ocm-dependencies-diagrams.md` can be viewed:
- In GitHub (renders automatically)
- In Markdown viewers that support Mermaid (VS Code, Obsidian, etc.)
- Online at [Mermaid Live Editor](https://mermaid.live)

### Graphviz Diagrams
Generate images from `ocm-dependencies-graphviz.dot`:
```bash
# Install Graphviz (if needed)
# Ubuntu/Debian: sudo apt-get install graphviz
# macOS: brew install graphviz
# Fedora: sudo dnf install graphviz

# Generate SVG (recommended for web)
dot -Tsvg ocm-dependencies-graphviz.dot -o ocm-dependencies-graphviz.svg

# Generate PNG (for documents)
dot -Tpng ocm-dependencies-graphviz.dot -o ocm-dependencies-graphviz.png

# Generate PDF (for printing)
dot -Tpdf ocm-dependencies-graphviz.dot -o ocm-dependencies-graphviz.pdf
```

## Code Locations

### Core OCM Integration
- **ManifestWork**: `internal/controller/util/mw_util.go`
- **ManagedClusterView**: `internal/controller/util/mcv_util.go`
- **ManagedCluster**: `internal/controller/util/managedcluster.go`
- **Placement**: `internal/controller/drplacementcontrol_controller.go`
- **PlacementRule**: `internal/controller/drplacementcontrol_controller.go`

### Controllers Using OCM
- **DRPlacementControl**: `internal/controller/drplacementcontrol.go`
- **DRPolicy**: `internal/controller/drpolicy_controller.go`
- **DRCluster**: `internal/controller/drcluster_controller.go`

## Recommendations

### For Understanding Dependencies
1. Start with `ocm-dependencies-analysis.md` for comprehensive overview
2. Review `ocm-dependencies-diagrams.md` for visual understanding
3. Generate Graphviz diagrams for detailed dependency graphs

### For Isolating Dependencies
1. Review `ocm-dependencies-isolation.md` for strategies
2. Start with Phase 1 (abstraction layers) - low risk
3. Consider Phase 2 (remove legacy) - medium risk
4. Skip Phase 3 (alternatives) unless OCM-free operation is required

### For Development
- Use abstraction layers to make dependencies explicit
- Keep OCM implementations as primary (they provide significant value)
- Consider alternatives only if OCM-free operation is a hard requirement

## Questions?

For questions or clarifications about OCM dependencies:
1. Review the detailed analysis documents
2. Check code locations referenced in the analysis
3. Review RBAC requirements in `config/rbac/role.yaml`

## Summary

Ramen is deeply integrated with OCM/RHACM, which provides:
- ✅ Scalable multi-cluster resource deployment (ManifestWork)
- ✅ Clean abstraction for reading remote cluster state (ManagedClusterView)
- ✅ Standard cluster selection mechanism (Placement)
- ✅ Cluster discovery and management (ManagedCluster)

While some dependencies can be abstracted or removed, the core architecture assumes OCM's multi-cluster management capabilities. Removing all OCM dependencies would require significant architectural changes and would result in a fundamentally different product.
