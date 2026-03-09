# Template for ManifestWork

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  finalizers:
  - cluster.open-cluster-management.io/manifest-work-cleanup
  labels:
    ramendr.openshift.io/created-by-ramen: "true"
  name: {{ .Name }}
  namespace: {{ .Cluster }}
spec:
  workload:
{{- toYaml .Manifests | nindent 2 }}
```
