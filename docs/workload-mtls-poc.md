# SPIFFE/SPIRE Workload Identity and mTLS Proof of Concept

## 1. Overview

This document describes the workload identity and mutual TLS (mTLS) proof of concept implemented on the Docker Desktop Kubernetes cluster using SPIFFE and SPIRE.

The objective is to demonstrate how Kubernetes workloads can:

1. Receive cryptographically verifiable SPIFFE identities.
2. Obtain X.509-SVID certificates from SPIRE.
3. Authenticate another workload using mutual TLS.
4. Verify the peer's SPIFFE identity using its certificate URI Subject Alternative Name (URI SAN).
5. Allow or reject requests based on the authenticated workload identity.

The implementation uses a dedicated Kubernetes namespace, explicitly registered ServiceAccounts, the SPIRE Workload API, and Python applications using the standard-library `ssl` module.

This is a learning environment, not a production deployment.

## 2. Environment

| Component | Configuration |
|---|---|
| Kubernetes platform | Docker Desktop Kubernetes |
| Kubernetes topology | Single node |
| Kubernetes version | 1.36.1 |
| SPIRE namespace | `spire` |
| Workload namespace | `spire-demo` |
| Argo CD namespace | `argocd` |
| SPIFFE trust domain | `myorg.local` |
| SPIRE cluster name | `k8s-poc` |
| SPIRE Helm chart | `spire` |
| Helm chart version | `0.30.4` |
| SPIRE version | `1.15.5` |
| Workload API integration | SPIFFE CSI driver |
| Application runtime | Python 3.12 Alpine |

The trust domain `myorg.local` is used for this lab. A stable DNS-like trust domain should be selected before a production deployment.

## 3. Architecture

### 3.1 Deployment architecture

SPIRE infrastructure is deployed and managed independently through the `gitops-spire` repository and Argo CD.

The parent Argo CD Application monitors the GitHub repository and deploys the child Application. The child Application installs the SPIRE Helm chart into the `spire` namespace.

The workload demonstration runs separately in `spire-demo`.

```text
GitHub: gitops-spire
       |
       v
Argo CD Parent Application
gitops-spire
       |
       v
Kustomize bootstrap/
       |
       v
Argo CD Child Application
spire
       |
       v
SPIFFE Hardened Helm Chart 0.30.4
       |
       +--------------------------+
       |                          |
       v                          v
  SPIRE Server              SPIRE Agent
  namespace: spire          DaemonSet
       |                          |
       |                          | Workload API
       |                          | Unix socket
       |                          v
       |                    SPIFFE CSI Driver
       |                          |
       +--------------------------+
                                  |
                                  v
                         Namespace: spire-demo
                                  |
                 +----------------+----------------+
                 |                |                |
                 v                v                v
             TLS Server     Authorized Client  Unauthorized Client
             SA:            SA:                SA:
             spire-server   spire-client       spire-unauthorized
                 |                |                |
                 +----------------+----------------+
                                  |
                                  v
                         X.509-SVID mTLS
```

### 3.2 Workload identities

The proof of concept uses these SPIFFE IDs:

| Workload | Kubernetes ServiceAccount | SPIFFE ID |
|---|---|---|
| mTLS server | `spire-server` | `spiffe://myorg.local/ns/spire-demo/sa/spire-server` |
| Authorized client | `spire-client` | `spiffe://myorg.local/ns/spire-demo/sa/spire-client` |
| Unauthorized client | `spire-unauthorized` | `spiffe://myorg.local/ns/spire-demo/sa/spire-unauthorized` |
| Inspection workload | `spire-demo` | `spiffe://myorg.local/ns/spire-demo/sa/spire-demo` |

The SPIFFE ID is derived from the trust domain and the Kubernetes namespace and ServiceAccount selectors configured in the SPIRE registration entry.

### 3.3 Component responsibilities

| Component | Responsibility |
|---|---|
| Kubernetes | Runs workloads and provides workload metadata |
| SPIRE Server | Manages registration entries and issues SVIDs |
| SPIRE Agent | Attests workloads and serves the Workload API |
| SPIFFE CSI driver | Makes the Workload API socket available to selected Pods |
| X.509-SVID | Provides a cryptographically verifiable workload identity |
| Trust bundle | Allows workloads to validate certificates issued by the SPIRE trust domain |
| TLS server | Validates client certificates and authorizes the client SPIFFE ID |
| TLS client | Validates the server certificate and expected server SPIFFE ID |

SPIRE provides workload identity and attestation. Application authorization remains the responsibility of the application.

## 4. Kubernetes workload registration

### 4.1 Create the workload namespace

```bash
kubectl create namespace spire-demo \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 4.2 Create dedicated ServiceAccounts

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spire-demo
  namespace: spire-demo
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spire-server
  namespace: spire-demo
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spire-client
  namespace: spire-demo
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spire-unauthorized
  namespace: spire-demo
```

Apply the configuration:

```bash
kubectl apply -f serviceaccounts.yaml
```

Alternatively, create the ServiceAccounts individually using `kubectl create serviceaccount`.

### 4.3 Register workload identities in SPIRE

The SPIRE Server registration entries associate a workload's Kubernetes namespace and ServiceAccount with a SPIFFE ID.

First, identify the current SPIRE Agent SPIFFE ID:

```bash
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server agent list
```

Use the current Agent SPIFFE ID as the parent ID when creating the workload entries.

The registration entry structure is:

```bash
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry create \
  -parentID "<CURRENT_AGENT_SPIFFE_ID>" \
  -spiffeID "spiffe://myorg.local/ns/spire-demo/sa/spire-server" \
  -selector "k8s:ns:spire-demo" \
  -selector "k8s:sa:spire-server"
```

Repeat the registration with the following identities and selectors:

| SPIFFE ID | Namespace selector | ServiceAccount selector |
|---|---|---|
| `spiffe://myorg.local/ns/spire-demo/sa/spire-server` | `k8s:ns:spire-demo` | `k8s:sa:spire-server` |
| `spiffe://myorg.local/ns/spire-demo/sa/spire-client` | `k8s:ns:spire-demo` | `k8s:sa:spire-client` |
| `spiffe://myorg.local/ns/spire-demo/sa/spire-unauthorized` | `k8s:ns:spire-demo` | `k8s:sa:spire-unauthorized` |
| `spiffe://myorg.local/ns/spire-demo/sa/spire-demo` | `k8s:ns:spire-demo` | `k8s:sa:spire-demo` |

The unauthorized client is deliberately registered too. This allows the test to demonstrate the difference between having a valid SPIRE-issued identity and being authorized by the receiving application.

Verify the entries:

```bash
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry show
```

**Important:** The Agent SPIFFE ID is specific to the current Agent registration. Do not permanently hardcode a previously observed Agent ID in documentation or automation. Retrieve the current value when recreating registration entries.

## 5. Workload API and X.509-SVID issuance

### 5.1 SPIFFE CSI integration

The workload Pods use the SPIFFE CSI driver to access the SPIRE Agent Workload API socket.

The volume definition used in the example manifests is:

```yaml
volumes:
  - name: spiffe-workload-api
    csi:
      driver: csi.spiffe.io
      readOnly: true
```

The init container mounts the socket at:

```text
/run/spire/sockets
```

The socket used by the workload is:

```text
/run/spire/sockets/spire-agent.sock
```

The socket is mounted read-only.

Only explicitly selected workloads should receive access to the Workload API socket. Socket access is security-sensitive because it provides the interface through which workloads request identities.

### 5.2 Fetching an X.509-SVID

The server and client Pods use an init container based on:

```text
ghcr.io/spiffe/spire-agent:1.15.5
```

The init container invokes the SPIRE Agent binary directly:

```yaml
command:
  - /opt/spire/bin/spire-agent
args:
  - api
  - fetch
  - x509
  - -socketPath
  - /run/spire/sockets/spire-agent.sock
  - -write
  - /svid
```

This retrieves the workload's X.509-SVID and trust bundle and writes them into the shared `/svid` directory.

The resulting files include:

```text
/svid/svid.0.pem
/svid/svid.0.key
/svid/bundle.0.pem
```

| File | Purpose |
|---|---|
| `svid.0.pem` | Workload X.509 certificate |
| `svid.0.key` | Corresponding private key |
| `bundle.0.pem` | Trust bundle used to validate peer certificates |

The directory is shared with the application container using an `emptyDir` volume.

The private key is not included in the Git repository. It is generated and written inside the Pod at runtime.

### 5.3 Why an init container is used

The init container obtains the X.509-SVID before the application starts.

This ensures that the application can load its certificate, private key and trust bundle during startup.

The current implementation performs a one-time fetch. It does not demonstrate automatic SVID renewal or application certificate reload.

## 6. mTLS server implementation

### 6.1 Kubernetes resources

The server manifest is:

```text
examples/server.yaml
```

It creates:

| Resource | Name |
|---|---|
| Service | `spire-mtls-server` |
| Pod | `spire-mtls-server` |
| Namespace | `spire-demo` |
| ServiceAccount | `spire-server` |
| Container | `tls-server` |
| Container image | `python:3.12-alpine` |
| Port | `8443` |

The Service is an internal `ClusterIP` Service.

The client connects using:

```text
spire-mtls-server.spire-demo.svc.cluster.local:8443
```

No Ingress, NodePort or LoadBalancer is required for this internal workload-to-workload test.

### 6.2 Server TLS configuration

The Python server uses the standard-library `ssl` module.

Its TLS context is configured with:

```python
context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
context.load_cert_chain(
    "/svid/svid.0.pem",
    "/svid/svid.0.key"
)
context.load_verify_locations(
    cafile="/svid/bundle.0.pem"
)
context.verify_mode = ssl.CERT_REQUIRED
```

This configuration:

1. Loads the server's SPIRE-issued certificate and private key.
2. Loads the SPIRE trust bundle.
3. Requires a client certificate during the TLS handshake.
4. Validates the client certificate against the configured trust bundle.

### 6.3 Client SPIFFE ID authorization

After the TLS connection is established, the server extracts URI SAN values from the client certificate.

The expected client identity is:

```text
spiffe://myorg.local/ns/spire-demo/sa/spire-client
```

The server compares the presented URI SAN values against this exact identity.

The authorization logic is:

```python
EXPECTED_CLIENT_ID = (
    "spiffe://myorg.local/ns/spire-demo/sa/spire-client"
)

uri_sans = [
    value
    for kind, value in peer.get("subjectAltName", [])
    if kind == "URI"
]

if EXPECTED_CLIENT_ID not in uri_sans:
    # Reject the authenticated but unauthorized identity.
    ...
```

The server returns:

- HTTP `200 OK` for the authorized client.
- HTTP `403 Forbidden` for a client whose certificate is valid but whose SPIFFE identity is not allowed.

This is application-level identity authorization after TLS authentication. It is not a TLS handshake rejection based on the SPIFFE ID.

## 7. Authorized mTLS client

### 7.1 Kubernetes resources

The client manifest is:

```text
examples/client.yaml
```

It creates the Pod:

```text
spire-mtls-client
```

The Pod uses:

| Setting | Value |
|---|---|
| Namespace | `spire-demo` |
| ServiceAccount | `spire-client` |
| Application container | `tls-client` |
| Image | `python:3.12-alpine` |
| Server | `spire-mtls-server.spire-demo.svc.cluster.local` |
| Port | `8443` |

### 7.2 Client TLS configuration

The client loads the SPIRE trust bundle and its own certificate:

```python
context = ssl.create_default_context(
    ssl.Purpose.SERVER_AUTH,
    cafile="/svid/bundle.0.pem"
)

context.check_hostname = False
context.verify_mode = ssl.CERT_REQUIRED

context.load_cert_chain(
    "/svid/svid.0.pem",
    "/svid/svid.0.key"
)
```

`check_hostname` is disabled because the example verifies the server's SPIFFE URI SAN explicitly rather than relying on a DNS Subject Alternative Name.

The client still validates the server certificate chain using the SPIRE trust bundle.

### 7.3 Server SPIFFE identity verification

The expected server identity is:

```text
spiffe://myorg.local/ns/spire-demo/sa/spire-server
```

The client extracts the URI SAN values from the server certificate and checks for the exact expected identity.

Only after the identity check succeeds does the client send its HTTP request.

This demonstrates two-way workload authentication:

- The client presents its SVID to the server.
- The server presents its SVID to the client.
- Both sides validate the certificate chain.
- Both sides check the expected SPIFFE identity at the application level.

### 7.4 Expected successful response

The client should report:

```text
SPIRE mTLS client started
Expected server SPIFFE ID: spiffe://myorg.local/ns/spire-demo/sa/spire-server
Server SPIFFE identities: ['spiffe://myorg.local/ns/spire-demo/sa/spire-server']
SERVER IDENTITY VERIFIED
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 37
Connection: close
SPIFFE mTLS authentication successful
```

The client retries the connection every 10 seconds.

## 8. Unauthorized client test

### 8.1 Purpose

The unauthorized client demonstrates that a valid certificate from the SPIRE trust domain does not automatically grant access to every application.

The manifest is:

```text
examples/unauthorized-client
```

The Pod is:

```text
spire-mtls-unauthorized-client
```

It uses:

| Setting | Value |
|---|---|
| Namespace | `spire-demo` |
| ServiceAccount | `spire-unauthorized` |
| Expected server identity | `spiffe://myorg.local/ns/spire-demo/sa/spire-server` |
| Application image | `python:3.12-alpine` |
| Server port | `8443` |

Its SPIFFE identity is:

```text
spiffe://myorg.local/ns/spire-demo/sa/spire-unauthorized
```

The client obtains its own X.509-SVID using the same Workload API procedure as the authorized client.

It validates the server's certificate chain and expected server SPIFFE identity, then sends an HTTP request.

### 8.2 Expected authorization result

The server accepts the TLS connection because the unauthorized workload has a valid certificate issued by the same SPIRE trust domain.

However, the server's application authorization check compares the client SPIFFE ID with the configured allow-list.

The unauthorized identity does not match:

```text
Expected:
spiffe://myorg.local/ns/spire-demo/sa/spire-client

Presented:
spiffe://myorg.local/ns/spire-demo/sa/spire-unauthorized
```

The server rejects the request and returns:

```text
HTTP/1.1 403 Forbidden
```

The response body is:

```text
Identity not allowed
```

The unauthorized client manifest collects and prints the server response.

### 8.3 Observed server-side evidence

The server logs confirmed the rejection:

```text
Client SPIFFE identities:
['spiffe://myorg.local/ns/spire-demo/sa/spire-unauthorized']

REJECTED: unauthorized SPIFFE identity
```

The authorized client continued to be accepted.

**Interpretation:** The negative test demonstrates application-level authorization based on the SPIFFE identity. It does not demonstrate that the TLS handshake itself rejects the unauthorized identity.

## 9. Deploying the example workloads

The example manifests are kept separate from the SPIRE infrastructure deployment.

Apply them after SPIRE is healthy, the CSI driver is available, and the required ServiceAccounts and registration entries exist.

From the repository root:

```bash
kubectl apply -f examples/server.yaml
kubectl apply -f examples/client.yaml
kubectl apply -f examples/unauthorized-client
```

The inspection workload can be applied separately when required:

```bash
kubectl apply -f examples/spire-demo-inspect.yaml
```

Check the workload Pods:

```bash
kubectl get pods -n spire-demo -o wide
```

Check the Service:

```bash
kubectl get svc -n spire-demo
```

Check the server logs:

```bash
kubectl logs -n spire-demo spire-mtls-server
```

Check the authorized client logs:

```bash
kubectl logs -n spire-demo spire-mtls-client
```

Check the unauthorized client logs:

```bash
kubectl logs -n spire-demo spire-mtls-unauthorized-client
```

Follow logs continuously:

```bash
kubectl logs -n spire-demo spire-mtls-server -f
```

```bash
kubectl logs -n spire-demo spire-mtls-client -f
```

```bash
kubectl logs -n spire-demo spire-mtls-unauthorized-client -f
```

## 10. Inspection workload

The optional inspection manifest is:

```text
examples/spire-demo-inspect.yaml
```

It creates a Pod named:

```text
spire-demo-inspect
```

The Pod uses:

- Namespace: `spire-demo`
- ServiceAccount: `spire-demo`
- Image: `alpine:3.22`
- SPIFFE CSI volume mounted at `/run/spire/sockets`

The container installs OpenSSL and remains running for inspection.

This is a diagnostic workload for examining the mounted Workload API socket and related TLS material. It does not itself implement the server/client mTLS exchange.

Check the Pod:

```bash
kubectl get pod spire-demo-inspect -n spire-demo
```

Open a shell:

```bash
kubectl exec -it -n spire-demo spire-demo-inspect -- /bin/sh
```

## 11. Validation checklist

### 11.1 SPIRE infrastructure

Check the SPIRE Server:

```bash
kubectl exec -n spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server healthcheck
```

Expected:

```text
Server is healthy.
```

Check the Agent:

```bash
AGENT_POD=$(kubectl get pods -n spire \
  -l app.kubernetes.io/name=agent \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n spire "$AGENT_POD" -c spire-agent -- \
  /opt/spire/bin/spire-agent healthcheck
```

Expected:

```text
Agent is healthy.
```

### 11.2 Registration entries

```bash
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry show
```

Confirm that the expected server, authorized client and unauthorized client SPIFFE IDs are registered with the correct namespace and ServiceAccount selectors.

### 11.3 Workload status

```bash
kubectl get pods -n spire-demo
```

Expected workload Pods:

```text
spire-mtls-server
spire-mtls-client
spire-mtls-unauthorized-client
```

The optional inspection Pod is:

```text
spire-demo-inspect
```

### 11.4 Positive test

Confirm the authorized client logs show:

```text
SERVER IDENTITY VERIFIED
HTTP/1.1 200 OK
SPIFFE mTLS authentication successful
```

Confirm the server logs show:

```text
ACCEPTED: authorized SPIFFE identity
```

### 11.5 Negative test

Confirm the server logs show:

```text
REJECTED: unauthorized SPIFFE identity
```

The unauthorized client's HTTP response should be:

```text
HTTP/1.1 403 Forbidden
```

The client manifest reads and prints the response. Capture this output when recording a fresh end-to-end test run.

## 12. Important implementation details and limitations

### 12.1 One-time SVID retrieval

The current implementation fetches the X.509-SVID during Pod initialization.

It does not maintain a streaming Workload API connection or automatically reload certificates.

If an SVID expires while the application is still running, the application may continue using the old certificate and fail subsequent TLS connections.

A production implementation should use the Workload API to receive SVID updates and implement certificate rotation and application reload.

### 12.2 Application-level authorization

The server uses a hardcoded expected SPIFFE ID:

```text
spiffe://myorg.local/ns/spire-demo/sa/spire-client
```

This is sufficient for demonstrating the identity-based authorization concept.

For production, authorization policy should be managed centrally and should support controlled policy changes, auditability and appropriate error handling.

### 12.3 Certificate validation and identity validation are different

Certificate-chain validation establishes that the certificate was issued by a trusted CA.

SPIFFE ID validation establishes that the peer presents the expected workload identity.

Both checks are important.

A trusted certificate alone does not mean that the workload is authorized to access a particular service.

### 12.4 SPIRE does not replace other security controls

SPIRE provides workload identity and attestation. It does not replace:

- Kubernetes RBAC for Kubernetes API permissions.
- NetworkPolicy for network reachability.
- Application authorization for service-level access decisions.
- OpenBao for secrets and privileged credential management.
- Authentik for human OIDC authentication.

These systems have separate responsibilities and can complement each other.

### 12.5 CSI socket access

The Workload API socket should only be mounted into workloads that are explicitly intended to obtain SPIFFE identities.

Avoid granting broad access to every namespace or Pod.

### 12.6 Lab-specific trust domain

The trust domain `myorg.local` is appropriate for this isolated lab but should not be treated as a production naming recommendation.

A production trust domain should be selected carefully because changing it affects SPIFFE IDs and trust relationships.

## 13. Troubleshooting

### 13.1 SPIRE Server is unhealthy

```bash
kubectl get pods -n spire
kubectl logs -n spire spire-server-0 -c spire-server
kubectl describe pod -n spire spire-server-0
```

Check the Server health:

```bash
kubectl exec -n spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server healthcheck
```

### 13.2 SPIRE Agent is unhealthy

```bash
kubectl get pods -n spire -o wide
kubectl logs -n spire <agent-pod-name> -c spire-agent
```

Check Agent health:

```bash
kubectl exec -n spire <agent-pod-name> -c spire-agent -- \
  /opt/spire/bin/spire-agent healthcheck
```

### 13.3 Workload cannot fetch an SVID

Check:

1. The workload uses the intended ServiceAccount.
2. The SPIRE entry has the correct namespace and ServiceAccount selectors.
3. The registration entry has the current Agent SPIFFE ID as its parent.
4. The SPIFFE CSI driver is installed and available.
5. The CSI volume is mounted into the Pod.
6. The socket path matches the path used by the init container.

Inspect the init container:

```bash
kubectl describe pod -n spire-demo <pod-name>
```

```bash
kubectl logs -n spire-demo <pod-name> -c fetch-svid
```

### 13.4 Init container fails with `/bin/sh` not found

The SPIRE Agent image used in this example does not provide `/bin/sh`.

Invoke the binary directly:

```yaml
command:
  - /opt/spire/bin/spire-agent
args:
  - api
  - fetch
  - x509
  - -socketPath
  - /run/spire/sockets/spire-agent.sock
  - -write
  - /svid
```

Do not use a shell-based command in this SPIRE Agent init container.

### 13.5 Client cannot connect to the server

Check the Service and endpoints:

```bash
kubectl get svc spire-mtls-server -n spire-demo
kubectl get endpoints spire-mtls-server -n spire-demo
```

Check the server Pod:

```bash
kubectl get pod spire-mtls-server -n spire-demo -o wide
kubectl logs spire-mtls-server -n spire-demo
```

Confirm that the server listens on port `8443`.

### 13.6 TLS certificate validation fails

Check:

- The SVID certificate and private key are present.
- The trust bundle is present.
- The certificate has not expired.
- The client and server use the expected trust domain.
- The server and client are loading the correct certificate files.

Inspect the certificate:

```bash
openssl x509 -in /svid/svid.0.pem \
  -noout -subject -issuer -dates -ext subjectAltName
```

The SPIFFE ID should appear as a URI SAN.

### 13.7 Unauthorized client is accepted

Check the server's expected identity:

```text
spiffe://myorg.local/ns/spire-demo/sa/spire-client
```

Check the identity presented in the server log:

```text
spiffe://myorg.local/ns/spire-demo/sa/spire-unauthorized
```

The server must compare the presented URI SAN against the exact authorized identity.

Do not treat successful TLS certificate validation as sufficient application authorization.

## 14. Security improvements for a production implementation

The current proof of concept demonstrates the basic workload identity and mTLS flow. The following improvements should be considered before production use:

1. **Automatic SVID rotation:** Use the streaming Workload API and reload certificates without restarting workloads.
2. **Centralized authorization:** Replace hardcoded SPIFFE ID checks with managed authorization policy.
3. **Least privilege:** Restrict SPIRE registration selectors and Workload API socket access.
4. **Network restrictions:** Apply Kubernetes NetworkPolicies to limit unnecessary workload communication.
5. **Audit logging:** Record identity and authorization decisions in a consistent, centrally managed logging system.
6. **Application robustness:** Add request parsing, connection limits, timeouts, structured logging and appropriate error handling.
7. **High availability:** Review SPIRE Server persistence, backup, recovery and availability requirements.
8. **Trust-domain governance:** Establish a stable trust-domain naming and certificate lifecycle strategy.
9. **Operational monitoring:** Monitor SPIRE Server and Agent health, registration changes, SVID expiry and workload authentication failures.
10. **Upgrade management:** Review SPIRE and Helm chart release notes and validate upgrades in a test environment.

## 15. Proof-of-concept results

| Test | Expected result | Result |
|---|---|---|
| SPIRE Server health | Healthy | Passed |
| SPIRE Agent health | Healthy | Passed |
| Workload registration | Explicit workload identities registered | Passed |
| Server X.509-SVID retrieval | SVID and bundle written to `/svid` | Passed |
| Authorized client X.509-SVID retrieval | SVID and bundle written to `/svid` | Passed |
| Unauthorized client X.509-SVID retrieval | SVID and bundle written to `/svid` | Passed |
| Server certificate validation | Client validates server certificate chain | Passed |
| Server SPIFFE identity validation | Client verifies exact server URI SAN | Passed |
| Client certificate validation | Server requires and validates client certificate | Passed |
| Authorized client request | HTTP 200 | Passed |
| Unauthorized client authorization | Server rejects unauthorized SPIFFE ID | Passed |
| Unauthorized client HTTP 403 capture | HTTP 403 response printed by client | Capture recommended in a fresh test run |
| Automatic SVID rotation | Automatic renewal and reload | Not implemented |

## 16. Repository files

The workload identity example files are:

```text
gitops-spire/
├── README.md
├── bootstrap/
├── docs/
│   ├── architecture.md
│   ├── operations.md
│   ├── troubleshooting.md
│   └── workload-mtls-poc.md
└── examples/
    ├── README.md
    ├── server.yaml
    ├── client.yaml
    ├── unauthorized-client
    └── spire-demo-inspect.yaml
```

The example manifests are intended to be deployed manually for this proof of concept. They are not automatically deployed by the SPIRE infrastructure Argo CD Application.

Keep the SPIRE infrastructure lifecycle separate from the workload demonstration lifecycle. This allows the SPIRE control plane to remain independently managed while individual test workloads can be deployed, inspected and removed without changing the SPIRE installation.

## 17. References

- GitHub repository: https://github.com/kumar-venukrishnan/gitops-spire
- SPIRE Server manifest: https://github.com/kumar-venukrishnan/gitops-spire/blob/main/examples/server.yaml
- Authorized client manifest: https://github.com/kumar-venukrishnan/gitops-spire/blob/main/examples/client.yaml
- Unauthorized client manifest: https://github.com/kumar-venukrishnan/gitops-spire/blob/main/examples/unauthorized-client
- Inspection manifest: https://github.com/kumar-venukrishnan/gitops-spire/blob/main/examples/spire-demo-inspect.yaml
- SPIFFE Kubernetes quickstart: https://spiffe.io/docs/latest/try/getting-started-k8s/
- SPIRE project: https://github.com/spiffe/spire
- SPIFFE project: https://spiffe.io/
- SPIFFE hardened Helm charts: https://github.com/spiffe/helm-charts-hardened
