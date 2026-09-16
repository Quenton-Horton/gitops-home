# Policy Exception Register — Meridian Platform

Components that cannot satisfy the CIS/PSS policy pack, with justification and
compensating controls. Reviewed each authorization cycle.

Enforcement scope: `meridian-dev`, `meridian-staging`, `meridian-prod`.
All other namespaces run the same policies in reporting mode via background scans.

---

## EX-001 — Calico CNI (`calico-system`, `tigera-operator`)

**Failing controls:** CIS 5.2.1 (privileged), 5.2.2–5.2.4 (host namespaces),
5.2.5 (privilege escalation), 5.2.9 (capabilities), 5.7.2 (seccomp)

**Component:** `calico-node` (DaemonSet), `tigera-operator`, `csi-node-driver`

**Justification:** The CNI plugin programs the node's network stack — routing
tables, iptables/eBPF rules, and interface configuration. This requires host
network namespace access and NET_ADMIN capability by design. Without it the
cluster has no pod networking and no NetworkPolicy enforcement.

**Compensating controls:**
- Image source restricted to the upstream Tigera registry, version-pinned
- Deployed and reconciled through GitOps; drift is auto-reverted
- Namespace access restricted via RBAC to platform administrators
- Component is a prerequisite for SC-7 boundary protection, which it enforces

**Risk:** Accepted. Removing the exception removes network segmentation.

---

## EX-002 — Node Exporter (`monitoring`)

**Failing controls:** CIS 5.2.2–5.2.4 (host namespaces), 5.2.5, 5.2.9, 5.7.2

**Component:** `kps-prometheus-node-exporter` (DaemonSet)

**Justification:** Node-level metrics (CPU, memory, disk, filesystem) are read
from `/proc` and `/sys` on the host. Host PID namespace and host path mounts are
required to collect them. Without this, AU-6 (audit review and analysis) and
continuous monitoring of node health are not satisfiable.

**Compensating controls:**
- Mounts are read-only
- Exposes metrics only; no write path to the host
- Network access restricted to the Prometheus scraper
- Supports SI-4 (system monitoring) and continuous monitoring requirements

**Risk:** Accepted. Required for the monitoring capability the ATO depends on.

---

## EX-003 — Traefik Service Load Balancer (`kube-system`)

**Failing controls:** CIS 5.2.2–5.2.4 (host network), 5.2.5, 5.2.9, 5.7.2

**Component:** `svclb-traefik` (DaemonSet)

**Justification:** The k3s service load balancer binds host ports to expose
LoadBalancer-type Services. Host networking is inherent to the function.

**Compensating controls:**
- Only ports explicitly declared by Services are bound
- Ingress TLS termination enforced at the Traefik layer
- Platform-managed; not exposed to application teams

**Risk:** Accepted for this environment. In a cloud deployment this component is
replaced by a managed load balancer, eliminating the exception.

---

## EX-004 — Local Path Provisioner (`kube-system`)

**Failing controls:** CIS 5.2.2–5.2.4, 5.2.5, 5.2.9, 5.7.2

**Component:** `local-path-provisioner`

**Justification:** Provisions PersistentVolumes from node-local storage, which
requires host path access.

**Compensating controls:**
- Paths restricted to a single provisioner-owned directory
- Non-production storage class only

**Risk:** Accepted in development. **POA&M:** replace with a CSI driver backed by
network storage before production authorization.

---

## EX-005 — CoreDNS (`kube-system`)

**Failing controls:** CIS 5.2.5, 5.2.9, 5.7.2 (partial)

**Component:** `coredns`

**Justification:** Upstream manifest ships without a full restricted security
context. Cluster DNS is a hard dependency.

**Compensating controls:**
- Runs with NET_BIND_SERVICE only; other capabilities dropped
- Cluster-internal service, no external exposure

**Risk:** Accepted. **POA&M:** patch the manifest to add seccompProfile and
explicit capability drops; upstream-compatible change.

---

## Open Findings (not exceptions — to be remediated)

| ID | Component | Controls | Action |
|---|---|---|---|
| F-001 | `podinfo` (default ns) | CIS 5.2.5, 5.2.6, 5.2.9, 5.7.2 | Add restricted securityContext. No operational reason for non-compliance. |
| F-002 | `local-path-provisioner` | Storage | Replace with CSI driver (see EX-004) |

---

## Review

Exceptions are reviewed when: the component is upgraded, the authorization
boundary changes, or at each annual assessment. An exception with no
compensating control is a finding, not an exception.
