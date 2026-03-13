## Context

The target environment is an OpenShift 4.20.15 bare-metal SingleReplica cluster (`cluster-d45v2.dynamic.redhatworkshops.io`) with wildcard route domain `*.apps.cluster-d45v2.dynamic.redhatworkshops.io`. Storage is provided exclusively by OpenShift Data Foundation via the `ocs-external-storagecluster-ceph-rbd` StorageClass (Ceph RBD). The following operators and controllers are pre-installed cluster-wide and are not provisioned as part of this design:

- **Zabbix Operator 6.0.40** — provides the `ZabbixFull` CRD and manages server, web frontend, Java gateway, and PostgreSQL lifecycle
- **OpenShift GitOps v1.19.2 (ArgoCD)** — provides the `Application` CRD and GitOps sync machinery
- **cert-manager v1.18.1** — provides `ClusterIssuer` and `Certificate` CRDs; the `openshift-routes` controller must also be pre-installed (via Helm) to support automatic TLS provisioning on OpenShift Routes via annotations

All workload manifests live under an `argocd/` directory in this repository and are reconciled by a single ArgoCD `Application` resource. The target namespace is `zabbix-demo`.

The Zabbix Agent runs as a sidecar container inside a sample application `Deployment`, not as a standalone DaemonSet or VM guest. The agent communicates with Zabbix Server over a `ClusterIP` Service on ports 10051 (active checks, agent → server) and 10050 (passive checks, server → agent).

---

## Goals / Non-Goals

**Goals:**

- Deploy a functional Zabbix monitoring stack (server, web frontend, Java gateway, managed PostgreSQL) on OpenShift using the `ZabbixFull` CRD
- Expose the Zabbix web frontend via an edge-terminated OpenShift Route with automatic TLS provisioned by cert-manager's `openshift-routes` controller
- Demonstrate Zabbix Agent monitoring by running the agent as a sidecar container in a sample application `Deployment`, connected to Zabbix Server via a `ClusterIP` Service
- Manage all manifests declaratively through a single ArgoCD `Application` with explicit sync wave ordering to handle resource dependencies
- Grant required Security Context Constraints (SCCs) to relevant service accounts via namespaced `Role` and `RoleBinding` resources, without modifying any cluster-level default SCCs
- Use the cluster's default Ceph RBD `StorageClass` for all persistent volumes, with `WaitForFirstConsumer` binding mode

**Non-Goals:**

- High availability — this is a SingleReplica cluster; no replica counts above 1 or pod anti-affinity are targeted
- Production hardening — secrets management, network policies, resource limits tuning, and audit logging are out of scope
- External or self-managed PostgreSQL — the `ZabbixFull` operator-managed PostgreSQL instance is used
- CNV/VM-based Zabbix Agent deployment — the agent runs only as a sidecar container, not inside a KubeVirt VM
- Custom Zabbix templates, triggers, or alerting pipelines
- Multi-cluster or federated monitoring

---

## Decisions

### Manifest inventory and sync wave ordering

All seven manifests reside in the `argocd/` directory. ArgoCD sync waves are controlled via the `argocd.argoproj.io/sync-wave` annotation. Resources with a lower wave number are applied and must reach a healthy state before the next wave begins.

| Wave | File | apiVersion | Kind | Namespace |
|------|------|-----------|------|-----------|
| 0 | `namespace.yaml` | `v1` | `Namespace` | — |
| 1 | `rbac.yaml` | `rbac.authorization.k8s.io/v1` | `Role`, `RoleBinding` | `zabbix-demo` |
| 1 | `cluster-issuer.yaml` | `cert-manager.io/v1` | `ClusterIssuer` | — (cluster-scoped) |
| 2 | `zabbix-full.yaml` | `zabbix.com/v1alpha1` | `ZabbixFull` | `zabbix-demo` |
| 3 | `zabbix-web-route.yaml` | `route.openshift.io/v1` | `Route` | `zabbix-demo` |
| 3 | `sample-app.yaml` | `apps/v1` | `Deployment` + `v1` `Service` | `zabbix-demo` |
| 4 | `argocd-application.yaml` | `argoproj.io/v1alpha1` | `Application` | `openshift-gitops` |

Wave 0 ensures the namespace exists before any namespaced resources are created. Wave 1 establishes RBAC and the `ClusterIssuer` in parallel — the issuer must exist and reach `Ready` status before the Route requests a certificate. Wave 2 deploys the full Zabbix stack, which requires the namespace and SCC grants to be in place. Wave 3 deploys the Route and the sample application concurrently; the Route depends on the `ClusterIssuer` (wave 1) and the Zabbix web `Service` created by the operator (wave 2). Wave 4 registers the `Application` object itself if this document is bootstrapped via a root App-of-Apps pattern.

---

### namespace.yaml

`apiVersion: v1`, `kind: Namespace`

Creates the `zabbix-demo` namespace with no special labels beyond standard OpenShift namespace metadata. No pod security admission label overrides are set at the namespace level; SCC enforcement is handled through RBAC in wave 1.

---

### rbac.yaml

`apiVersion: rbac.authorization.k8s.io/v1`, `kind: Role` and `kind: RoleBinding`, namespace: `zabbix-demo`

The Zabbix Operator creates pods that may require elevated SCCs (e.g., `anyuid`) for the PostgreSQL or Zabbix Server containers to bind to specific UIDs. The approach is:

- A namespaced `Role` is created with a rule granting `use` on the `resourceNames` for the specific SCC (e.g., `anyuid`) under the `security.openshift.io` API group and `securitycontextconstraints` resource.
- A `RoleBinding` binds that `Role` to the relevant `ServiceAccount`(s) within `zabbix-demo` (typically the default service account or a dedicated SA created by the operator).

This approach scopes the SCC grant to the namespace only and does not modify the cluster-level `anyuid` or any other default SCC object, preserving cluster security posture. If the Zabbix Operator creates its own `ServiceAccount`, the `RoleBinding` subject references that SA explicitly by name.

---

### cluster-issuer.yaml

`apiVersion: cert-manager.io/v1`, `kind: ClusterIssuer`, cluster-scoped

Defines the certificate authority used to sign the Zabbix web frontend TLS certificate. Because cert-manager's `openshift-routes` controller watches `Route` objects for specific annotations and issues certificates using a `ClusterIssuer`, the issuer must be cluster-scoped.

The `ClusterIssuer` is configured for the appropriate CA or ACME solver available in the environment. It must reach `Ready: True` status before the Route in wave 3 can successfully obtain a certificate. ArgoCD's health check for `ClusterIssuer` should validate the `Ready` condition before proceeding to wave 3.

**Dependency note:** The `openshift-routes` cert-manager controller is not deployed by this design — it must be pre-installed cluster-wide (e.g., via a Helm release in the `cert-manager` namespace) before this ArgoCD `Application` is synced. Without it, Route annotations are ignored and no certificate is issued.

---

### zabbix-full.yaml

`apiVersion: zabbix.com/v1alpha1`, `kind: ZabbixFull`, namespace: `zabbix-demo`

This is the primary workload manifest. The Zabbix Operator reconciles this resource and creates all sub-resources: Zabbix Server `Deployment`, Zabbix Web frontend `Deployment`, Java Gateway `Deployment`, and a managed PostgreSQL `StatefulSet` with a `PersistentVolumeClaim`.

Key configuration decisions:

- **StorageClass**: The PostgreSQL PVC explicitly sets `storageClassName: ocs-external-storagecluster-ceph-rbd` to ensure the correct provisioner is used. The `volumeBindingMode: WaitForFirstConsumer` behavior of this StorageClass means the PVC will remain `Pending` until the PostgreSQL pod is scheduled to a node — this is expected and correct behavior, not an error.
- **Replicas**: All components are set to 1 replica. No HA configuration is applied.
- **Internal Services**: The operator creates `ClusterIP` Services for Zabbix Server (port 10051 for active agent connections, port 10050 for passive checks). These services are referenced by the Zabbix Agent sidecar in `sample-app.yaml`.
- **Web frontend hostname**: The frontend must be configured with the exact hostname that will be used in the Route (`zabbix.apps.cluster-d45v2.dynamic.redhatworkshops.io` or equivalent) so that the Zabbix Agent `Hostname` field in the sidecar matches the host registered in the frontend.

---

### zabbix-web-route.yaml

`apiVersion: route.openshift.io/v1`, `kind: Route`, namespace: `zabbix-demo`

Exposes the Zabbix web frontend `Service` (created by the operator) externally under the cluster wildcard domain.

TLS configuration and cert-manager annotation flow:

1. The Route sets `spec.tls.termination: edge` — TLS is terminated at the OpenShift router; traffic from router to pod is plain HTTP.
2. The following annotations instruct cert-manager's `openshift-routes` controller to provision a certificate:
   - `cert-manager.io/issuer-kind: ClusterIssuer`
   - `cert-manager.io/issuer-name: <name-of-ClusterIssuer-from-cluster-issuer.yaml>`
3. The `openshift-routes` controller detects the annotated Route, creates a cert-manager `Certificate` resource, and injects the resulting TLS certificate and key into `spec.tls.certificate` and `spec.tls.key` on the Route object.
4. The Route does not carry a static certificate in source control — cert-manager manages certificate lifecycle and renewal automatically.

The `spec.host` field is set explicitly to a subdomain under `*.apps.cluster-d45v2.dynamic.redhatworkshops.io`. The `spec.to.kind: Service` references the Zabbix web frontend `Service` by name.

---

### sample-app.yaml

`apiVersion: apps/v1`, `kind: Deployment` and `apiVersion: v1`, `kind: Service`, namespace: `zabbix-demo`

Demonstrates Zabbix Agent monitoring by running the agent as a sidecar alongside a workload container.

**Zabbix Agent sidecar configuration:**

The agent container uses the official `zabbix/zabbix-agent2` image (or the image specified by the operator's recommendations for version 6.0.x). The following environment variables or config file parameters are critical:

- `Server`: Set to the `ClusterIP` Service hostname of Zabbix Server within the cluster (e.g., `zabbix-server.zabbix-demo.svc.cluster.local`). This controls which server addresses the agent accepts passive check connections from.
- `ServerActive`: Set to the same `ClusterIP` Service hostname and port 10051. This is the address the agent connects to for active checks and self-registration.
- `Hostname`: Must match exactly the hostname registered for this host in the Zabbix frontend. Mismatch causes active checks to fail silently. This value must be coordinated with the Zabbix frontend host configuration.

**ClusterIP Service for agent-server communication:**

A `ClusterIP` Service is defined (or re-used from the operator-created Service) targeting the Zabbix Server pod on:
- Port 10051 — used by the agent for active check connections (agent initiates)
- Port 10050 — used by Zabbix Server for passive checks (server initiates connection to agent)

For passive checks to work, the Zabbix Server must be able to reach the agent sidecar. The `sample-app.yaml` also includes a `ClusterIP` Service exposing port 10050 on the sample application pod, so the Zabbix Server can poll the agent.

**No `hostNetwork` or `hostPID`** is used. The agent runs as an unprivileged sidecar within the pod's network namespace.

---

### argocd-application.yaml

`apiVersion: argoproj.io/v1alpha1`, `kind: Application`, namespace: `openshift-gitops`

Defines the ArgoCD `Application` that tracks the `argocd/` directory in this repository and syncs all manifests to the cluster.

Key configuration:
- `spec.source.repoURL` and `spec.source.path: argocd/` point to this repository
- `spec.destination.server: https://kubernetes.default.svc` and `spec.destination.namespace: zabbix-demo`
- `spec.syncPolicy.automated` is enabled with `prune: true` and `selfHeal: true` for continuous reconciliation
- `spec.syncPolicy.syncOptions` includes `CreateNamespace=false` (the namespace is managed by `namespace.yaml` in wave 0, not by ArgoCD's built-in namespace creation)
- Sync wave annotations on individual manifests drive the ordered rollout as described in the manifest inventory table above

If this `Application` manifest is itself managed by a parent App-of-Apps pattern, it is placed in wave 4 so that all workload resources are healthy before the application registration completes.

---

## Risks / Trade-offs

**SingleReplica — no HA**
The cluster has a single control plane and single worker. Any node maintenance or failure takes down all workloads simultaneously. This design accepts that trade-off because the environment is a demo cluster, not production.

**WaitForFirstConsumer PVC binding**
Ceph RBD PVCs with `WaitForFirstConsumer` binding mode remain in `Pending` until the consuming pod is scheduled. ArgoCD health checks may report the `ZabbixFull` resource as degraded during initial sync until the PostgreSQL pod is scheduled and the PVC binds. Operators and users should expect a brief period of `Progressing` status on first deploy.

**ClusterIssuer and openshift-routes controller as pre-conditions**
If the `openshift-routes` cert-manager controller is not pre-installed, the Route TLS annotations are silently ignored and the frontend will be accessible only over plain HTTP (or the router's default certificate). This design does not deploy the controller and relies on it being present. This dependency should be validated as part of environment setup before applying this `Application`.

**Agent Hostname coupling**
The Zabbix Agent `Hostname` parameter in the sidecar must exactly match the host record configured in the Zabbix frontend. There is no automatic registration mechanism configured in this design. If the frontend host record is renamed or the sidecar env var drifts, active checks silently stop working. This requires operational discipline when updating either side.

**SCC grant via namespaced Role**
Using a namespaced `Role` to grant SCC use is the recommended OpenShift pattern and avoids cluster-wide privilege escalation. However, if the Zabbix Operator creates new `ServiceAccounts` across upgrades, the `RoleBinding` subjects may need to be updated to include the new accounts. Operator upgrades should be reviewed for SA changes.

**Operator-managed PostgreSQL**
The `ZabbixFull` CRD manages the PostgreSQL instance internally. There is no access to fine-tune PostgreSQL configuration, connection pooling, or backup schedules through this abstraction. For a demo this is acceptable; for production, an external PostgreSQL instance with proper backup integration would be required.

**No network policy**
All pods in `zabbix-demo` can communicate with each other and with cluster services without restriction. This is appropriate for a demo but would require network policies for any production or multi-tenant use case.
