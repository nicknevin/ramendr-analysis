# CSI-Addon DR Architecture and Workflows

This folder contains documentation and diagrams for **standard CSI-addon based DR** architecture and the six storage operations.

## Documents

- **[csi-addon-dr-architecture-and-workflows.md](./csi-addon-dr-architecture-and-workflows.md)** — Main doc: Basic architecture (Hub, managed clusters, DRPC, VRG, VolumeReplication, S3) and operation workflows for all six operations. Includes Mermaid and Graphviz.
- **[csi-addon-dr-architecture-workflows-graphviz.dot](./csi-addon-dr-architecture-workflows-graphviz.dot)** — Graphviz source for the CSI-addon DR architecture diagram. Render with: `dot -Tpng csi-addon-dr-architecture-workflows-graphviz.dot -o csi-addon-dr-arch.png`

## Six Operations Covered

1. **Start data replication** – Enable protection; VRG Primary/Secondary and VolumeReplication (VR) drive replication.
2. **Pause data sync** – Stop replication (storage/vendor or VR state; Ramen has no dedicated pause API).
3. **Data re-sync** – Resume or re-sync (e.g. AutoResync on VR after failover).
4. **Cluster-down failover** – Unplanned failover; DRPC `action=Failover`, VRG Primary on surviving cluster, restore from S3.
5. **Planned failover (migration)** – Relocate; DRPC `action=Relocate`, final sync, VRG Primary on target.
6. **Failback** – Relocate back to original primary; DRPC `action=Relocate` to original cluster.

## Related Docs

- [failover-sequence-diagrams.md](../failover-sequence-docs/failover-sequence-diagrams.md) – Failover sequence details (Mermaid + Graphviz).
- [dr-api-interface-reference.md](../api-interface-reference/dr-api-interface-reference.md) – DR API (CRDs, status, orchestration).
- [OCM-DEPENDENCIES-README.md](../docs/OCM-DEPENDENCIES-README.md) – Hub/OCM dependency analysis.
