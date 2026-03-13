# Demo Objective: Zabbix Operator Integration on OpenShift

## Goal

Showcase the **Zabbix Operator** integration within OpenShift by deploying a Zabbix monitoring stack and a sample application with a connected Zabbix Agent. The Zabbix Dashboard must display the connected agent and its live metrics.

The entire deployment is managed through **ArgoCD** — all manifests live in the `argocd/` directory so ArgoCD can sync the demo into the cluster in a single operation.

---

## What the Demo Proves

1. The Zabbix Operator can deploy a fully functional Zabbix Server with a managed PostgreSQL database on OpenShift via the `ZabbixFull` CRD.
2. A containerized application running a Zabbix Agent can register with the Zabbix Server and report metrics.
3. The Zabbix Web Dashboard is accessible via an OpenShift Route (TLS-secured with cert-manager) and shows the connected agent with live metrics.
4. The full stack is GitOps-managed through ArgoCD.

---

## Scope

### In-Scope

| Component | Description |
|-----------|-------------|
| **Zabbix Server** | Deploy via `ZabbixFull` CR (Zabbix Operator). Includes server, web frontend, Java gateway, and managed PostgreSQL. |
| **Zabbix Web Dashboard** | Expose via OpenShift Route with TLS (cert-manager + openshift-routes controller). Demonstrate the connected agent and its metrics. |
| **Sample Application + Zabbix Agent** | Deploy a simple container/pod running a Zabbix Agent configured to connect to the Zabbix Server. The agent reports host metrics (CPU, memory, disk, network). |
| **Namespace** | `zabbix-demo` — all demo resources live here. |
| **ArgoCD Application** | An ArgoCD `Application` CR pointing to the `argocd/` directory in this repository. |
| **TLS** | Edge-terminated Route with automatic certificate via cert-manager `ClusterIssuer`. |
| **SCC / RBAC** | Any required SecurityContextConstraint grants and RBAC for the Zabbix workloads. |

### Out-of-Scope (Pre-existing on the Cluster)

These operators must already be installed and healthy before running this demo. Deploying them is **not** part of this demo:

| Operator | Expected Version | Purpose |
|----------|-----------------|---------|
| **Zabbix Operator** | 6.0.40 (certified) | Provides `ZabbixFull`, `ZabbixAgent`, and related CRDs |
| **ArgoCD Operator** (OpenShift GitOps) | v1.19.2 | Provides the ArgoCD instance that syncs the demo |
| **Cert-Manager Operator** | v1.18.1 | Provides certificate issuance for the Zabbix Web Route |

Also out-of-scope:
- OpenShift cluster provisioning and configuration
- Storage backend setup (ODF/Ceph is already available)
- OpenShift Virtualization / CNV setup
- Network infrastructure (DNS wildcard, load balancers)

---

## Target Cluster Environment

| Property | Value |
|----------|-------|
| **OCP Version** | 4.20.15 |
| **Platform** | Bare-metal with Red Hat CNV (SingleReplica) |
| **Default StorageClass** | `ocs-external-storagecluster-ceph-rbd` (Ceph RBD via ODF) |
| **Route Domain** | `*.apps.cluster-d45v2.dynamic.redhatworkshops.io` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenShift Cluster                         │
│                                                             │
│  ┌───────────────── zabbix-demo namespace ────────────────┐ │
│  │                                                        │ │
│  │  ┌──────────────┐    ZabbixFull CR    ┌─────────────┐  │ │
│  │  │ Zabbix       │◄──(Operator)──────►│ PostgreSQL   │  │ │
│  │  │ Server       │                     │ (managed)    │  │ │
│  │  │ + Web UI     │                     └─────────────┘  │ │
│  │  │ + Java GW    │                                      │ │
│  │  └──────┬───────┘                                      │ │
│  │         │ :10051 / :10050                               │ │
│  │         │ ClusterIP Service                             │ │
│  │         │                                               │ │
│  │  ┌──────┴───────┐                                      │ │
│  │  │ Sample App   │  Pod with Zabbix Agent                │ │
│  │  │ + Zabbix     │  → active checks to server:10051     │ │
│  │  │   Agent      │  → passive checks on agent:10050     │ │
│  │  └──────────────┘                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌─ Route (TLS edge) ──────────────────────────────────┐   │
│  │ zabbix-demo.apps.cluster-d45v2.dynamic...           │   │
│  │ → cert-manager auto-TLS                              │   │
│  │ → Zabbix Web UI Service :8080                        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ArgoCD syncs ← argocd/ directory (this repo)               │
└─────────────────────────────────────────────────────────────┘
```

---

## Repository Layout

```
argocd/
├── namespace.yaml              # zabbix-demo namespace
├── zabbix-full.yaml            # ZabbixFull CR (server + db + web)
├── zabbix-web-route.yaml       # Route for Zabbix Web UI (TLS via cert-manager)
├── cluster-issuer.yaml         # ClusterIssuer for TLS certificates (if not already present)
├── sample-app.yaml             # Deployment: simple app container with Zabbix Agent sidecar
├── rbac.yaml                   # SCC grants and RBAC for Zabbix workloads
└── argocd-application.yaml     # ArgoCD Application CR pointing to this directory
```

---

## Demo Flow

1. **ArgoCD Sync** — ArgoCD detects the manifests in `argocd/` and syncs them to the cluster.
2. **Namespace Created** — `zabbix-demo` namespace is provisioned.
3. **Zabbix Server Deploys** — The Zabbix Operator reconciles the `ZabbixFull` CR: provisions PostgreSQL, starts the Zabbix Server, web frontend, and Java gateway.
4. **Route Becomes Available** — The OpenShift Route exposes the Zabbix Web UI with a TLS certificate automatically issued by cert-manager.
5. **Sample App Deploys** — A pod with a Zabbix Agent starts, configured to connect to the Zabbix Server's ClusterIP Service.
6. **Agent Registers** — The Zabbix Agent connects to the server (active checks on port 10051). The agent appears in the Zabbix Dashboard.
7. **Metrics Visible** — The Zabbix Dashboard shows the connected agent and its host metrics (CPU, memory, disk, network).

---

## Success Criteria

- [ ] All pods in `zabbix-demo` are Running and healthy.
- [ ] Zabbix Web Dashboard is accessible via the TLS-secured Route.
- [ ] The sample application's Zabbix Agent appears as a connected host in the Dashboard.
- [ ] Live metrics (CPU, memory, disk, network) are visible for the connected agent.
- [ ] The entire deployment was performed through a single ArgoCD sync.

---

## Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Zabbix CRD | `ZabbixFull` | Self-contained: includes managed PostgreSQL — no external DB setup required. |
| Agent deployment | Sidecar in a pod (not KubeVirt VM) | Simpler and faster for a demo. Avoids CNV dependency for the agent. |
| TLS | Edge-terminated Route + cert-manager | Standard OpenShift pattern. Automatic certificate lifecycle. |
| Namespace | `zabbix-demo` | Isolated, disposable namespace for the demo. |
| StorageClass | `ocs-external-storagecluster-ceph-rbd` (cluster default) | Already available on the cluster. Suitable for PostgreSQL PVCs. |
| GitOps | ArgoCD Application | All manifests in `argocd/` — single sync deploys everything. |
