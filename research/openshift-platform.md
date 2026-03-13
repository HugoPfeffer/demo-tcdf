# OpenShift Container Platform — Official Reference

> **Scope**: OperatorHub, Operator Lifecycle Manager (OLM), namespace/project management, AWS EBS CSI Driver (gp3-csi StorageClass), Security Context Constraints (SCCs), and RBAC for operator workloads on OpenShift 4.x/AWS.

---

## 1. OperatorHub

### What It Is

OperatorHub is the **web console interface** in OpenShift for discovering and installing Operators. It is deployed **by default** in OpenShift 4.x. The UI component is driven by the Marketplace Operator running in `openshift-marketplace`.

### Catalog Categories

| Category | Description |
|----------|-------------|
| **Red Hat Operators** | Red Hat products, packaged and shipped by Red Hat. Supported by Red Hat. |
| **Certified Operators** | ISV products. Red Hat partners with ISVs to package and ship. Supported by the ISV. |
| **Red Hat Marketplace** | Certified software purchasable from Red Hat Marketplace. |
| **Community Operators** | Maintained in `redhat-openshift-ecosystem/community-operators-prod` GitHub repo. No official support. |
| **Custom Operators** | Operators you add to the cluster yourself. |

### How to Install Operators

**Via Web Console:**

1. Navigate to **Operators > OperatorHub**
2. Search/filter for the desired Operator
3. Click the Operator tile, review details
4. Click **Install**
5. Choose **Installation Mode** (all namespaces or specific namespace)
6. Choose **Update Channel** and **Approval Strategy** (Automatic or Manual)
7. Click **Subscribe/Install**

**Via CLI:**

```bash
# List available operators
oc get packagemanifests -n openshift-marketplace

# Create a Subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-operator
  namespace: my-namespace
spec:
  channel: <channel-name>
  name: <operator-package-name>
  source: certified-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

### OperatorHub Custom Resource

The Marketplace Operator manages an `OperatorHub` CR named `cluster` that controls default CatalogSources:

```yaml
apiVersion: config.openshift.io/v1
kind: OperatorHub
metadata:
  name: cluster
spec:
  disableAllDefaultSources: true  # For restricted networks
  sources:
    - name: "community-operators"
      disabled: false
```

---

## 2. Operator Lifecycle Manager (OLM)

### What It Is

OLM controls **installation, upgrade, and RBAC** of Operators. It is deployed by default in OpenShift 4.x and is part of the Operator Framework.

### Key CRDs

| Resource | Short Name | Description |
|----------|-----------|-------------|
| **ClusterServiceVersion** | `csv` | Application metadata: name, version, icon, required resources, RBAC, install strategies |
| **CatalogSource** | `catsrc` | Repository of CSVs, CRDs, and packages |
| **Subscription** | `sub` | Tracks a channel in a package, keeps CSVs up to date |
| **InstallPlan** | `ip` | Calculated list of resources needed for install/upgrade |
| **OperatorGroup** | `og` | Configures Operators in a namespace to watch CRs in target namespaces |
| **OperatorConditions** | — | Communication channel between OLM and the managed Operator |

### How It Works

1. **CatalogSource** provides the catalog of available Operators (index images)
2. User creates a **Subscription** (package, channel, source)
3. OLM generates an **InstallPlan** listing all required resources
4. Upon approval, OLM creates the **CSV** and deploys the Operator
5. OLM monitors the **Subscription** for updates and manages upgrades

### Operator Maturity Model (5 Phases)

1. Basic Install
2. Seamless Upgrades
3. Full Lifecycle (backup, restore)
4. Deep Insights (metrics, alerts)
5. Auto Pilot (auto-scaling, tuning, abnormality detection)

---

## 3. Namespace/Project Management

### Projects vs Namespaces

OpenShift **Projects** extend Kubernetes namespaces with:
- Additional metadata (display name, description)
- Built-in RBAC integration
- Network policy enforcement
- Resource quota integration
- Self-provisioning capabilities

A Project and Namespace are **1:1 mapped**. Creating a Project creates the underlying namespace.

### `oc new-project` vs `oc create namespace`

| Command | Behavior |
|---------|----------|
| `oc new-project <name>` | Creates Project + namespace, sets current context, applies default RBAC/templates/quotas. **Preferred.** |
| `oc create namespace <name>` | Creates bare K8s namespace. No RBAC, no templates, no context switch. |
| `oc adm new-project <name>` | Admin-only. Required for `openshift-*` or `kube-*` prefixed namespaces. |

### Important Restrictions

- Cannot use `oc new-project` for namespaces starting with `openshift-` or `kube-`
- Default projects (`default`, `kube-public`, `kube-system`, `openshift`, `openshift-infra`, `openshift-node`) are **highly privileged** — do NOT run workloads in these
- Admission plugins (pod security, SCC, quotas) **do not work** in highly privileged projects

### Creating `zabbix-system`

```bash
oc new-project zabbix-system \
    --description="Zabbix monitoring stack" \
    --display-name="Zabbix System"
```

Or via YAML:

```yaml
apiVersion: project.openshift.io/v1
kind: ProjectRequest
metadata:
  name: zabbix-system
  annotations:
    openshift.io/description: "Zabbix monitoring stack"
    openshift.io/display-name: "Zabbix System"
```

---

## 4. AWS EBS CSI Driver & gp3-csi StorageClass

### Overview

OpenShift 4.18+ installs the following **by default** in `openshift-cluster-csi-drivers`:
- **AWS EBS CSI Driver Operator** — manages the CSI driver and default StorageClass
- **AWS EBS CSI driver** — enables creating and mounting AWS EBS PVs

### gp3-csi StorageClass

- **Default for clusters installed with OCP 4.10+**: `gp3-csi` is the default StorageClass
- **Provisioner**: `ebs.csi.aws.com`
- Older clusters (pre-4.10) may have `gp2` as default with `kubernetes.io/aws-ebs` (in-tree)

**StorageClass Definition:**

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp3-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**With KMS encryption:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-csi-encrypted
parameters:
  fsType: ext4
  encrypted: "true"
  kmsKeyId: <ARN-of-KMS-key>
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### EBS Volume Types: gp2 vs gp3

| Feature | gp2 | gp3 |
|---------|-----|-----|
| **Baseline IOPS** | 3 IOPS/GiB (min 100, max 16,000) | 3,000 IOPS (regardless of size) |
| **Max IOPS** | 16,000 | 16,000 |
| **Baseline Throughput** | Tied to volume size (max 250 MiB/s) | 125 MiB/s baseline (any size) |
| **Max Throughput** | 250 MiB/s | 1,000 MiB/s |
| **Cost** | Higher (bundled) | ~20% cheaper (decoupled IOPS/throughput) |

**Key advantage**: A 10 GiB gp3 volume gets 3,000 IOPS, vs only 30 IOPS on gp2.

### Verifying StorageClass Availability

```bash
# List all StorageClasses
oc get storageclass

# Describe a specific StorageClass
oc describe storageclass gp3-csi

# Verify CSI driver pods
oc get pods -n openshift-cluster-csi-drivers

# Check ClusterCSIDriver status
oc get clustercsidriver ebs.csi.aws.com -o yaml
```

### Managing the Default StorageClass

```bash
# Remove default from current
oc patch storageclass gp3-csi -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'

# Set new default
oc patch storageclass my-custom-sc -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```

Via ClusterCSIDriver management:
- `spec.storageClassState: Managed` — operator creates and manages the default StorageClass
- `spec.storageClassState: Unmanaged` — operator does not reconcile (allows custom edits)
- `spec.storageClassState: Removed` — operator deletes the StorageClass

---

## 5. Security Context Constraints (SCCs)

SCCs control permissions for pods — what a container can and cannot do on the host.

### Default SCCs

| SCC | Description |
|-----|-------------|
| **restricted / restricted-v2** | Default for most pods. Denies host features, requires pre-allocated UID/SELinux/FSGroup. `restricted-v2` drops ALL capabilities, sets seccomp to `runtime/default`. |
| **nonroot / nonroot-v2** | Like restricted, but allows any non-root UID. |
| **anyuid** | Like restricted, but allows any UID/GID. |
| **privileged** | Full access. **Only for cluster administration.** |
| **hostnetwork / hostnetwork-v2** | Allows host networking/ports. |

> **Important**: Do NOT modify default SCCs. Create custom SCCs instead. Default SCCs are reset during cluster upgrades.

### RBAC for SCCs

```bash
# Create a role granting use of an SCC
oc create role scc-anyuid-role \
  --verb=use \
  --resource=scc \
  --resource-name=anyuid \
  -n zabbix-system

# Bind the role to a service account
oc create rolebinding scc-anyuid-binding \
  --role=scc-anyuid-role \
  --serviceaccount=zabbix-system:default \
  -n zabbix-system
```

**Role YAML:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: scc-anyuid-role
  namespace: zabbix-system
rules:
- apiGroups:
  - security.openshift.io
  resourceNames:
  - anyuid
  resources:
  - securitycontextconstraints
  verbs:
  - use
```

### For Zabbix Operator Workloads

- The Zabbix Operator and PostgreSQL pods may require **anyuid** or **nonroot** SCC
- Check the Operator's documentation for SCC requirements
- Grant the minimum-privilege SCC: prefer `restricted-v2` > `nonroot-v2` > `anyuid` > `privileged`

---

## 6. Prerequisites and Gotchas

1. **Cluster admin required** for installing operators and managing SCCs
2. **AWS IAM permissions**: EBS CSI Driver Operator requires appropriate IAM roles for EBS operations
3. **OLM must be running** (default on OCP 4.x): verify with `oc get pods -n openshift-operator-lifecycle-manager`
4. **Namespace must exist** before creating a Subscription targeting it
5. **Multiple default StorageClasses**: Having >1 default causes PVCs to get the most recently created one. Alert `MultipleDefaultStorageClasses` fires.
6. **gp2 vs gp3 on upgrades**: Clusters upgraded from pre-4.10 may still use `gp2`. Verify with `oc get sc`.
7. **SCC reset on upgrade**: Default SCCs are reset during cluster upgrades — always create custom SCCs
8. **OperatorGroup scope**: The `OperatorGroup` in a namespace defines the watch scope. For single-namespace operators, ensure it targets only that namespace.
9. **PVC WaitForFirstConsumer**: `gp3-csi` uses this binding mode. PVCs remain `Pending` until a pod is scheduled. This is expected.
10. **EBS AZ constraint**: EBS volumes are AZ-specific. Pod must be scheduled in the same AZ as the volume.
11. **Approval strategy**: Use `Manual` in production for change control over operator updates.

---

## 7. Official Documentation URLs

| Resource | URL |
|----------|-----|
| Understanding Operators (OCP 4.18) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/operators/understanding-operators |
| OperatorHub | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/operators/understanding-operators#olm-understanding-operatorhub |
| OLM Concepts | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/operators/understanding-operators#olm-understanding-olm |
| Installing Operators in Namespace | https://docs.openshift.com/container-platform/4.7/operators/user/olm-installing-operators-in-namespace.html |
| All Operators Docs (OCP 4.18) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/operators/index |
| Projects | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/building_applications/projects |
| CSI Storage (EBS) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/using-container-storage-interface-csi#persistent-storage-csi-ebs |
| Managing Default StorageClass | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/using-container-storage-interface-csi#persistent-storage-csi-sc-manage |
| Switch Default StorageClass (KCS) | https://access.redhat.com/solutions/6779501 |
| Managing SCCs | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/managing-pod-security-policies |
| Auth & Authorization | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/authentication_and_authorization/index |
| OCP 4.18 Landing Page | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/ |
| OLM Framework Upstream | https://olm.operatorframework.io/docs/concepts/operators-on-cluster/ |
| AWS EBS CSI Driver (GitHub) | https://github.com/openshift/aws-ebs-csi-driver |
| OCP CLI Tools | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/cli_tools/openshift-cli-oc |
