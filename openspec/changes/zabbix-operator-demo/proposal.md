## Why

We need a turnkey demo that proves the Zabbix Operator can deploy and manage a complete monitoring stack on OpenShift — server, database, web UI, and a connected agent reporting live metrics — all driven by a single ArgoCD sync. This gives the team a reproducible, GitOps-managed showcase for customer engagements on the existing workshop cluster.

## What Changes

- Create all Kubernetes/OpenShift manifests in `argocd/` to deploy the Zabbix monitoring stack:
  - `namespace.yaml` — `zabbix-demo` namespace
  - `zabbix-full.yaml` — `ZabbixFull` CR (server + PostgreSQL + web frontend + Java gateway)
  - `zabbix-web-route.yaml` — TLS-terminated OpenShift Route via cert-manager
  - `cluster-issuer.yaml` — Self-signed `ClusterIssuer` for certificate issuance
  - `sample-app.yaml` — Deployment with a Zabbix Agent sidecar reporting host metrics
  - `rbac.yaml` — SCC grants and RBAC for Zabbix workloads
  - `argocd-application.yaml` — ArgoCD Application CR pointing to `argocd/`

## Capabilities

### New Capabilities

- `zabbix-server-stack`: Deploy the Zabbix Server, managed PostgreSQL, web frontend, and Java gateway via the `ZabbixFull` CRD.
- `tls-route`: Expose the Zabbix Web UI through an edge-terminated OpenShift Route with automatic TLS via cert-manager and a self-signed ClusterIssuer.
- `zabbix-agent-app`: Deploy a sample application pod with a Zabbix Agent sidecar that connects to the server and reports CPU, memory, disk, and network metrics.
- `rbac-scc`: Grant required SecurityContextConstraints and RBAC permissions for all Zabbix workloads in the `zabbix-demo` namespace.
- `argocd-sync`: Define an ArgoCD Application CR that syncs the entire `argocd/` directory, deploying the full stack in one operation.

### Modified Capabilities

_(none — this is a greenfield demo)_

## Impact

- **New files**: 7 YAML manifests in `argocd/` (listed above).
- **Cluster resources**: Creates a new `zabbix-demo` namespace with Deployments, Services, PVCs (Ceph RBD), Route, Certificate, and RBAC objects.
- **Dependencies**: Requires pre-installed Zabbix Operator 6.0.40, OpenShift GitOps v1.19.2, and cert-manager v1.18.1. No new operator installs.
- **Non-goals**: This demo does not cover HA, production hardening, external database setup, CNV/VM-based agents, custom Zabbix templates, or alerting configuration.
