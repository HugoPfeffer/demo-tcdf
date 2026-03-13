## ADDED Requirements

### Requirement: ArgoCD Application CR Points to the argocd/ Directory
The ArgoCD Application custom resource SHALL reference this repository and the `argocd/` directory as the source path, so that ArgoCD manages only the manifests located in that directory.

#### Scenario: Application source path is set to the argocd/ directory
- **WHEN** the ArgoCD Application CR is applied
- **THEN** `spec.source.path` MUST equal `argocd/` (or the equivalent relative path), and `spec.source.repoURL` MUST match the Git remote URL of this repository

### Requirement: Application Targets zabbix-demo Namespace
The ArgoCD Application SHALL target the `zabbix-demo` namespace as its destination, unless individual manifests override the namespace via their own `metadata.namespace` fields.

#### Scenario: Application destination namespace is zabbix-demo
- **WHEN** the ArgoCD Application CR is inspected
- **THEN** `spec.destination.namespace` MUST equal `zabbix-demo` and `spec.destination.server` MUST reference the local cluster (e.g., `https://kubernetes.default.svc`)

### Requirement: Sync Policy Is Automated with Self-Heal and Prune
The ArgoCD Application SHALL have an automated sync policy with both `selfHeal: true` and `prune: true` enabled, so that drift from Git is corrected automatically and removed resources are garbage-collected.

#### Scenario: Automated sync policy with selfHeal and prune is configured
- **WHEN** the ArgoCD Application CR is inspected
- **THEN** `spec.syncPolicy.automated.selfHeal` MUST be `true` and `spec.syncPolicy.automated.prune` MUST be `true`

#### Scenario: Manual drift is corrected automatically by ArgoCD
- **WHEN** a resource managed by the Application is manually deleted from the cluster
- **THEN** ArgoCD MUST re-create the deleted resource within one sync interval without any manual trigger, and the Application sync status MUST return to `Synced`

### Requirement: Sync Waves Enforce Correct Deployment Order
Resources in the `argocd/` directory SHALL use ArgoCD sync-wave annotations to enforce the following deployment order: (1) Namespace, (2) RBAC and ClusterIssuer, (3) ZabbixFull CR, (4) Route and sample-app Deployment.

#### Scenario: Namespace resource is deployed in wave 0 (first)
- **WHEN** the ArgoCD Application performs a sync
- **THEN** the Namespace resource MUST have `argocd.argoproj.io/sync-wave: "0"` (or the lowest wave value among all managed resources) and MUST reach `Healthy` status before resources in higher waves are applied

#### Scenario: RBAC and ClusterIssuer resources are deployed before ZabbixFull
- **WHEN** the ArgoCD Application performs a sync
- **THEN** all Role, RoleBinding, and ClusterIssuer resources MUST have a lower sync-wave value than the ZabbixFull CR, and ArgoCD MUST not begin applying the ZabbixFull CR until the lower-wave resources are `Healthy`

#### Scenario: ZabbixFull CR is deployed before Route and sample-app
- **WHEN** the ArgoCD Application performs a sync
- **THEN** the ZabbixFull CR MUST have a lower sync-wave value than the Route and sample-app Deployment, and ArgoCD MUST not begin applying the Route or sample-app until the ZabbixFull CR wave is `Healthy`

### Requirement: Single Sync Deploys All Resources
A single ArgoCD sync operation SHALL be sufficient to deploy all resources in the correct order, producing a fully operational stack without requiring additional manual steps.

#### Scenario: All resources reach Synced and Healthy after one sync
- **WHEN** the ArgoCD Application triggers a sync (automated or manual) against an empty zabbix-demo namespace
- **THEN** all managed resources MUST reach `Synced` status, the Application overall sync status MUST be `Synced`, and the Application health status MUST be `Healthy` within the expected reconciliation window

### Requirement: Application Health Reflects Aggregate Health of All Managed Resources
The ArgoCD Application health status SHALL reflect the aggregate health of every resource it manages, so that any unhealthy resource causes the Application to report a non-Healthy status.

#### Scenario: Application reports Degraded when a managed resource is unhealthy
- **WHEN** a managed resource (e.g., the ZabbixFull CR or a Deployment) enters a degraded or error state
- **THEN** the ArgoCD Application health status MUST change from `Healthy` to `Degraded` (or `Progressing`) within one ArgoCD reconciliation cycle, without any manual refresh

#### Scenario: Application reports Healthy only when all resources are healthy
- **WHEN** every resource managed by the Application reports a healthy status
- **THEN** the ArgoCD Application health status MUST be `Healthy` and the sync status MUST be `Synced`
