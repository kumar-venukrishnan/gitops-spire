# Bootstrap

This Kustomize root applies only the Argo CD Application object.

```bash
kubectl kustomize bootstrap/
kubectl apply -k bootstrap/
```

It does not directly apply SPIRE Server/Agent manifests. Argo CD installs the pinned SPIRE Helm chart as a separate application.
