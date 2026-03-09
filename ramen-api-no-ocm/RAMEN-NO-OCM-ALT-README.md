# Alternative APIs for Ramen Without OCM

The [proposal](./RAMEN-NO-OCM-README.md) for using Ramen to orchestrate Regional Disaster Recovery with no
dependencies on Open Cluster management (OCM) proposed reusing the ManifestWork[^MW] and ManagedClusterView[^MCV] CRDs.

This might be problematic for some because it has the appearance of requiring OCM/ACM though it is just the CRD
definitions which are being used and no OCM/ACM controllers or other such infrastructure.

To remove the apparent dependency on ManifestWork and ManagedClusterView, two approaches seem reasonable.

1. Provide a templating mechanism where an implementation of OTS provides templates which Ramen uses to generate CRs
for CRDs defined by the OTS implementation.
1. Create new CRDs to replace ManifestWork and ManagedClusterView. These would essentially be renamed copies of
   ManifestWork and ManagedClusterView with the same semantics.

Let's look at these two approaches in more detail.

## Templating Approach

The idea here is for the OTS implementation to provide two templates to Ramen, one for generating CR manifests for
deploying a resource from a cluster to another cluster (resource deployment) and one for generating CR manifests for
viewing on a cluster the state of a resource on another cluster (resource viewing).

The templates will be standard go templates with support for using functions from the
[github.com/go-sprout/sprout](https://github.com/go-sprout/sprout) package _hermetic_ registry group.
They will be provided to Ramen via a configuration setting in a known ConfigMap.

When Ramen wishes to deploy resource(s) remotely or view a remote resource it will pass a description of the resource(s) in as data to the
template expansion and it will then apply locally the manifest returned from the template expansion.

Via the template the OTS implementation can generate any sort of _spec_ it desires in the CR. Ramen does not consume the _spec_.

Dealing the the _status_ however is more complicated. Ramen expects a certain set of conditions and values for those
conditions in _status:conditions_ and in the case of MCV the result of the view, i.e. the resource object being viewed,
in _status:result_.

The set of the conditions and their expected values will be the same as for MW and MCV. Similarly the _result_ will be as for MCV. 
With respect to their location in the CR we can either fix them as for MW/MCV as _status:conditions_ and
_status:result_, or we can have the OTS implementation provide, along with the template, the jsonpath to them. For example in the
case of OCM/ACM the paths would be specifies as _status/conditions_ and _status/result_. 

An advantage of this approach is that it provides for the possibility that an implementor of OTS could use their
existing CRDs by providing templates to generate such. The requirements Ramen places on the _status_ however, makes this
unlikely.

An added complication of this approach is that it will require changes to Ramen to dynamically add watches for the GVK
of the generated CRs because their types will not be known a priori.

The input data to the template generation will follow the following schemas.
- for resource deployment
    ```yaml
    components:
      schemas:
        DeployConfig:
          type: object
          required:
            - Name
            - Cluster
            - Manifests
          properties:
            Name:
              type: string
              description: Name for the custom resource being generated
              example: "test-resource-deployment"
            Cluster:
              type: string
              description: Target cluster name
              example: "dr1"
            Manifests:
              type: array
              description: Array of Kubernetes manifest objects
              items:
                type: object
                additionalProperties: true
    ```
- for resource viewing
    ```yaml
    components:
      schemas:
        ViewConfig:
          type: object
          required:
            - Cluster
            - Name
            - Scope
          properties:
            Cluster:
              type: string
              description: Target cluster name
              example: "dr1"
            Name:
              type: string
              description: Name for the custom resource being generated
              example: "test-resource-view"
            Scope:
              $ref: '#/components/schemas/Scope'
        Scope:
          type: object
          description: Specifies the resource to view
          properties:
            ApiGroup:
              type: string
              description: API group of the resource to view
              example: "ramendr.openshift.io"
            Kind:
              type: string
              description: Kind of the resource to view
              example: "VolumeReplicationGroup"
            Name:
              type: string
              description: Name of the resource to view
              example: "ramendr.openshift.io"
            Namespace:
              type: string
              description: Namespace of the resource to view
              example: "default"
            Resource:
              type: string
              description: Type of the resource to view
              example: "volumereplicationgroups"
            UpdateIntervalSeconds:
              type: integer
              format: int32
              description: How often to update the view (in seconds)
              example: 3
              minimum: 0
            Version:
              type: string
              description: Version of the resource to view
              example: "v1alpha1"
    ```

The templates for an OCM/ACM based implementation of OTS would look something like this

- [template for ManifestWork](./mw-template.md).
- [template for ManagedClusterView](./mcv-template.md).

#### Additional considerations

- Should returning multiple resource manifests from a template expansion be allowed?

## Renamed CRDs Approach

This is the simpler of the two approaches. In this approach we simply create two new CRDs which are essentially copies
of MW and MCV scrubbed of all references to OCM/ACM and all OCM/ACM specific features (e.g. ManifestWork's
_spec:executor_ field), and which otherwise have the same semantics.

These CRDs might look something like this.

Replacing ManifestWork.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: deploy.resource.ramendr.openshift.io
spec:
  group: resource.ramendr.openshift.io
  names:
    kind: DeployResource
    listKind: DeployResourceList
    plural: deployresources
    singular: deployresource
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      ... as for ManifestWork with OCM/ACM features scrubbed ...
```

Replacing ManagedClusterView.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: view.resource.ramendr.openshift.io
spec:
  group: resource.ramendr.openshift.io
  names:
    kind: ViewResource
    listKind: ViewResourceList
    plural: viewresources
    singular: viewresource
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      ... as for ManagedClusterView ...
```

## Notes

### Security

Viewing of secrets via MCV or whatever replaces it, **is not allowed**.

## References

[^MW]: [ManifestWork](./manifestwork.crd.md) Custom Resource Definition
[^MCV]: [ManagedClusterView](./managedclusterview.crd.md) Custom Resource Definition
