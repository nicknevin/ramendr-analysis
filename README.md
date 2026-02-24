## Less ACM/OCM in Ramen

### Rationale

Ramen is currently the premier community Disaster Recovery orchestrator for Kubernetes. It is, however, very clearly
slanted towards the Red Hat/IBM "ecosystem" within Kubernetes, currently having siginificant dependencies on both
Ceph/ODF and OCM/RHACM. We believe that it would be beneficial for the whole Kubernetes community -- especially
the parts that are not running OpenShift and Red Hat Advanced Cluster Management -- to have standards for
Disaster Recovery, and Ramen has some very appealing features to make it that. SpecificallyAt the same time, other
orchestrators can make use of a standardized set of replication objects, and compete with Ramen in this space as
desired. The landscape is especially challenging for storage array vendors, whose products feature sophisticated
replication capabilities, but Kubernetes currently lacks the standardized mechanisms to allow their use. One of
the main goals of this effort is to create standard API objects that storage array vendors can provide in their
drivers; these objects can then be operated on by Ramen directly.

### Goals

1. *Standardize Ramen's CSI Add-ons as part of the Kubernetes CSI specification.* Today, Kubernetes does not have a
standardized way of talking about replication status between clusters. A pre-requisite for having a standardized
orchestrator is having a vendor-independent way of operating on storage objects, which Ramen has defined. This effort
is now underway in the Kubernetes Storage SIG and Data Protection Working Group as of February 2026. Previous attempts
to standardize these add-ons failed because there was not quorum or agreement from storage solution
vendors/implementors; this proposal has in part been prompted by a higher level of interest in such a standardization
process from different storage solution providers.

1. *Weaken (but do not eliminate) Ramen's dependency on ACM/OCM for orchestration.* Clearly a significant part of
the Kubernetes community (especially those who use ACM/OCM today) are well served by ACM-style orchestration, which
Ramen currently uses. Other customers and users have expressed desires to not require ACM, but to be able to use it
if needed. This has significant architectural ramifications for the project, which will be discussed below. This
proposal can and should be characterized as "ACM First, but not ACM _only_." This will consist in documenting Ramen's
existing ACM/OCM dependencies and providing concrete advice and a proposal on how to create space for an alternative
orchestration implementation to control Ramen and the DR process.

1. *Strengthen Ramen Upstream.* We believe that creating this architectural space within the Ramen project will make
Ramen more appealing to a broader audience, especially storage solution providers; in particular any who may have
initially evaluated Ramen and rejected it for being too Red Hat or IBM-specific. Historically, that is an
accurate description of Ramen upstream, though we note that upstream tests use OCM as opposed to ACM, do not use
OpenShift, and use MinIO (for example). Still, without a path to the standardization of the CSI add-ons, or without
any expressed interest from the project in accepting work towards making the ACM dependency optional, potential
contributors might have considered other options before trying to contribute to Ramen.

### Non-Goals

1. *GitOps integration/Subscription applications are out of scope.* Ramen's current GitOps/Subscription Applications
feature depends on some very specific integrations OCM has with ArgoCD/OpenShift GitOps. As a practical matter,
anecdotes suggest that this is not the way most users are doing Disaster Recovery, especially with Virtual Machines.

1. *UI/UX Presentation is out of scope.* It is absolutely the intention of this proposal to provide all the plumbing
and status reporting necessary to implement a UI/UX for Disaster Recovery. However, the framework or location for that
UI/UX is out of scope of this proposal. Ramen's current UI/UX layer is implemented as an ACM extension, which runs in
the OpenShift Console.

1. *Secure transport of secrets is out of scope.* This proposal will not specify how to transport secrets; either
how to retrieve them from managed clusters or how to create or manage them on DR-protected clusters.

1. *Management of ManagedClusterAddOns is out of scope.* Ramen can install and configure plugins like volsync for
OADP/Velero. This depends on ManagedClusterAddOns, an ACM/OCM feature. As such, users of Ramen without ACM/OCM will
need to orchestrate the installation of those components another way.

### Key Documents and Diagrams

These documents were prepared with the assistance of Cursor. The diagrams are in Mermaid format; and in addition
there are graphviz "dot" files provided as well.

#### [DR API Reference](api-interface-reference/dr-api-reference.md)

This document describes the APIs that Ramen itself provides.

#### [CSI AddOns Architecture and Workflows](csi-sequence/CSI-ADDON-DR-WORKFLOWS-README.md)

These documents describe how Ramen interacts with the CSI Replication Addons.

[Architecture and Workflows](csi-sequence/csi-addon-dr-architecture-and-workflows.md)
[GraphViz dot files](csi-sequence/csi-addon-dr-architecture-and-workflows.dot)

#### [Failover Sequence Documentation](failover-sequence-docs/FAILOVER-DIAGRAMS-README.md)

These documents describe Ramen's failover and state machine operations in detail.

[Diagrams](failover-sequence-docs/failover-sequence-diagrams.md)
[GraphViz dot files](failover-sequence-docs/failover-sequence-diagrams-graphviz.dot)

#### [Ramen OCM Dependency Analysis](docs/OCM-DEPENDENCIES-README.md)

These documents list the actual dependencies on ACM/OCM in Ramen.

[Analysis](docs/ocm-dependencies-analysis.md)
[Diagrams](docs/ocm-dependencies-diagrams.md)
[Isolation "Straw Man" Proposal](docs/ocm-dependencies-isolation.md)
[GraphViz dot files](docs/ocm-dependencies-graphviz.dot)

#### [Disaster Recovery Placement Control (DRPC) Migration to Managed Cluster "Straw Man"](drpc-managed-cluster/DRPC-MANAGED-CLUSTER-README.md)

In the absence of ACM/OCM, the question of where to locate the DR orchestration and control plane is open. This is a
strawman proposal for what such a thing might look like if the managed clusters themselves were running the
orchestration layer. (Other products in the marketplace take this approach.) This is not the only possible solution,
and is not intended to be construed as a requirement.

[Strawman Proposal](drpc-managed-cluster/drpc-managed-cluster-strawman.md)
[Strawman Sequences Proposal](drpc-managed-cluster/drpc-managed-cluster-strawman-sequences.md)
[GraphViz dot files](drpc-managed-cluster/drpc-managed-cluster-strawman-graphviz.dot)
