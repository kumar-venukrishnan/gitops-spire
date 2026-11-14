# Troubleshooting

## Application does not appear

```bash
kubectl get applications -n argocd
kubectl describe application spire -n argocd
```

Confirm that Argo CD is installed and the Application CRD exists.

## Chart cannot be fetched

Check Argo CD repo-server logs and whether it can reach:

- `https://spiffe.github.io/helm-charts-hardened/`

Check the chart version is still published.

## Server PVC Pending

```bash
kubectl get pvc -n spire
kubectl describe pvc -n spire
kubectl get storageclass
```

Docker Desktop normally provides a default StorageClass, but verify it.

## Pod CrashLoopBackOff or ImagePullBackOff

```bash
kubectl get pods -n spire
kubectl describe pod -n spire <pod-name>
kubectl logs -n spire <pod-name> --all-containers=true --previous
```

## Kubernetes 1.36 compatibility

The upstream SPIFFE Kubernetes quickstart currently documents testing through Kubernetes 1.34. If a component fails on 1.36.1, capture the exact pod events and logs before changing cluster-wide settings. Do not modify the existing cluster's API server flags as a first troubleshooting step.
