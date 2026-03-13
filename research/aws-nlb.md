# Networking for Zabbix on CNV/KubeVirt OpenShift

> **Scope**: Networking topology for Zabbix Server (pods) and Zabbix Agent (KubeVirt VM) on OpenShift 4.20 with CNV. Both components run on the same cluster SDN — no external TCP exposure needed.

---

## 1. Architecture Overview

| Component | Location | Network |
| --------- | -------- | ------- |
| Zabbix Server | Pods (Zabbix Operator) | Cluster SDN (OVN-Kubernetes) |
| Zabbix Agent | KubeVirt VirtualMachine | Pod network (masquerade) — same SDN |
| Connectivity | Direct pod-to-pod | ClusterIP Service |

**Topology:**

- **Zabbix Server** runs as pods deployed by the Zabbix Operator, behind a ClusterIP Service.
- **Zabbix Agent** runs inside a KubeVirt VirtualMachine using the pod network (masquerade mode).
- Both are on the OpenShift SDN (OVN-Kubernetes) — direct pod-to-pod communication over the cluster overlay.
- **No NLB, NodePort, or external exposure needed** — the VM and server pods share the same network fabric.

The VM receives a pod IP from the cluster CIDR and can reach any ClusterIP Service via `<svc>.<ns>.svc.cluster.local`.

---

## 2. ClusterIP Service for Zabbix Server

The only Service required is a ClusterIP Service exposing the Zabbix Server ports:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zabbix-demo-server
  namespace: zabbix-system
spec:
  type: ClusterIP
  ports:
    - name: zabbix-trapper
      port: 10051
      targetPort: 10051
      protocol: TCP
    - name: zabbix-server
      port: 10050
      targetPort: 10050
      protocol: TCP
  selector:
    app.kubernetes.io/name: zabbix-server
    zabbix-cluster: zabbix-demo
```

- **10051** — Zabbix trapper (active agent checks, sender)
- **10050** — Zabbix server (passive agent checks)

---

## 3. VM-to-Service Connectivity

**How it works:**

1. The KubeVirt VM on pod network (masquerade) gets a pod IP from the cluster CIDR.
2. The VM uses the cluster's CoreDNS for name resolution.
3. The VM resolves `zabbix-demo-server.zabbix-system.svc.cluster.local` to the ClusterIP.
4. TCP connections to port 10051 (trapper) and 10050 (agent checks) work directly.
5. No firewall rules or load balancers are required.

**Zabbix Agent configuration (inside the VM):**

```ini
# /etc/zabbix/zabbix_agent2.conf
Server=zabbix-demo-server.zabbix-system.svc.cluster.local
ServerActive=zabbix-demo-server.zabbix-system.svc.cluster.local:10051
```

---

## 4. KubeVirt Networking Modes

| Mode | Use Case |
| ---- | -------- |
| **Masquerade** (pod network) | VM on cluster SDN — best for intra-cluster communication |

For this demo, masquerade (pod network) is the only mode needed.

---

## 5. Verifying Connectivity

From inside the VM (via `virtctl console` or SSH):

```bash
# Resolve the Service DNS name
nslookup zabbix-demo-server.zabbix-system.svc.cluster.local

# Test TCP connectivity to trapper port
curl -v telnet://zabbix-demo-server.zabbix-system.svc.cluster.local:10051
```

From the cluster:

```bash
# Confirm Service and endpoints
oc get svc zabbix-demo-server -n zabbix-system
oc get endpoints zabbix-demo-server -n zabbix-system
```

---

## 6. Official Documentation

| Resource | URL |
| -------- | --- |
| KubeVirt networking | <https://kubevirt.io/user-guide/network/network_binding_plugins/> |
| OpenShift Virtualization networking | <https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/networking> |
| Kubernetes Services (ClusterIP) | <https://kubernetes.io/docs/concepts/services-networking/service/#type-clusterip> |
| OVN-Kubernetes | <https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ovn-kubernetes_network_plugin/about-ovn-kubernetes> |
