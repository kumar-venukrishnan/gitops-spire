# Workload identity examples (next phase)

The first repository version deploys only the SPIRE control plane and agent. It intentionally does not issue identities to arbitrary workloads.

Suggested follow-up lab:

1. Create a dedicated namespace, e.g. `spire-demo`.
2. Create two dedicated ServiceAccounts, `client-a` and `server-b`.
3. Register narrowly scoped selectors for those ServiceAccounts and namespaces.
4. Mount the SPIRE Agent Workload API socket into only those test pods (use the chart's supported socket/CSI integration).
5. Fetch X.509-SVIDs and trust bundles from the Workload API.
6. Demonstrate mTLS success between the registered workloads.
7. Demonstrate rejection when the client does not present a trusted SVID.
8. Add application authorization based on the authenticated SPIFFE ID.

Do not mount the agent socket into every namespace by default. Socket access is security-sensitive because it is the interface through which workloads request identities.
