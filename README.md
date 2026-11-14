# gitops-spire

Standalone GitOps repository for a small SPIFFE/SPIRE Kubernetes lab, intended to coexist with an existing Argo CD + Kustomize platform repository.

**Target cluster:** Docker Desktop Kubernetes, single node  
**Namespace:** `spire`  
**SPIFFE trust domain:** `ai-cloud.local` (lab-only; choose a stable DNS-like trust domain before production)  
**Cluster name:** `docker-desktop`  
**Git branch to create:** `main`

## Important design choices

- This repository is separate from the existing PoC GitOps repository.
- The repository's Kustomize root deploys only an Argo CD `Application`.
- That Application installs the SPIFFE-maintained `spire` umbrella Helm chart from the `helm-charts-hardened` project.
- The Helm chart is beta and its values/API can change. The chart release is pinned to `0.30.4`; do not track `main`.
- The chart release metadata reports SPIRE `1.15.5`. This is deliberately pinned as a known chart/application pairing. Review the chart release and SPIRE release notes before upgrading.
- No Authentik, OpenBao, Access Broker, Step CA, cert-manager, or existing namespaces are modified.
- No ingress, NodePort, LoadBalancer, external endpoint, or cross-cluster federation is enabled.
- The initial installation is a learning environment, not a production deployment.

Official documentation:
- Kubernetes quickstart: https://spiffe.io/docs/latest/try/getting-started-k8s/
- SPIRE downloads and chart guidance: https://spiffe.io/downloads/
- Hardened Helm charts: https://github.com/spiffe/helm-charts-hardened
- SPIRE release notes: https://github.com/spiffe/spire/releases

> Compatibility note: SPIFFE's current Kubernetes quickstart states it has been tested with Kubernetes 1.29–1.34. Your cluster reports Kubernetes 1.36.1. The chart declares a Kubernetes floor, but that does not prove every SPIRE integration has been tested on 1.36.1. Treat the first sync as a compatibility test; inspect Argo CD and pod events before proceeding.

## Repository layout

```text
gitops-spire/
├── README.md
├── .gitignore
├── argocd/
│   ├── application.yaml
│   └── kustomization.yaml
├── bootstrap/
│   ├── kustomization.yaml
│   └── README.md
├── docs/
│   ├── architecture.md
│   ├── operations.md
│   └── troubleshooting.md
└── examples/
    └── README.md
```

## 1. Create the GitHub repository

Create an **empty** repository named `gitops-spire` under your GitHub account. Use `main` as the default branch. Do not add a README or license on GitHub because this ZIP already contains them.

Extract this ZIP, then upload/push the extracted contents to the new repository's `main` branch using your preferred GitHub workflow.

The expected repository URL is:

```text
https://github.com/kumar-venukrishnan/gitops-spire
```

## 2. Update the repository URL if necessary

The Argo CD Application uses the expected URL above. If you create the repository under a different owner, edit:

`bootstrap/application.yaml`

and change `spec.source.repoURL`.

## 3. Validate locally before applying

From the extracted repository directory:

```bash
kubectl version
kubectl kustomize bootstrap
```

The Kustomize output should contain one Argo CD `Application` named `spire`.

Check that Argo CD is installed and that its Application CRD exists:

```bash
kubectl get crd applications.argoproj.io
kubectl get pods -n argocd
```

## 4. Enable SPIRE

Apply the standalone bootstrap:

```bash
kubectl apply -k bootstrap/
```

This creates an Argo CD Application named `spire`. Argo CD then installs the chart into the dedicated `spire` namespace.

Watch the Application and workloads:

```bash
kubectl get applications -n argocd
kubectl get pods -n spire -w
kubectl get pvc -n spire
```

Expected core components are SPIRE Server and SPIRE Agent. Depending on the chart's pinned defaults, supporting chart components may also be installed. Review the rendered resources in Argo CD before accepting the first sync.

## 5. Validate the installation

```bash
kubectl get all -n spire
kubectl get serviceaccount,configmap,secret,pvc -n spire
kubectl get events -n spire --sort-by=.lastTimestamp
```

Inspect logs:

```bash
kubectl logs -n spire statefulset/spire-server --all-containers=true
kubectl get daemonset -n spire
kubectl get pods -n spire -o wide
```

The precise resource names can vary with chart release. Use `kubectl get pods -n spire` to identify pod names before reading agent logs:

```bash
kubectl logs -n spire <spire-agent-pod-name> --all-containers=true
```

## 6. Workload identity demo

The first deployment intentionally does **not** automatically register identities for every namespace or ServiceAccount. That is a security boundary: workloads should receive identities only after explicit registration/selection.

Use the SPIRE chart's documentation for its installed release and choose one of these next steps:

1. Create a narrowly scoped workload registration entry for a dedicated test namespace and dedicated ServiceAccount; or
2. Enable/configure SPIRE Controller Manager and a `ClusterSPIFFEID` policy after reviewing its CRDs and selector semantics.

Do not grant a broad default identity to all pods or all ServiceAccounts. The `examples/` directory explains the planned two-workload mTLS exercise but deliberately does not deploy a permissive identity policy.

## 7. Enable / disable model

This is a separate Argo CD Application, so it is independent of the existing root application.

**Pause GitOps reconciliation only** (workloads continue running):

```bash
argocd app set spire --sync-policy none
```

**Resume automated reconciliation** (if desired):

```bash
argocd app set spire --sync-policy automated
```

**Remove the Argo CD Application but leave managed resources orphaned** (workloads continue running and are no longer reconciled):

```bash
argocd app delete spire --cascade=false
```

**Full uninstall** (destructive; deletes the chart-managed resources and namespace, and may remove SPIRE state):

```bash
argocd app delete spire --cascade
```

Before a full uninstall, export any registration entries and back up the SPIRE server datastore/PVC. Do not use full uninstall as a routine pause mechanism.

To reinstall after a full uninstall:

```bash
kubectl apply -k bootstrap/
```

## 8. Resource sizing

The chart values in `bootstrap/application.yaml` set conservative starting requests/limits for a single-node lab. They are not official minimums. Check actual use after installation:

```bash
kubectl top nodes
kubectl top pods -n spire
```

Your Docker Desktop Kubernetes allocation is approximately 7.748 GiB, not the full 16 GB physical Mac memory. Existing pods can use more than their requests, so watch actual memory and events.

## 9. Security and production gaps

This lab is not production-ready. Before production use, design and test:

- HA SPIRE Server and a supported external datastore strategy.
- Backup/restore and CA key protection.
- Restricted workload registration and Kubernetes RBAC.
- Node attestation appropriate to the platform (PSAT in this Kubernetes lab is not hardware-rooted attestation).
- Agent socket access controls and workload isolation.
- Trust-domain naming, federation and key rotation.
- Monitoring, audit retention, alerting and upgrade/rollback procedures.
- NetworkPolicy after verifying the chart's ports, DNS and kubelet/API dependencies.
- Compatibility testing against the exact Kubernetes and container runtime versions.

Do not reuse `ai-cloud.local` as a production trust domain without an explicit naming and federation design.
