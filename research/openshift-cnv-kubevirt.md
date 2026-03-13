# OpenShift Virtualization (CNV) & KubeVirt — Official Reference

> **Scope**: OpenShift Virtualization (Container-Native Virtualization / CNV), KubeVirt upstream, VirtualMachine CRD, networking modes (masquerade, bridge, SR-IOV), DataVolumes, cloud-init provisioning, and running RHEL 9 VMs on OpenShift 4.x bare-metal with ODF Ceph RBD storage.

---

## 1. OpenShift Virtualization / CNV Overview

### What It Is

**OpenShift Virtualization** (formerly Container-Native Virtualization / CNV) is a Red Hat-supported feature of OpenShift that enables running and managing **virtual machines alongside containers** on the same Kubernetes cluster. It is built on the **KubeVirt** open-source project and uses the **KVM** (Kernel-based Virtual Machine) type-1 hypervisor from RHEL.

### Relationship to KubeVirt

| Aspect | KubeVirt (Upstream) | OpenShift Virtualization (Downstream) |
|--------|--------------------|-----------------------------------------|
| **Project** | CNCF open-source project | Red Hat productized distribution |
| **API** | `kubevirt.io/v1` | Same — `kubevirt.io/v1` |
| **Operator** | Community KubeVirt operator | `kubevirt-hyperconverged` (HCO) operator via OperatorHub |
| **Namespace** | `kubevirt` | `openshift-cnv` |
| **Support** | Community | Red Hat enterprise support |
| **Extras** | Base KubeVirt + CDI | Includes CDI, Hostpath Provisioner, VM console proxy, SSP operator, common templates, Tekton tasks |

### Why It Matters for This Demo

- **Unified platform**: Run the Zabbix Agent as a RHEL 9 VM on the same OpenShift cluster that hosts the Zabbix Server containers — no separate external infrastructure needed
- **Pod networking**: The VM gets a pod IP and can communicate with Kubernetes Services (like the Zabbix Server ClusterIP) natively via cluster DNS
- **Shared storage**: The VM uses the same ODF Ceph RBD storage already available on the cluster
- **Single pane of glass**: Manage VMs with `oc`/`kubectl` and the OpenShift web console alongside container workloads

### Architecture Components

| Component | Description |
|-----------|-------------|
| **virt-operator** | Deploys and manages KubeVirt components |
| **virt-controller** | Manages VM lifecycle (create, start, stop, migrate) |
| **virt-handler** | DaemonSet on each node — manages VM processes via libvirt/QEMU |
| **virt-launcher** | Per-VM pod that wraps the QEMU/KVM process |
| **virt-api** | API server for KubeVirt custom resources |
| **CDI (Containerized Data Importer)** | Imports VM disk images into PVCs from HTTP, S3, registry, or upload |
| **HyperConverged Operator (HCO)** | Meta-operator that deploys all of the above |

---

## 2. VirtualMachine CRD — Full Spec Reference

### API Basics

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: <vm-name>
  namespace: <namespace>
  labels:
    app: <vm-name>
spec:
  running: true                    # or use runStrategy (see below)
  runStrategy: Always              # Always | Halted | Manual | RerunOnFailure
  template:
    metadata:
      labels:
        kubevirt.io/vm: <vm-name>
    spec:
      domain:
        cpu:
          cores: 2
          sockets: 1
          threads: 1
        resources:
          requests:
            memory: 4Gi
        devices:
          disks: [...]
          interfaces: [...]
          rng: {}                   # Virtio RNG device (recommended)
      networks: [...]
      volumes: [...]
      terminationGracePeriodSeconds: 180
  dataVolumeTemplates: [...]       # Inline DataVolume definitions
```

### Key Spec Fields

#### `spec.running` vs `spec.runStrategy`

| Field | Description |
|-------|-------------|
| `running: true/false` | Simple boolean to start/stop the VM |
| `runStrategy: Always` | VM is always running; restarts on failure or node reboot |
| `runStrategy: Halted` | VM is stopped; must be started manually |
| `runStrategy: Manual` | User controls start/stop via `virtctl` |
| `runStrategy: RerunOnFailure` | Auto-restart only on failure (not on success) |

> `running` and `runStrategy` are **mutually exclusive**. Use `runStrategy` for production.

#### `spec.template.spec.domain`

```yaml
domain:
  cpu:
    cores: 2               # vCPU cores per socket
    sockets: 1             # Number of CPU sockets
    threads: 1             # Threads per core
  resources:
    requests:
      memory: 4Gi          # Guest memory allocation
  devices:
    disks:
    - name: rootdisk
      disk:
        bus: virtio         # virtio | sata | scsi
    - name: cloudinitdisk
      disk:
        bus: virtio
    interfaces:
    - name: default
      masquerade: {}        # Networking mode
    rng: {}                 # Hardware RNG (recommended for entropy)
    networkInterfaceMultiqueue: true  # Multi-queue for multi-vCPU VMs
```

#### `spec.template.spec.volumes`

```yaml
volumes:
- name: rootdisk
  dataVolume:
    name: rhel9-dv          # References a DataVolume (inline or pre-existing)
- name: cloudinitdisk
  cloudInitNoCloud:
    userData: |-
      #cloud-config
      ...
```

**Volume types:**

| Volume Type | Description |
|-------------|-------------|
| `dataVolume` | PVC created/managed by CDI (DataVolume or dataVolumeTemplate) |
| `persistentVolumeClaim` | Directly reference an existing PVC |
| `containerDisk` | Ephemeral disk from a container image (no persistence) |
| `cloudInitNoCloud` | cloud-init user-data / network-data for VM provisioning |
| `emptyDisk` | Ephemeral disk (lost on VM restart) |
| `configMap` / `secret` | Mount a ConfigMap or Secret as a disk |

#### `spec.template.spec.networks`

```yaml
networks:
- name: default
  pod: {}                   # Use the pod network (cluster SDN)
```

#### `spec.dataVolumeTemplates`

DataVolumeTemplates create PVCs inline with the VM. CDI imports the disk image before the VM starts. The PVC lifecycle is tied to the VM — deleting the VM deletes the PVC.

```yaml
dataVolumeTemplates:
- metadata:
    name: rhel9-dv
  spec:
    storage:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 30Gi
      storageClassName: ocs-external-storagecluster-ceph-rbd
    source:
      http:
        url: https://download.example.com/rhel-9-cloud.qcow2
      # OR registry:
      #   url: docker://registry.example.com/rhel9-cloud:latest
      # OR pvc:
      #   name: source-pvc
      #   namespace: source-ns
```

**DataVolume source types:**

| Source | Description |
|--------|-------------|
| `http` | Download from HTTP/HTTPS URL (qcow2, raw, iso) |
| `registry` | Pull from container image registry (containerdisk format) |
| `pvc` | Clone from an existing PVC |
| `s3` | Download from S3-compatible object storage |
| `upload` | Upload via `virtctl image-upload` |
| `blank` | Empty disk |

---

## 3. Networking Modes for VMs

KubeVirt VMs connect to networks via two coordinated specs:
- `spec.template.spec.networks` — declares the **network backend**
- `spec.template.spec.domain.devices.interfaces` — declares the **virtual NIC** connected to each network

Each interface must have a corresponding network with the same `name`.

### 3.1 Pod Network with Masquerade (Default & Recommended)

**Best for our use case.** The VM gets a pod IP on the cluster SDN. It can communicate with pods, Services, and DNS natively.

- KubeVirt allocates an internal IP to the VM and uses `nftables` NAT rules to translate traffic
- Outbound traffic uses the pod's IP address (source NAT)
- Inbound traffic to the pod IP is forwarded to the VM
- The guest OS acquires its IP via DHCP (provided by KubeVirt's internal DHCP server)
- **Supports live migration** (VM moves to a new pod with a new IP; use a Kubernetes Service for stable addressing)
- Masquerade is **only allowed** for the pod network (not secondary networks)

```yaml
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
          - name: default
            masquerade: {}
            ports:
            - port: 10050       # Zabbix Agent passive check port
              protocol: TCP
      networks:
      - name: default
        pod: {}
```

**Port forwarding**: If the `ports` section is omitted, all ports are forwarded to the VM. Specifying ports restricts forwarding to listed ports only.

### 3.2 Bridge

The VM connects to a network via a Linux bridge. Used with **secondary networks** via Multus `NetworkAttachmentDefinition`.

- The pod interface MAC is delegated to the VM via DHCP
- The pod loses its IP address (may affect service mesh, monitoring)
- **Does NOT support live migration** on the pod network
- Useful for L2 external network access or when the VM needs a direct connection to a physical network

```yaml
domain:
  devices:
    interfaces:
    - name: bridge-net
      bridge: {}
networks:
- name: bridge-net
  multus:
    networkName: my-bridge-nad
```

### 3.3 SR-IOV

Hardware passthrough of SR-IOV Virtual Functions (VFs) for high-performance networking.

- Requires SR-IOV device plugin and CNI on the cluster
- VFs are passed via `vfio-pci` directly to the guest OS
- Near-native network performance
- Requires dedicated hardware and node configuration

```yaml
domain:
  devices:
    interfaces:
    - name: sriov-net
      sriov: {}
networks:
- name: sriov-net
  multus:
    networkName: default/sriov-net
```

### 3.4 Passt (Beta, v1.8+)

User-space network binding using PASST (Plug A Simple Socket Transport). Alternative to masquerade.

- Does not require CAP_NET_RAW or CAP_NET_ADMIN
- Supports service meshes and IPv6 out of the box
- Supports live migration
- Adds 250Mi memory overhead

### Networking Mode Comparison

| Feature | Masquerade | Bridge | SR-IOV | Passt |
|---------|------------|--------|--------|-------|
| **Pod network** | Yes | Yes (limited) | No | Yes |
| **Secondary network** | No | Yes | Yes | No |
| **Live migration** | Yes | No (pod net) | Limited | Yes |
| **Service mesh** | No | No | No | Yes |
| **Performance** | Good | Good | Best | Good |
| **Requires extra HW** | No | No | Yes | No |
| **Pod gets IP** | Yes | No | Yes | Yes |
| **Our use case** | **Best fit** | Not needed | Overkill | Alternative |

### How a VM on Pod Network Reaches Kubernetes Services

When using masquerade mode with the pod network:

1. The VM gets an internal IP via DHCP (typically `10.0.2.2/24` with gateway `10.0.2.1`)
2. KubeVirt's NAT rules translate this to the pod's actual cluster IP
3. The pod (and thus the VM) can resolve Kubernetes Service DNS names:
   - `<service-name>.<namespace>.svc.cluster.local`
   - Short form: `<service-name>.<namespace>` or just `<service-name>` (within same namespace)
4. ClusterIP Services are directly reachable — no NodePort or LoadBalancer needed
5. **For Zabbix**: The VM can reach the Zabbix Server at `zabbix-server.zabbix-system.svc.cluster.local:10051`

---

## 4. RHEL 9 VM Example — Complete VirtualMachine YAML

This is a complete VirtualMachine manifest for a RHEL 9 VM running the Zabbix Agent, using:
- DataVolume to import the RHEL 9 cloud image
- 2 CPU cores, 4Gi memory
- Masquerade networking (pod network)
- cloud-init for provisioning (hostname, zabbix-agent install, configuration)
- ODF Ceph RBD storage

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: zabbix-agent-rhel9
  namespace: zabbix-system
  labels:
    app: zabbix-agent-rhel9
    app.kubernetes.io/part-of: zabbix
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        app: zabbix-agent-rhel9
        kubevirt.io/vm: zabbix-agent-rhel9
    spec:
      domain:
        cpu:
          cores: 2
          sockets: 1
          threads: 1
        resources:
          requests:
            memory: 4Gi
        devices:
          disks:
          - name: rootdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
            ports:
            - name: zabbix-agent
              port: 10050
              protocol: TCP
          rng: {}
      networks:
      - name: default
        pod: {}
      terminationGracePeriodSeconds: 180
      volumes:
      - name: rootdisk
        dataVolume:
          name: zabbix-agent-rhel9-rootdisk
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |-
            #cloud-config
            hostname: zabbix-agent-rhel9
            fqdn: zabbix-agent-rhel9.zabbix-system.svc.cluster.local

            user: cloud-user
            password: changeme
            chpasswd:
              expire: false
            ssh_authorized_keys: []

            packages:
              - qemu-guest-agent

            runcmd:
              - ["systemctl", "enable", "--now", "qemu-guest-agent"]
              # Install Zabbix repository
              - ["rpm", "-Uvh", "https://repo.zabbix.com/zabbix/7.2/release/el/9/noarch/zabbix-release-latest-7.2.el9.noarch.rpm"]
              - ["dnf", "clean", "all"]
              # Install Zabbix Agent
              - ["dnf", "install", "-y", "zabbix-agent"]
              # Configure Zabbix Agent
              - >-
                sed -i
                's/^Server=127.0.0.1$/Server=zabbix-server.zabbix-system.svc.cluster.local/'
                /etc/zabbix/zabbix_agentd.conf
              - >-
                sed -i
                's/^ServerActive=127.0.0.1$/ServerActive=zabbix-server.zabbix-system.svc.cluster.local/'
                /etc/zabbix/zabbix_agentd.conf
              - >-
                sed -i
                's/^Hostname=Zabbix server$/Hostname=zabbix-agent-rhel9/'
                /etc/zabbix/zabbix_agentd.conf
              # Enable and start Zabbix Agent
              - ["systemctl", "enable", "--now", "zabbix-agent"]

  dataVolumeTemplates:
  - metadata:
      name: zabbix-agent-rhel9-rootdisk
    spec:
      storage:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 30Gi
        storageClassName: ocs-external-storagecluster-ceph-rbd
      source:
        http:
          url: https://download.example.com/rhel-9.6-x86_64-kvm.qcow2
```

> **Note**: Replace the `source.http.url` with the actual RHEL 9 Generic Cloud (KVM) image URL from the Red Hat Customer Portal or an internal mirror. The official image name is typically `rhel-9.x-x86_64-kvm.qcow2`. Alternatively, use `source.registry` to pull from a container registry that hosts the RHEL 9 cloud image as a containerdisk.

---

## 5. VM-to-Pod Communication

### Same-Cluster Service Discovery

When the VM uses **masquerade** mode on the **pod network**, it participates in the Kubernetes network like any other pod:

| Capability | Details |
|------------|---------|
| **DNS resolution** | The VM can resolve `<service>.<namespace>.svc.cluster.local` via CoreDNS |
| **ClusterIP access** | The VM can directly reach any ClusterIP Service in any namespace |
| **Same-namespace shortname** | Within `zabbix-system`, the VM can use just `zabbix-server` as the hostname |
| **Pod-to-pod** | The VM's pod IP is reachable from other pods, and the VM can reach other pod IPs |
| **No NodePort needed** | All communication happens over the cluster internal network |
| **No LoadBalancer needed** | No external ingress required for intra-cluster traffic |

### Zabbix-Specific Communication Flow

```
┌─────────────────────────────────┐     ┌──────────────────────────────────┐
│  Zabbix Agent VM (RHEL 9)       │     │  Zabbix Server (Pod)             │
│  KubeVirt VirtualMachine        │     │  Deployment in zabbix-system     │
│                                 │     │                                  │
│  Pod IP: 10.128.x.x            │────>│  ClusterIP: zabbix-server        │
│  Port: 10050 (agent)           │     │  Port: 10051 (trapper)           │
│                                 │<────│                                  │
│  Connects to:                   │     │  Polls agent at pod IP:10050     │
│  zabbix-server.zabbix-system    │     │                                  │
│  .svc.cluster.local:10051       │     │                                  │
└─────────────────────────────────┘     └──────────────────────────────────┘
         Both on pod network — direct ClusterIP communication
```

**Active checks** (agent → server): The agent connects to `zabbix-server.zabbix-system.svc.cluster.local:10051` — this resolves to the Zabbix Server's ClusterIP, directly reachable from the VM.

**Passive checks** (server → agent): The Zabbix Server connects to the VM's pod IP on port 10050. Create a Service for the VM to provide a stable DNS name:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zabbix-agent-rhel9
  namespace: zabbix-system
spec:
  selector:
    app: zabbix-agent-rhel9
  ports:
  - name: zabbix-agent
    port: 10050
    targetPort: 10050
    protocol: TCP
  type: ClusterIP
```

This Service allows the Zabbix Server to reach the agent at `zabbix-agent-rhel9.zabbix-system.svc.cluster.local:10050` — stable even if the VM migrates to a different pod/IP.

---

## 6. cloud-init for VM Provisioning

### How `cloudInitNoCloud` Works in KubeVirt

KubeVirt supports cloud-init via the `cloudInitNoCloud` volume type. It creates a virtual CD-ROM (NoCloud datasource) that the guest OS reads on first boot.

```yaml
volumes:
- name: cloudinitdisk
  cloudInitNoCloud:
    userData: |-
      #cloud-config
      ...
    networkData: |-
      version: 2
      ethernets:
        eth0:
          dhcp4: true
```

### cloud-init Fields

| Field | Description |
|-------|-------------|
| `userData` | Cloud-config YAML or shell script for instance initialization |
| `userDataBase64` | Base64-encoded userData (alternative to plaintext) |
| `networkData` | Network configuration (Netplan v2 format) |
| `networkDataBase64` | Base64-encoded networkData |
| `secretRef` | Reference to a Kubernetes Secret containing userData |
| `networkDataSecretRef` | Reference to a Secret containing networkData |

### userData Modules for Zabbix Agent Provisioning

```yaml
cloudInitNoCloud:
  userData: |-
    #cloud-config
    # Set hostname
    hostname: zabbix-agent-rhel9
    fqdn: zabbix-agent-rhel9.zabbix-system.svc.cluster.local

    # Create user with password
    user: cloud-user
    password: changeme
    chpasswd:
      expire: false

    # SSH keys (optional)
    ssh_authorized_keys:
      - ssh-rsa AAAA...

    # Install packages (requires network access to repos)
    packages:
      - qemu-guest-agent

    # Run commands on first boot
    runcmd:
      # Enable QEMU guest agent (for VM management)
      - ["systemctl", "enable", "--now", "qemu-guest-agent"]

      # Add Zabbix repository
      - ["rpm", "-Uvh", "https://repo.zabbix.com/zabbix/7.2/release/el/9/noarch/zabbix-release-latest-7.2.el9.noarch.rpm"]
      - ["dnf", "clean", "all"]

      # Install Zabbix Agent
      - ["dnf", "install", "-y", "zabbix-agent"]

      # Configure Zabbix Agent
      - >-
        sed -i
        's/^Server=127.0.0.1$/Server=zabbix-server.zabbix-system.svc.cluster.local/'
        /etc/zabbix/zabbix_agentd.conf
      - >-
        sed -i
        's/^ServerActive=127.0.0.1$/ServerActive=zabbix-server.zabbix-system.svc.cluster.local/'
        /etc/zabbix/zabbix_agentd.conf
      - >-
        sed -i
        's/^Hostname=Zabbix server$/Hostname=zabbix-agent-rhel9/'
        /etc/zabbix/zabbix_agentd.conf

      # Start Zabbix Agent
      - ["systemctl", "enable", "--now", "zabbix-agent"]

    # Write arbitrary files (alternative to sed)
    write_files:
      - path: /etc/zabbix/zabbix_agentd.d/custom.conf
        content: |
          # Custom Zabbix Agent configuration
          Timeout=30
          DebugLevel=3
```

### Important Notes on cloud-init

- The `#cloud-config` header is **mandatory** — without it, cloud-init treats the data as a shell script
- The `packages` module requires the guest to have network access to package repositories (RHEL subscription or local mirror)
- `runcmd` commands execute **after** the `packages` module
- For complex configurations, prefer `write_files` + `runcmd` over multiple `sed` commands
- The `qemu-guest-agent` package enables VM management features (IP reporting, graceful shutdown, exec)
- On SELinux-enabled RHEL, run `setsebool -P virt_qemu_ga_manage_ssh on` if using guest-agent SSH key injection

---

## 7. Installing the CNV Operator

### Prerequisites

- OpenShift 4.x cluster (bare-metal, vSphere, or other platform supporting virtualization)
- Cluster nodes must have **hardware virtualization** support (Intel VT-x / AMD-V)
- Bare-metal or nested virtualization enabled
- Sufficient resources on worker/master nodes for VM workloads

### Installation via OperatorHub (Web Console)

1. Navigate to **Operators → OperatorHub**
2. Search for **"OpenShift Virtualization"**
3. Click the **OpenShift Virtualization** tile
4. Click **Install**
5. Settings:
   - **Update channel**: `stable` (recommended)
   - **Installation mode**: All namespaces (cluster-wide)
   - **Installed Namespace**: `openshift-cnv` (auto-created)
   - **Approval strategy**: `Automatic` or `Manual`
6. Click **Install**
7. After the operator is installed, go to **Operators → Installed Operators → OpenShift Virtualization**
8. Click **Create HyperConverged** to deploy the virtualization infrastructure
9. Accept defaults and click **Create**

### Installation via CLI

```yaml
# 1. Create the namespace
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cnv
---
# 2. Create the OperatorGroup
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
  - openshift-cnv
---
# 3. Create the Subscription
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: kubevirt-hyperconverged
  channel: stable
  installPlanApproval: Automatic
```

```bash
# Apply the manifests
oc apply -f cnv-operator-install.yaml

# Wait for the operator to install
oc wait csv -n openshift-cnv -l operators.coreos.com/kubevirt-hyperconverged.openshift-cnv --for=jsonpath='{.status.phase}'=Succeeded --timeout=600s
```

```yaml
# 4. Create the HyperConverged CR to deploy virtualization components
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec: {}
```

```bash
# Apply the HyperConverged CR
oc apply -f cnv-hyperconverged.yaml

# Verify all pods are running
oc get pods -n openshift-cnv

# Verify the HyperConverged CR is ready
oc get hyperconverged kubevirt-hyperconverged -n openshift-cnv -o jsonpath='{.status.conditions[?(@.type=="Available")].status}'
# Expected: True
```

### Verification Commands

```bash
# Check operator status
oc get csv -n openshift-cnv

# Check all CNV pods
oc get pods -n openshift-cnv

# Check KubeVirt CR status
oc get kubevirt -n openshift-cnv

# Check HyperConverged CR status
oc get hyperconverged -n openshift-cnv -o yaml

# Verify virtctl is available (after installing the CLI)
virtctl version

# List available VM templates
oc get templates -n openshift

# Check node virtualization capability
oc describe node <node-name> | grep -i "kvm\|virt\|cpu-model"
```

### Namespace Structure

| Namespace | Purpose |
|-----------|---------|
| `openshift-cnv` | CNV operator, virt-controller, virt-api, CDI, HCO, SSP |
| `openshift-cnv` (pods) | virt-handler DaemonSet runs on each node |
| User namespace (e.g. `zabbix-system`) | Where VirtualMachine CRs are created |

---

## 8. Working with VMs — Common Operations

### Using `virtctl` CLI

```bash
# Start a VM
virtctl start <vm-name> -n <namespace>

# Stop a VM
virtctl stop <vm-name> -n <namespace>

# Restart a VM
virtctl restart <vm-name> -n <namespace>

# Console access (VNC)
virtctl console <vm-name> -n <namespace>

# SSH into a VM (requires guest-agent or exposed SSH)
virtctl ssh -i ~/.ssh/id_rsa cloud-user@<vm-name> -n <namespace>

# Upload a disk image to a PVC
virtctl image-upload pvc <pvc-name> \
  --size=30Gi \
  --image-path=/path/to/rhel-9-kvm.qcow2 \
  --storage-class=ocs-external-storagecluster-ceph-rbd \
  -n <namespace>

# Expose a VM as a Service
virtctl expose vm <vm-name> --port=10050 --name=<service-name> -n <namespace>
```

### Using `oc` / `kubectl`

```bash
# List VMs
oc get vm -n zabbix-system

# List running VM instances
oc get vmi -n zabbix-system

# Describe a VM
oc describe vm zabbix-agent-rhel9 -n zabbix-system

# Get VM IP address
oc get vmi zabbix-agent-rhel9 -n zabbix-system -o jsonpath='{.status.interfaces[0].ipAddress}'

# Check DataVolume import progress
oc get dv -n zabbix-system

# Watch DataVolume import
oc get dv zabbix-agent-rhel9-rootdisk -n zabbix-system -w

# Check CDI importer pod logs
oc logs -n zabbix-system -l cdi.kubevirt.io=importer
```

---

## 9. Prerequisites and Gotchas

1. **Hardware virtualization required**: Nodes must have Intel VT-x or AMD-V enabled in BIOS. Check with `oc debug node/<node> -- chroot /host grep -c vmx /proc/cpuinfo` (Intel) or `svm` (AMD).

2. **SingleReplica topology**: This demo cluster has a single node — the VM and all containers run on the same node. No HA or live migration (migration requires multiple nodes). This is acceptable for a demo.

3. **RHEL cloud image source**: The RHEL 9 KVM guest image (`rhel-9.x-x86_64-kvm.qcow2`) must be downloaded from the Red Hat Customer Portal or an internal mirror. CDI will import it via the DataVolume `source.http.url`. Ensure the URL is accessible from the cluster.

4. **Storage for VM disks**: The `ocs-external-storagecluster-ceph-rbd` StorageClass uses `WaitForFirstConsumer` binding mode. DataVolume PVCs stay `Pending` until the virt-launcher pod is scheduled. This is expected behavior.

5. **VM SCC requirements**: KubeVirt's `virt-launcher` pods require the `kubevirt-controller` SCC (installed automatically by the operator). No manual SCC configuration is needed for the VM itself.

6. **Masquerade port forwarding**: If you specify `ports` on the masquerade interface, only those ports are forwarded. Omit the `ports` section to forward all traffic. For Zabbix Agent, expose port 10050.

7. **DNS resolution in the VM**: The guest OS resolves Kubernetes Service names through CoreDNS (the pod's `/etc/resolv.conf` is propagated). If DNS fails inside the VM, check that the guest has DHCP-acquired DNS settings (nameserver should be the cluster DNS IP, typically `172.30.0.10`).

8. **cloud-init runs only on first boot**: The `runcmd` and `packages` modules execute only during the first boot. If you need to reconfigure, either rebuild the VM or SSH in manually.

9. **RHEL subscription for packages**: The `packages` cloud-init module needs repo access. For the Zabbix Agent install, the VM needs internet access to reach `repo.zabbix.com`. Alternatively, pre-bake the agent into a custom RHEL 9 image.

10. **VM Service for stable addressing**: Create a ClusterIP Service for the VM (selector matches VM pod labels). This provides a stable DNS name for the Zabbix Server to reach the agent, even if the VM's pod IP changes.

11. **qemu-guest-agent**: Install and enable `qemu-guest-agent` in the VM for proper lifecycle management (graceful shutdown, IP reporting in `oc get vmi`, exec commands).

12. **DataVolume import time**: Importing a RHEL 9 cloud image (~700MB-1.5GB) takes several minutes depending on network speed and storage performance. Monitor with `oc get dv -w`.

13. **Ceph RBD + KubeVirt**: Ceph RBD block storage works well with KubeVirt VMs. The VM disk is stored as an RBD image in the Ceph pool. Volume expansion is supported for growing the VM disk.

---

## 10. Official Documentation URLs

### Red Hat OpenShift Virtualization

| Resource | URL |
|----------|-----|
| OCP 4.18 Virtualization (full docs) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/virtualization/index |
| OCP 4.15 Installing Virtualization | https://docs.redhat.com/en/documentation/openshift_container_platform/4.15/html/virtualization/installing#installing-virt |
| OCP 4.12 Virtualization (single page) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.12/html-single/virtualization/index |
| OCP 4.16 Getting Started with Virt | https://docs.openshift.com/container-platform/4.16/virt/getting_started/virt-getting-started.html |
| OpenShift Examples — CNV/KubeVirt | https://examples.openshift.pub/kubevirt/ |
| Red Hat Developer — OCP Virt Setup Guide | https://developers.redhat.com/blog/2024/10/04/step-step-guide-setting-ocp-virtualization-hyperconverged-odf-and-deploy-10k-vms |

### KubeVirt Upstream

| Resource | URL |
|----------|-----|
| KubeVirt User Guide — Home | https://kubevirt.io/user-guide/ |
| Creating VMs (virtctl) | https://kubevirt.io/user-guide/virtual_machines/creating_vms/ |
| Interfaces and Networks | https://kubevirt.io/user-guide/network/interfaces_and_networks/ |
| Disks and Volumes | https://kubevirt.io/user-guide/storage/disks_and_volumes/ |
| Startup Scripts (cloud-init) | https://kubevirt.io/user-guide/virtual_machines/startup_scripts/ |
| VM Instances | https://kubevirt.io/user-guide/user_workloads/virtual_machine_instances/ |
| Instance Types and Preferences | https://kubevirt.io/user-guide/virtual_machines/instancetypes/ |
| API Reference | https://kubevirt.io/api-reference/master/definitions.html |
| KubeVirt GitHub | https://github.com/kubevirt/kubevirt |
| CDI (Containerized Data Importer) | https://github.com/kubevirt/containerized-data-importer |

### Community & Learning

| Resource | URL |
|----------|-----|
| Red Hat Quick Courses — OCP Virt Cookbook | https://redhatquickcourses.github.io/ocp-virt-cookbook/ |
| RHEL 9 VM + Network Policies on OCP Virt | https://stephennimmo.com/2025/02/26/setting-up-network-policies-on-a-rhel-9-vm-running-in-openshift-virtualization/ |
| KubeVirt Implementation Guide | https://marcinkujawski.pl/running-virtual-machines-on-kubernetes-with-kubevirt-complete-implementation-guide/ |
| OpenShift Virtualization Key Concepts | https://ivolve.io/blog/openshift-virtualization-guide/ |
