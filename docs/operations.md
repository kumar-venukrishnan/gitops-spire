# Operations

## Inspect

```bash
kubectl get pods,svc,sts,ds,pvc -n spire
kubectl get events -n spire --sort-by=.lastTimestamp
kubectl top pods -n spire
```

## Argo CD

```bash
argocd app get spire
argocd app diff spire
argocd app sync spire
```

## Pause versus uninstall

- Disabling automated sync does not stop running pods.
- Deleting the Application with `--cascade=false` leaves resources running but unmanaged.
- Cascading deletion removes managed resources; verify PVC and datastore retention before doing this.
- Back up the SPIRE datastore and registration state before destructive changes.

## Upgrade discipline

Do not use a floating chart version or chart `main` branch. Review chart release notes, SPIRE release notes, CRD changes and rendered manifests. Upgrade in a lab first, then update `targetRevision` and image tags together where the chart supports it.
