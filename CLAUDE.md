# Zabbix Operator Demo on OpenShift

Demo project showcasing Zabbix Operator integration on OpenShift, deployed entirely via ArgoCD GitOps. All manifests live in `argocd/` and are synced by a single ArgoCD Application.

## Development Philosophy

This is a **demo project** — prioritize simplicity and fast deployment over production-grade patterns. Keep manifests minimal, avoid over-engineering, and favor working quickly over perfection. When in doubt, choose the simpler approach.

## Target Cluster

| Property | Value |
|----------|-------|
| OCP Version | 4.20.15 |
| Platform | Bare-metal SingleReplica with Red Hat CNV |
| Route Domain | `*.apps.cluster-d45v2.dynamic.redhatworkshops.io` |
| Default StorageClass | `ocs-external-storagecluster-ceph-rbd` (Ceph RBD via ODF) |
| Namespace | `zabbix-demo` |

## Pre-installed Operators (DO NOT deploy these)

- **Zabbix Operator** 6.0.44 (certified) — provides `ZabbixFull`, `ZabbixAgent` CRDs (API group: `kubernetes.zabbix.com`)
- **OpenShift GitOps** v1.19.2 (ArgoCD) — provides `Application` CRD
- **cert-manager** v1.18.1 + `openshift-routes` controller (pre-installed via Helm)

## Repository Layout

```
argocd/                         # ALL manifests go here — ArgoCD syncs this directory
├── namespace.yaml              # v1/Namespace (sync-wave 0)
├── rbac.yaml                   # Role + RoleBinding for SCC grants (sync-wave 1)
├── cluster-issuer.yaml         # cert-manager ClusterIssuer (sync-wave 1)
├── zabbix-mysql-resources.yaml # MySQL Secret + PVC for ZabbixFull (sync-wave 1)
├── zabbix-full.yaml            # ZabbixFull CR (sync-wave 2)
├── zabbix-web-route.yaml       # Route with cert-manager TLS (sync-wave 3)
├── sample-app.yaml             # nginx + load-generator + Zabbix Agent sidecar (sync-wave 3)
└── argocd-application.yaml     # ArgoCD Application CR (sync-wave 4)
research/                       # Background research docs (read-only reference)
openspec/                       # OpenSpec change tracking
```

## Manifest Authoring Rules

### Sync Wave Ordering (MUST follow)

| Wave | Resources | Why |
|------|-----------|-----|
| 0 | Namespace | Must exist before any namespaced resources |
| 1 | RBAC, ClusterIssuer, MySQL Secret+PVC | SCC grants needed before operator workloads; issuer needed before Route; MySQL resources needed before ZabbixFull |
| 2 | ZabbixFull CR | Requires namespace + SCC grants to be in place |
| 3 | Route, Sample App | Route needs ClusterIssuer (wave 1) + web Service (wave 2); app needs server Service (wave 2) |
| 4 | ArgoCD Application | Ties everything together last |

Every manifest MUST have the annotation `argocd.argoproj.io/sync-wave: "<N>"`.

### API Versions and Kinds

Use these exact API versions — do not guess or use deprecated versions:

- Namespace: `v1`
- Role/RoleBinding: `rbac.authorization.k8s.io/v1`
- ClusterIssuer: `cert-manager.io/v1`
- ZabbixFull: `kubernetes.zabbix.com/v1alpha1`
- Route: `route.openshift.io/v1`
- Deployment: `apps/v1`
- Service: `v1`
- ArgoCD Application: `argoproj.io/v1alpha1`

### Namespace Targeting

- All namespaced resources go in `zabbix-demo` EXCEPT `argocd-application.yaml` which goes in `openshift-gitops`
- ClusterIssuer is cluster-scoped (no namespace)

## Critical Constraints and Gotchas

### SCC / RBAC
- **NEVER modify default OpenShift SCCs** (e.g., `anyuid`, `restricted`). Instead, create a namespaced `Role` granting `use` on the specific SCC and bind it to the relevant ServiceAccount(s).
- The Zabbix Operator may create its own ServiceAccounts — the RoleBinding must reference them by name.

### Zabbix Agent Hostname
- The `Hostname` env var on the Zabbix Agent sidecar **must exactly match** the host record configured in the Zabbix frontend. Mismatch causes silent failure of active checks.

### PVC Binding Behavior
- `ocs-external-storagecluster-ceph-rbd` uses `WaitForFirstConsumer` binding mode. PVCs will stay `Pending` until the pod is scheduled — this is expected, not an error.
- Always set `storageClassName: ocs-external-storagecluster-ceph-rbd` explicitly on MySQL PVCs.

### TLS / Route
- Route uses `spec.tls.termination: edge` (TLS terminates at the router, HTTP to pod).
- cert-manager annotations on the Route: `cert-manager.io/issuer-kind: ClusterIssuer` and `cert-manager.io/issuer-name: <issuer-name>`.
- Do NOT put static TLS certificates in manifests — cert-manager manages certificate lifecycle.
- The `openshift-routes` controller must be pre-installed for Route annotations to work.

### ZabbixFull CRD
- Use `kind: ZabbixFull` (NOT `ZabbixServer`) — it includes a managed MySQL database.
- API group is `kubernetes.zabbix.com/v1alpha1` (NOT `zabbix.com/v1alpha1`).
- CRD full name: `zabbixfulls.kubernetes.zabbix.com`
- **Required fields**: `spec.web_size`, `spec.java_gateway_size`, `spec.zabbix_mysql_volumeclaim` (PVC name), `spec.zabbix_mysqlsecret` (Secret name), `spec.web.server_name`, `spec.web.timezone`
- The MySQL Secret must contain keys: `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`
- The PVC must be pre-created and referenced by name in `zabbix_mysql_volumeclaim`
- The CRD is proprietary — inspect on-cluster with `oc describe crd zabbixfulls.kubernetes.zabbix.com`
- Operator supports `web_enable_route: true` to auto-create a Route (but we use a custom Route with cert-manager annotations instead)
- Known issue ZBX-26152: compatibility issues between Zabbix Operator v6.0.38 and OpenShift Virtualization (CNV). Verify resolved in current version.

### Zabbix Web Dashboard
- Default credentials: `Admin` (capital A) / `zabbix`
- Web UI served on port 8080 via the operator-created Service
- Agent host registration: manual via Configuration > Hosts, or auto-registration via Actions
- Recommended template: "Linux by Zabbix agent active" for sidecar agents

### ArgoCD Application
- `spec.syncPolicy.automated` with `prune: true` and `selfHeal: true`
- `CreateNamespace=false` — namespace is managed by `namespace.yaml`, not ArgoCD
- Source path: `argocd/`

## Zabbix Agent Sidecar Config

The sample app uses a `zabbix/zabbix-agent2` sidecar with these critical env vars:

| Env Var | Value | Purpose |
|---------|-------|---------|
| `ZBX_SERVER_HOST` | `zabbix-server.zabbix-demo.svc.cluster.local` | Server address for passive checks |
| `ZBX_ACTIVESERVERS` | `zabbix-server.zabbix-demo.svc.cluster.local:10051` | Server address for active checks |
| `ZBX_HOSTNAME` | `sample-app-agent` | Must match Zabbix frontend host record |
| `ZBX_METADATAITEM` | `system.uname` | Enables auto-registration with host metadata |

Ports: 10051 (active checks, agent → server), 10050 (passive checks, server → agent).

## Verification Commands

```bash
# Namespace
oc get namespace zabbix-demo

# RBAC
oc get role,rolebinding -n zabbix-demo

# ClusterIssuer
oc get clusterissuer -o wide   # Should show Ready: True

# Zabbix pods
oc get pods -n zabbix-demo     # All should be Running

# Services
oc get svc -n zabbix-demo      # Should show ports 10051, 10050, 8080

# Route
oc get route -n zabbix-demo
curl -kI https://zabbix-demo.apps.cluster-d45v2.dynamic.redhatworkshops.io

# Sample app agent
oc get pods -n zabbix-demo -l app=sample-app   # 3/3 containers ready (nginx + load-gen + agent)
oc logs -n zabbix-demo -l app=sample-app -c zabbix-agent2

# ArgoCD
oc get application -n openshift-gitops          # Synced and Healthy
```

## Documentation Validation with Sub-agents

When writing manifests for CRDs or operator features where you are unsure about the exact spec fields, annotations, or behavior — **use a sub-agent** to validate against official documentation before committing to a particular approach.

### When to use sub-agents for docs validation
- Unsure about CRD spec fields (e.g., ZabbixFull spec structure, cert-manager annotation names)
- Need to verify operator behavior or version-specific features
- Checking OpenShift Route or ArgoCD Application configuration options
- Validating Zabbix Agent environment variable names or defaults

### How to validate
Launch a sub-agent (general-purpose or Explore type) and instruct it to:
1. Use the **Firecrawl MCP** tools (`firecrawl_scrape`, `firecrawl_search`) to fetch and search official documentation pages
2. Use the **WebSearch** tool to find current official docs or community references
3. Cross-reference findings with the research docs in `research/` directory

### Key documentation sources to check
- Zabbix OpenShift docs: `https://www.zabbix.com/documentation/6.0/en/manual/installation/containers/openshift`
- cert-manager OpenShift Routes: `https://cert-manager.io/docs/usage/openshift/`
- ArgoCD sync waves: `https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/`
- OpenShift Route API: `https://docs.openshift.com/container-platform/4.20/networking/routes/route-configuration.html`

### Example sub-agent prompt
> "Use firecrawl_scrape to fetch the Zabbix Operator OpenShift documentation at [URL] and extract the ZabbixFull CRD spec fields. Also use WebSearch to find any community examples of ZabbixFull CR YAML. Report back the confirmed spec structure."

## Non-Goals (Out of Scope)

- High availability or replica counts > 1
- Production hardening (network policies, resource limits, secrets management)
- External or self-managed MySQL/PostgreSQL
- CNV/VM-based Zabbix Agent (agent runs as sidecar only)
- Custom Zabbix templates, triggers, or alerting
- Multi-cluster or federated monitoring
- Deploying operators or cluster infrastructure
