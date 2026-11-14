# gitops-spire

Standalone GitOps repository for deploying and managing SPIFFE/SPIRE on Kubernetes using **Argo CD, Kustomize and Helm**.

This repository is intentionally separate from the existing platform PoC GitOps repository. It allows SPIRE to be enabled, disabled, upgraded and managed independently.

## Environment

| Setting | Value |
|---|---|
| Kubernetes platform | Docker Desktop Kubernetes |
| Kubernetes topology | Single node |
| Kubernetes namespace | `spire` |
| Argo CD namespace | `argocd` |
| GitHub repository | `https://github.com/kumar-venukrishnan/gitops-spire.git` |
| Git branch | `main` |
| SPIFFE trust domain | `myorg.local` |
| SPIRE cluster name | `k8s-poc` |
| SPIRE Helm chart | `spire` |
| Helm chart version | `0.30.4` |
| SPIRE application version | `1.15.5` |

The trust domain and cluster name are stable logical identifiers. `k8s-poc` does not need to match the actual Docker Desktop Kubernetes context name.

`myorg.local` is a lab trust domain, not a production domain recommendation.

---

## 1. Architecture and GitOps design

This repository uses two Argo CD Applications.

### Parent Application: `gitops-spire`

The parent Application monitors this GitHub repository and branch.

It deploys the Kustomize configuration under `bootstrap/`.

### Child Application: `spire`

The child Application is created by the parent Application.

It retrieves the SPIRE Helm chart from the hardened SPIFFE Helm chart repository and installs SPIRE into the dedicated `spire` namespace.

### Deployment flow

```text
GitHub Repository
gitops-spire
main branch
      |
      v
Argo CD Parent Application
gitops-spire
      |
      | Monitors bootstrap/
      v
Kustomize
bootstrap/kustomization.yaml
      |
      v
Argo CD Child Application
spire
      |
      | Retrieves Helm chart
      v
SPIFFE Hardened Helm Repository
https://spiffe.github.io/helm-charts-hardened/
      |
      v
SPIRE Helm Chart 0.30.4
      |
      v
Kubernetes Namespace: spire
      |
      +---- SPIRE Server
      |
      +---- SPIRE Agent
```

### Application responsibilities

| Application | Responsibility |
|---|---|
| `gitops-spire` | Monitors GitHub and manages the child Application |
| `spire` | Installs and reconciles SPIRE Helm chart resources |

The parent Application is deployed once using `kubectl apply`.

After that, Argo CD monitors the repository and manages subsequent changes.

**Important:** The parent Application is deployed into the existing `argocd` namespace. The child Application deploys SPIRE into `spire`.

---

## 2. Important design choices

- SPIRE has its own GitHub repository, separate from the existing PoC GitOps repository.
- The parent Application monitors the `main` branch.
- The parent Application uses Kustomize to deploy the child Application.
- The child Application installs the SPIRE Helm chart.
- The Helm chart version is pinned to `0.30.4`; do not track the chart repository's `main` branch.
- Chart release metadata reports SPIRE `1.15.5`.
- The chart is beta. Review its release notes and values before upgrading.
- SPIRE Server and Agent use the configured trust domain `myorg.local`.
- The logical SPIRE Kubernetes cluster name is `k8s-poc`.
- SPIRE is isolated in its own Kubernetes namespace.
- No existing Authentik, OpenBao, Access Broker, Step CA, cert-manager or other PoC resources are modified.
- No Ingress, NodePort, LoadBalancer or external SPIRE Server endpoint is enabled.
- OIDC Discovery Provider and Tornjak are disabled in the initial configuration.
- Cross-cluster federation is not enabled.
- The initial installation is a learning environment, not a production deployment.

### Official documentation

- Kubernetes quickstart: https://spiffe.io/docs/latest/try/getting-started-k8s/
- SPIRE downloads: https://spiffe.io/downloads/
- Hardened Helm charts: https://github.com/spiffe/helm-charts-hardened
- Helm chart repository: https://spiffe.github.io/helm-charts-hardened/
- SPIRE releases: https://github.com/spiffe/spire/releases

### Kubernetes compatibility note

The SPIFFE Kubernetes quickstart has documented testing against Kubernetes versions 1.29–1.34.

The existing Docker Desktop cluster reports Kubernetes 1.36.1.

Although the chart declares a Kubernetes compatibility floor, this does not establish that all SPIRE integrations have been tested on Kubernetes 1.36.1.

Treat the first deployment as a compatibility test. Inspect Argo CD sync status, Kubernetes events and SPIRE pod logs before proceeding.

---

## 3. Repository structure

The expected repository layout is:

```text
gitops-spire/
├── .gitignore
├── README.md
│
├── bootstrap/
│   ├── README.md
│   ├── root-application.yaml
│   ├── application.yaml
│   └── kustomization.yaml
│
├── docs/
│   ├── architecture.md
│   ├── operations.md
│   └── troubleshooting.md
│
└── examples/
    └── README.md
```

### File responsibilities

| File | Purpose |
|---|---|
| `bootstrap/root-application.yaml` | Parent Argo CD Application |
| `bootstrap/application.yaml` | Child Argo CD Application that installs SPIRE |
| `bootstrap/kustomization.yaml` | Kustomize root referencing the child Application |
| `bootstrap/README.md` | Bootstrap instructions |
| `docs/architecture.md` | Architecture and component design |
| `docs/operations.md` | Operational procedures |
| `docs/troubleshooting.md` | Troubleshooting guidance |
| `examples/README.md` | Future workload identity and mTLS exercises |

---

## 4. Configure the GitHub repository

Create the GitHub repository:

```text
https://github.com/kumar-venukrishnan/gitops-spire
```

Use `main` as the default branch.

Upload the extracted repository contents to the root of the GitHub repository.

Make sure the `bootstrap` directory contains all three files:

```text
bootstrap/
├── root-application.yaml
├── application.yaml
└── kustomization.yaml
```

### Important: Two different repository URLs

The parent and child Applications use different source URLs.

**Parent Application: `bootstrap/root-application.yaml`**

```yaml
source:
  repoURL: https://github.com/kumar-venukrishnan/gitops-spire.git
  targetRevision: main
  path: bootstrap
```

This points to your GitHub repository.

**Child Application: `bootstrap/application.yaml`**

```yaml
source:
  repoURL: https://spiffe.github.io/helm-charts-hardened/
  chart: spire
  targetRevision: 0.30.4
```

This points to the SPIRE Helm chart repository.

Do not replace the child Application's Helm repository URL with the GitHub URL.

If the GitHub repository is private, configure repository credentials in Argo CD before bootstrapping.

---

## 5. Validate the repository locally

From the extracted repository directory:

```bash
kubectl version
kubectl kustomize bootstrap
```

The Kustomize output should contain one Argo CD Application:

```text
kind: Application
metadata:
  name: spire
```

Confirm Argo CD is installed:

```bash
kubectl get crd applications.argoproj.io
kubectl get pods -n argocd
```

Check the current Kubernetes context:

```bash
kubectl config current-context
```

The destination server configured in both Applications is:

```text
https://kubernetes.default.svc
```

This tells Argo CD to deploy to the same Kubernetes cluster where Argo CD is running.

No Kubernetes authentication token needs to be embedded in the Application manifests.

---

## 6. Bootstrap the parent Application

The parent Application must be applied directly because it is not included in the child Kustomize resources.

Run:

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/kumar-venukrishnan/gitops-spire/main/bootstrap/root-application.yaml
```

This creates:

```text
Application: gitops-spire
Namespace:   argocd
```

The parent Application then retrieves the Git repository and deploys the child Application.

Check the Applications:

```bash
kubectl get applications -n argocd
```

Expected:

```text
NAME          SYNC STATUS   HEALTH STATUS
gitops-spire  Synced        Healthy
spire         Synced        Healthy
```

The actual status may initially show `Progressing`, `OutOfSync` or `Missing` while resources are being created.

Inspect the parent:

```bash
kubectl describe application gitops-spire -n argocd
```

Inspect the child:

```bash
kubectl describe application spire -n argocd
```

Alternatively, use the Argo CD UI:

1. Open `https://argocd.myorg.local`.
2. Log in.
3. Locate `gitops-spire`.
4. Check its sync and health status.
5. Locate the child Application `spire`.

---

## 7. Namespace creation

The child Application targets:

```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: spire
```

Its sync options should include:

```yaml
syncOptions:
  - CreateNamespace=true
```

Argo CD creates the `spire` namespace automatically if it does not already exist.

No manual namespace creation is necessary.

Verify:

```bash
kubectl get namespace spire
```

---

## 8. Validate the SPIRE installation

Check all resources:

```bash
kubectl get all -n spire
```

Check configuration and persistent storage:

```bash
kubectl get serviceaccount,configmap,secret,pvc -n spire
```

Check events:

```bash
kubectl get events -n spire --sort-by=.lastTimestamp
```

Check SPIRE Server logs:

```bash
kubectl logs -n spire statefulset/spire-server \
  --all-containers=true
```

Check the Agent DaemonSet:

```bash
kubectl get daemonset -n spire
```

List the SPIRE pods:

```bash
kubectl get pods -n spire -o wide
```

Use the actual Agent pod name when inspecting logs:

```bash
kubectl logs -n spire <spire-agent-pod-name> \
  --all-containers=true
```

The precise resource names can vary with the pinned chart release.

---

## 9. Validate SPIRE Server health

Run:

```bash
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server healthcheck
```

List registered entries:

```bash
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry show
```

The initial installation intentionally does not register identities for every namespace or ServiceAccount.

---

## 10. Workload identity demo

The first deployment intentionally does not automatically register identities for every namespace or ServiceAccount.

This is an important security boundary.

Workloads should receive SPIFFE identities only after explicit registration and selector configuration.

The next stage will implement one of the following:

1. A narrowly scoped workload registration entry for a dedicated test namespace and ServiceAccount.
2. SPIRE Controller Manager and a `ClusterSPIFFEID` policy after reviewing its CRDs and selector semantics.

Do not grant a broad default identity to all Pods or
