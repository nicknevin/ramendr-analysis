# Template for ManagedClusterView

```yaml
apiVersion: view.open-cluster-management.io/v1beta1
kind: ManagedClusterView
metadata:
  annotations:
    drpolicy.ramendr.openshift.io: {{ .Cluster | quote }}
  labels:
    ramendr.openshift.io/created-by-ramen: "true"
  name: {{ .Name | quote }}
  namespace: {{ .Cluster | quote }}
spec:
  scope:
{{- if .Scope.ApiGroup }}
    apiGroup: {{ .Scope.ApiGroup }}
{{- end }}
{{- if .Scope.Kind }}
    kind: {{ .Scope.Kind }}
{{- end }}
{{- if .Scope.Name }}
    name: {{ .Scope.Name | quote }}
{{- end }}
{{- if .Scope.Version }}
    version: {{ .Scope.Version }}
{{- end }}
{{- if .Scope.UpdateIntervalSeconds }}
    updateIntervalSeconds: {{ .Scope.UpdateIntervalSeconds }}
{{- end }}
```
