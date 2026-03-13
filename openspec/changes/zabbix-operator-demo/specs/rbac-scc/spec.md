## ADDED Requirements

### Requirement: Default OpenShift SCCs Are Not Modified
The deployment SHALL NOT alter any default OpenShift SecurityContextConstraints. All SCC grants MUST be achieved through RBAC (Role and RoleBinding) without patching or overwriting built-in SCC objects.

#### Scenario: Default SCCs are unchanged after deployment
- **WHEN** all RBAC and SCC manifests have been applied via ArgoCD or kubectl
- **THEN** `oc get scc anyuid -o yaml` and `oc get scc nonroot -o yaml` MUST produce output identical to the pre-deployment state, with no modifications to `users`, `groups`, or any other field caused by this deployment

### Requirement: Role Grants "use" Verb on the Required SCC
A Role (or ClusterRole scoped via RoleBinding) SHALL be created in the zabbix-demo namespace that grants the `use` verb on the Security Context Constraint required by Zabbix workloads (anyuid or nonroot).

#### Scenario: Role with SCC use permission exists in zabbix-demo namespace
- **WHEN** the RBAC manifests are applied
- **THEN** a Role MUST exist in the zabbix-demo namespace containing a rule with `apiGroups: ["security.openshift.io"]`, `resources: ["securitycontextconstraints"]`, `resourceNames: ["anyuid"]` (or `"nonroot"`), and `verbs: ["use"]`

### Requirement: RoleBinding Binds the SCC Role to Zabbix Service Accounts
A RoleBinding SHALL exist in the zabbix-demo namespace that binds the SCC-granting Role to the ServiceAccount(s) used by Zabbix workload pods.

#### Scenario: RoleBinding references the correct Role and Zabbix ServiceAccounts
- **WHEN** the RBAC manifests are applied
- **THEN** a RoleBinding MUST exist in the zabbix-demo namespace with `roleRef` pointing to the SCC Role, and `subjects` MUST include every ServiceAccount that runs Zabbix Server, Web Frontend, Java Gateway, or PostgreSQL pods

### Requirement: RBAC Resources Are Scoped to zabbix-demo Namespace
All Role and RoleBinding resources created for this deployment SHALL have `namespace: zabbix-demo` so that the grants do not affect workloads in other namespaces.

#### Scenario: No ClusterRoleBinding is created for SCC grants
- **WHEN** the RBAC manifests are applied
- **THEN** no ClusterRoleBinding MUST exist that grants the `use` verb on any SCC to Zabbix ServiceAccounts, confirming the grants are namespace-scoped only

#### Scenario: Role and RoleBinding resources reside in zabbix-demo
- **WHEN** all RBAC resources created by this deployment are enumerated
- **THEN** every Role and RoleBinding MUST have `metadata.namespace: zabbix-demo` and MUST NOT appear in any other namespace

### Requirement: Zabbix Workloads Can Run After RBAC Grants Are Applied
After the Role and RoleBinding are created, the Zabbix workload pods SHALL start and reach the `Running` state without SCC-related admission failures.

#### Scenario: No SCC admission errors appear after RBAC is in place
- **WHEN** the Role and RoleBinding are applied before or alongside the ZabbixFull CR
- **THEN** no pod in the zabbix-demo namespace MUST fail admission with an error message containing `unable to validate against any security context constraint`, and all Zabbix pods MUST reach `Running` state
