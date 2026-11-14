# Architecture

## Deployment boundary

```text
Existing Argo CD installation (namespace: argocd)
                 |
                 | Application: spire
                 v
SPIFFE Helm chart (dedicated namespace: spire)
       |                         |
       v                         v
SPIRE Server                 SPIRE Agent (DaemonSet)
       |                         |
       | registration / trust    | Workload API Unix socket
       +-------------------------+
                                 |
                         Explicitly registered
                           test workloads
```

The repository creates a standalone Argo CD Application. It does not modify the existing root application or any existing PoC component.

## Identity responsibilities

- NetworkPolicy: limits network reachability.
- SPIRE: attests workloads and issues SPIFFE SVIDs.
- Application authorization: decides what an authenticated SPIFFE ID can do.
- OpenBao: remains the secrets and privileged credential service.
- Kubernetes RBAC: remains the Kubernetes API authorization layer.
- Authentik: remains human OIDC authentication for the existing Access Broker.

SPIRE does not replace NetworkPolicy, Kubernetes RBAC, OpenBao or Authentik.

## Lab assumptions

- One Kubernetes node, therefore one SPIRE Agent pod.
- SPIRE Server uses chart-default SQLite persistence on a PVC.
- Kubernetes PSAT is the node-attestation mechanism in this chart setup.
- Workload identities are not automatically granted to every pod.
