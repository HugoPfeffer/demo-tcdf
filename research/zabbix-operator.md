# Zabbix Operator for Kubernetes/OpenShift — Official Reference

> **Scope**: Zabbix Operator installation via OperatorHub, CRD definitions (ZabbixFull, ZabbixServer, ZabbixProxy, ZabbixAgent), PostgreSQL storage configuration, and deployment on OpenShift 4.x.

---

## 1. Official Zabbix Operator Overview

- **Provider**: Zabbix SIA (official vendor)
- **Type**: Red Hat Certified Operator — available via OperatorHub on OpenShift
- **Packaging**: Ansible-based Kubernetes operator
- **Privilege mode**: Unprivileged
- **Category**: Monitoring
- **Latest certified bundle version**: 6.0.40
- **Source code availability**: The official operator is **proprietary/closed-source**. No public GitHub repo exists for the certified operator. The CRD schema is embedded in the operator bundle.

---

## 2. Cluster Environment Context

This reference targets a **live OpenShift cluster** with the following ground-truth facts:

| Attribute | Value |
|-----------|-------|
| **OCP version** | 4.20.15 |
| **Platform** | `None` (bare-metal with Red Hat CNV) |
| **Topology** | SingleReplica (1 node: control-plane + master + worker) |
| **Node** | `control-plane-cluster-d45v2-1`, RHCOS 9.6, kernel 5.14.0-570, cri-o 1.33.9 |
| **Default StorageClass** | `ocs-external-storagecluster-ceph-rbd` (ODF/Ceph RBD) |
| **Storage backend** | OpenShift Data Foundation (ODF) v4.20.7 with external Ceph cluster |
| **Zabbix Operator** | Not yet installed (pre-deployment reference) |
| **Route domain** | `*.apps.cluster-d45v2.dynamic.redhatworkshops.io` |
| **Cluster API** | `https://api.cluster-d45v2.dynamic.redhatworkshops.io:6443` |
| **VM Workloads** | The RHEL 9 Zabbix Agent host is a KubeVirt VirtualMachine running on the same cluster, sharing the pod network with the Zabbix Server pods |

> **Important**: This is a **bare-metal environment with Red Hat CNV**. Storage is provided by **external Ceph via ODF** (Ceph RBD). Use `ocs-external-storagecluster-ceph-rbd` for ZabbixFull database PVCs.

---

## 3. Operands (Custom Resource Definitions)

The Zabbix Operator provides **four** operand types:

| Operand / Kind | API Group | Description |
|----------------|-----------|-------------|
| **ZabbixFull** | `zabbix.com/v1alpha1` | Complete Zabbix installation: server + web UI + Java gateway + **managed PostgreSQL database**. Self-contained — includes DB provisioning. |
| **ZabbixServer** | `zabbix.com/v1alpha1` | Zabbix server + web UI + Java gateway with **MySQL** database support. Does **NOT** provision MySQL — requires an **external MySQL database**. |
| **ZabbixProxy** | `zabbix.com/v1alpha1` | Standalone Zabbix proxy deployment. |
| **ZabbixAgent** | `zabbix.com/v1alpha1` | Deploys Zabbix agents as a **DaemonSet** across cluster nodes. |

### Critical Distinction: ZabbixFull vs ZabbixServer

- **ZabbixFull** = Use this when you want the operator to manage PostgreSQL automatically (self-contained)
- **ZabbixServer** = Use this when you have an external MySQL database already provisioned

> **Important for our demo**: The spec references `kind: ZabbixServer` with `database.type: postgresql`. The official `ZabbixServer` operand historically uses MySQL. If operator-managed PostgreSQL is desired, the correct kind is likely **`ZabbixFull`**. Verify against the actual CRD installed on your cluster after operator installation.

---

## 4. ZabbixFull / ZabbixServer CRD Spec Fields

### Known Spec Fields

| Field Path | Type | Description |
|-----------|------|-------------|
| `spec.replicas` | integer | Number of Zabbix server replicas |
| `spec.database.type` | string | Database backend: `postgresql` or `mysql` |
| `spec.database.storage.className` | string | Kubernetes StorageClass name (e.g., `ocs-external-storagecluster-ceph-rbd`) |
| `spec.database.storage.size` | string | PVC size (e.g., `10Gi`) |
| `spec.web.replicas` | integer | Number of web frontend replicas |
| `spec.web.service.type` | string | Service type: `ClusterIP`, `NodePort`, `LoadBalancer` |
| `spec.start_ipmi_pollers` | integer | Number of IPMI pollers |
| `spec.metadata_item` | string | Host metadata item for agent auto-registration (e.g., `system.uname`) |
| `spec.server_host` | string | Zabbix server address (for agent/proxy CRs) |

### Discovering All Fields on Your Cluster

After installing the operator, inspect the full CRD:

```bash
oc get crd | grep zabbix
oc describe crd zabbixfulls.zabbix.com
oc describe crd zabbixservers.zabbix.com
oc get crd zabbixfulls.zabbix.com -o yaml
```

### Example ZabbixFull CR

```yaml
apiVersion: zabbix.com/v1alpha1
kind: ZabbixFull
metadata:
  name: zabbix-demo
  namespace: zabbix-system
spec:
  replicas: 1
  database:
    type: postgresql
    storage:
      className: ocs-external-storagecluster-ceph-rbd
      size: 10Gi
  web:
    replicas: 1
    service:
      type: ClusterIP
```

---

## 5. Installation on OpenShift via OperatorHub

### Prerequisites

- OpenShift 4.x cluster (tested with 4.5+)
- Cluster admin privileges
- For `ZabbixFull`: a StorageClass that supports dynamic provisioning. On this cluster, use `ocs-external-storagecluster-ceph-rbd` (ODF/Ceph RBD). The cluster uses OpenShift Data Foundation with external Ceph.
- For `ZabbixServer`: an external MySQL database must already exist

### Installation Steps (Web Console)

1. Navigate to **Operators > OperatorHub** in the OpenShift Console
2. Search for **"Zabbix"**
3. Select **"Zabbix Operator"** (certified operator tile)
4. Click **Install**
5. Configure:
   - **Update channel**: select version
   - **Installation mode**: specific namespace (e.g., `zabbix-system`) or all namespaces
   - **Approval strategy**: Automatic or Manual
6. Click **Install** to begin
7. Verify at **Operators > Installed Operators** — status should show "Succeeded"
8. Click on **Zabbix Operator**, select the desired operand tab (Zabbix Full, Zabbix Server, etc.)
9. Click **Create**, switch to YAML view, configure the CR, and click **Create**

### Installation Steps (CLI)

```bash
# List available Zabbix packages
oc get packagemanifests -n openshift-marketplace | grep -i zabbix

# Create a Subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: zabbix-operator
  namespace: zabbix-system
spec:
  channel: <channel-name>
  name: zabbix-operator
  source: certified-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

---

## 6. PostgreSQL Configuration

### Managed PostgreSQL (ZabbixFull)

- PostgreSQL is provisioned **automatically** as a StatefulSet with a PersistentVolumeClaim
- Storage class and size are configurable via the CR spec under `database.storage`
- The operator manages the DB lifecycle (provisioning, configuration)

### Zabbix 7.0 Database Encryption Parameters

| Parameter | Description |
|-----------|-------------|
| `DBHost` | PostgreSQL server hostname |
| `DBName` | Database name (default: `zabbix`) |
| `DBUser` | Database username |
| `DBPassword` | Database password |
| `DBTLSConnect` | TLS mode: `required`, `verify_ca`, `verify_full` |
| `DBTLSCAFile` | Path to CA certificate file |

### Web Frontend PostgreSQL Config (PHP)

```php
$DB['ENCRYPTION'] = true;
$DB['KEY_FILE'] = '<key_file_path>';
$DB['CERT_FILE'] = '<cert_file_path>';
$DB['CA_FILE'] = '<ca_file_path>';
$DB['VERIFY_HOST'] = true;
```

---

## 7. ZabbixAgent DaemonSet

When deploying the `ZabbixAgent` operand:

- Agents deploy as a **DaemonSet** across all worker nodes
- To monitor **master/control-plane nodes**, add a toleration:

```yaml
tolerations:
  - key: node-role.kubernetes.io/master
    operator: Exists
    effect: NoSchedule
```

---

## 8. Version Information

| Component | Version |
|-----------|---------|
| Latest certified operator bundle | 6.0.40 |
| Zabbix 7.0 | LTS release (supported until June 2029) |
| Zabbix 7.4 | Current release |
| Zabbix 8.0 | Development |
| OpenShift tested versions | 4.5.x, 4.6.x+, 4.20.x |

---

## 9. Known Issues and Caveats

1. **Configuration persistence**: Some `zabbix_server.conf` parameters set in the CR spec may be **reset on pod restart** due to the operator's reconciliation loop overwriting manual changes.

2. **OpenShift Virtualization conflict**: Zabbix support ticket ZBX-26152 reports compatibility issues between the Zabbix Operator (v6.0.38) and OpenShift Virtualization. **Since our cluster runs CNV, verify this is resolved in the current operator version before deploying.**

3. **StorageClass must exist**: Ensure the target StorageClass exists and supports dynamic provisioning before deploying the CR. On this cluster, use `ocs-external-storagecluster-ceph-rbd` (ODF Ceph RBD).

4. **Helm Chart alternative**: For Kubernetes (non-OpenShift), Zabbix also provides a Helm chart via `zabbix/zabbix-docker` which gives more granular control.

---

## 10. Official Documentation URLs

| Resource | URL |
|----------|-----|
| Zabbix OpenShift Docs (7.0) | https://www.zabbix.com/documentation/7.0/en/manual/installation/containers/openshift |
| Zabbix OpenShift Docs (current/7.4) | https://www.zabbix.com/documentation/current/en/manual/installation/containers/openshift |
| Zabbix OpenShift Docs (6.0) | https://www.zabbix.com/documentation/6.0/en/manual/installation/containers/openshift |
| Zabbix OpenShift Docs (5.0) | https://www.zabbix.com/documentation/5.0/en/manual/installation/containers/openshift |
| Zabbix Kubernetes Integration | https://www.zabbix.com/integrations/kubernetes |
| Zabbix OpenShift Integration | https://www.zabbix.com/integrations/openshift |
| Red Hat Blog: Monitoring with Zabbix | https://www.redhat.com/en/blog/monitoring-infrastructure-openshift-4.x-using-zabbix-operator |
| Red Hat Catalog: Operator Container | https://catalog.redhat.com/software/containers/zabbix/zabbix-operator-certified-44/5e986d552937381642a07a51 |
| Red Hat Catalog: Operator Bundle | https://catalog.redhat.com/software/containers/zabbix/zabbixoperator-certified-bundle-marketplace/63d421c63cd852e4d554cf9c |
| Zabbix Docker (official GitHub) | https://github.com/zabbix/zabbix-docker |
| Zabbix Docker Kubernetes YAML | https://github.com/zabbix/zabbix-docker/blob/7.4/kubernetes.yaml |
| Zabbix DB Encryption Docs | https://www.zabbix.com/documentation/7.0/en/manual/appendix/install/db_encrypt/postgres |
| Zabbix Summit 2020 Presentation | https://assets.zabbix.com/files/events/zabbix_summit_2020/Tyler_French_Zabbix_meets_Kubernetes.pdf |
