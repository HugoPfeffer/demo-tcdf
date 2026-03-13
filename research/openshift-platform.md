# OpenShift Container Platform — Official Reference

> **Scope**: OperatorHub, Operator Lifecycle Manager (OLM), namespace/project management, OpenShift Data Foundation (ODF) with Ceph RBD storage, Security Context Constraints (SCCs), and RBAC for operator workloads on OpenShift 4.x (bare-metal with Red Hat CNV).

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

## 1.1 Cluster Environment Context

> **Single source of truth** for the target cluster. All other research files reference these facts.

| Property | Value |
|----------|-------|
| **OCP Version** | 4.20.15 |
| **Platform** | `None` (bare-metal with Red Hat CNV / Container-Native Virtualization) |
| **Topology** | SingleReplica (1 node: control-plane + master + worker) |
| **Node** | `control-plane-cluster-d45v2-1` |
| **OS / Kernel** | RHCOS 9.6 / 5.14.0-570 |
| **Container Runtime** | cri-o 1.33.9 |
| **Internal IP** | 10.10.10.10 |
| **Cluster Name** | `cluster-d45v2` |
| **Infrastructure Name** | `cluster-d45v2-9xb6p` |
| **Cluster API** | `https://api.cluster-d45v2.dynamic.redhatworkshops.io:6443` |
| **Route Domain** | `*.apps.cluster-d45v2.dynamic.redhatworkshops.io` |
| **Default StorageClass** | `ocs-external-storagecluster-ceph-rbd` (Ceph RBD) |
| **Storage Backend** | OpenShift Data Foundation (ODF) v4.20.7 — external Ceph cluster, CephCSI operator v4.20.7 |
| **Virtualization platform** | Red Hat OpenShift Virtualization (CNV) / KubeVirt — hosts KubeVirt VirtualMachines alongside container workloads. The RHEL 9 Zabbix Agent VM runs on the same cluster and pod network as the Zabbix Server. |

### Installed Operators

| Operator | Version |
|----------|---------|
| cert-manager | v1.18.1 |
| OpenShift GitOps (ArgoCD) | v1.19.2 |
| Keycloak (RHBK) | v26.4.10 |
| OpenShift Data Foundation (ODF) | v4.20.7 |
| OpenShift Virtualization (CNV) | — for running RHEL 9 VMs on the cluster |
| OpenShift Lightspeed | v1.0.10 |

### StorageClasses

| StorageClass | Provisioner | Binding Mode | Default | Use Case |
|-------------|-------------|-------------|---------|----------|
| `ocs-external-storagecluster-ceph-rbd` | `openshift-storage.rbd.csi.ceph.com` | WaitForFirstConsumer | **Yes** | Block storage for databases, stateful workloads |
| `ocs-external-storagecluster-ceph-rbd-immediate` | `openshift-storage.rbd.csi.ceph.com` | Immediate | No | Block storage when pre-binding is needed |
| `ocs-external-storagecluster-cephfs` | `openshift-storage.cephfs.csi.ceph.com` | — | No | Shared filesystem (ReadWriteMany) |
| `openshift-storage.noobaa.io` | `openshift-storage.noobaa.io` | — | No | S3-compatible object storage |

> **Storage is provided by ODF/Ceph only. No cloud provider storage drivers exist on this cluster.**

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

## 4. OpenShift Data Foundation & Ceph RBD Storage

### Overview

This cluster uses **OpenShift Data Foundation (ODF) v4.20.7** in **external mode**, connecting to a pre-existing Ceph cluster rather than deploying Ceph pods locally. ODF provides software-defined storage via Ceph, exposing block (RBD), filesystem (CephFS), and object (NooBaa/S3) storage through CSI drivers.

Key components:
- **ODF Operator** — manages the StorageCluster CR and CSI driver deployments
- **CephCSI Operator v4.20.7** — manages Ceph CSI plugins (RBD + CephFS)
- **External Ceph cluster** — the actual storage backend; ODF connects to it via RADOS credentials

### Default StorageClass: `ocs-external-storagecluster-ceph-rbd`

- **Provisioner**: `openshift-storage.rbd.csi.ceph.com`
- **Binding Mode**: `WaitForFirstConsumer` — PVCs stay `Pending` until a pod is scheduled
- **Volume Expansion**: enabled (`allowVolumeExpansion: true`)
- **Recommended for**: database workloads (PostgreSQL, Zabbix), stateful applications requiring dedicated block devices

**StorageClass Definition:**

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ocs-external-storagecluster-ceph-rbd
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: openshift-storage.rbd.csi.ceph.com
parameters:
  clusterID: openshift-storage
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: openshift-storage
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: openshift-storage
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: openshift-storage
  imageFeatures: layering
  imageFormat: "2"
  pool: ceph-rbd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### Available StorageClasses

| StorageClass | Type | Binding Mode | Access Modes | Best For |
|-------------|------|-------------|-------------|----------|
| `ocs-external-storagecluster-ceph-rbd` (default) | Block (RBD) | WaitForFirstConsumer | RWO | Databases, single-pod stateful workloads |
| `ocs-external-storagecluster-ceph-rbd-immediate` | Block (RBD) | Immediate | RWO | Pre-provisioned volumes, batch jobs |
| `ocs-external-storagecluster-cephfs` | Filesystem (CephFS) | Immediate | RWO, RWX | Shared storage across multiple pods |
| `openshift-storage.noobaa.io` | Object (S3) | — | — | S3-compatible object storage via ObjectBucketClaim |

### Ceph RBD vs CephFS

| Feature | Ceph RBD (Block) | CephFS (Filesystem) |
|---------|------------------|---------------------|
| **Access Mode** | ReadWriteOnce (RWO) — single pod | ReadWriteMany (RWX) — multiple pods |
| **Protocol** | RADOS Block Device mapped to node | FUSE or kernel CephFS mount |
| **Performance** | Higher IOPS, lower latency — optimized for databases | Good throughput for shared reads/writes |
| **Snapshots** | Supported via CSI VolumeSnapshot | Supported via CSI VolumeSnapshot |
| **Volume Expansion** | Online expansion supported | Online expansion supported |
| **Use Case** | PostgreSQL, MySQL, single-writer stateful apps | Shared config, logs, media, CMS content |
| **Default SC on cluster** | `ocs-external-storagecluster-ceph-rbd` (**yes**) | `ocs-external-storagecluster-cephfs` |

**Key difference from cloud block storage**: Ceph RBD is also block storage, but it is **network-attached** (RADOS protocol over the cluster network) rather than tied to a specific cloud AZ. This means volumes are not constrained to a single availability zone and can be accessed from any node that has network connectivity to the Ceph cluster.

### Verifying StorageClass Availability

```bash
# List all StorageClasses
oc get storageclass

# Describe the default StorageClass
oc describe storageclass ocs-external-storagecluster-ceph-rbd

# Verify Ceph CSI driver pods
oc get pods -n openshift-storage -l app=csi-rbdplugin
oc get pods -n openshift-storage -l app=csi-rbdplugin-provisioner

# Check ODF StorageCluster status
oc get storagecluster -n openshift-storage

# Verify external Ceph connection health
oc get cephcluster -n openshift-storage -o jsonpath='{.items[0].status.ceph.health}'
```

### Managing the Default StorageClass

```bash
# Remove default from current
oc patch storageclass ocs-external-storagecluster-ceph-rbd \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'

# Set new default
oc patch storageclass my-custom-sc \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```

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
2. **OLM must be running** (default on OCP 4.x): verify with `oc get pods -n openshift-operator-lifecycle-manager`
3. **Namespace must exist** before creating a Subscription targeting it
4. **Multiple default StorageClasses**: Having >1 default causes PVCs to get the most recently created one. Alert `MultipleDefaultStorageClasses` fires.
5. **SCC reset on upgrade**: Default SCCs are reset during cluster upgrades — always create custom SCCs
6. **OperatorGroup scope**: The `OperatorGroup` in a namespace defines the watch scope. For single-namespace operators, ensure it targets only that namespace.
7. **PVC WaitForFirstConsumer**: `ocs-external-storagecluster-ceph-rbd` uses this binding mode. PVCs remain `Pending` until a pod is scheduled. This is expected — use `ocs-external-storagecluster-ceph-rbd-immediate` if you need pre-binding.
8. **Approval strategy**: Use `Manual` in production for change control over operator updates.
9. **External Ceph connectivity**: ODF in external mode depends on network connectivity to the Ceph cluster. If the external Ceph cluster is unreachable, all RBD and CephFS provisioning fails. Monitor with `oc get cephcluster -n openshift-storage`.
10. **Ceph health checks**: Before deploying storage-dependent workloads, verify Ceph health: `oc get cephcluster -n openshift-storage -o jsonpath='{.items[0].status.ceph.health}'` should return `HEALTH_OK`.
11. **RBD vs CephFS for workload types**: Use Ceph RBD (default SC) for databases and single-writer workloads requiring low latency. Use CephFS only when ReadWriteMany access is required (shared filesystems across pods). Do not use CephFS for database WAL/data directories.
12. **SingleReplica topology**: This cluster has a single node. There is no HA for the control plane or workloads. Pod eviction and rescheduling will not help during node failure.
13. **Ceph RBD image features**: The default StorageClass uses `imageFeatures: layering`. Ensure Ceph cluster supports the required features; mismatches cause mount failures.
14. **OpenShift Virtualization prerequisite**: The OpenShift Virtualization operator must be installed before creating VirtualMachine CRs. Install via OperatorHub (search for "OpenShift Virtualization").
15. **KubeVirt masquerade networking**: KubeVirt VMs using `masquerade` networking are on the pod network and can reach ClusterIP Services directly — no external load balancer needed for intra-cluster VM-to-pod communication.

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
| Red Hat OpenShift Data Foundation | https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/ |
| ODF 4.18 — Managing Storage | https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.18/html/managing_and_allocating_storage_resources/index |
| ODF — External Mode | https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.18/html/deploying_openshift_data_foundation_in_external_mode/index |
| OCP Storage — CSI Overview | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/using-container-storage-interface-csi |
| Managing Default StorageClass | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/using-container-storage-interface-csi#persistent-storage-csi-sc-manage |
| Switch Default StorageClass (KCS) | https://access.redhat.com/solutions/6779501 |
| Managing SCCs | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/managing-pod-security-policies |
| Auth & Authorization | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/authentication_and_authorization/index |
| OCP 4.18 Landing Page | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/ |
| OLM Framework Upstream | https://olm.operatorframework.io/docs/concepts/operators-on-cluster/ |
| OCP CLI Tools | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/cli_tools/openshift-cli-oc |
