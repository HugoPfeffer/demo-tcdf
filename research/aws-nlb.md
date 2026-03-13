# AWS Network Load Balancer on Kubernetes/OpenShift — Official Reference

> **Scope**: Provisioning an internal AWS NLB via Kubernetes Service annotations, in-tree vs AWS Load Balancer Controller, TCP traffic for Zabbix trapper (port 10051), health checks, security groups, and DNS resolution.

---

## 1. Two Provisioning Paths: In-Tree vs AWS Load Balancer Controller

### In-Tree AWS Cloud Provider (Legacy)

- Built into the Kubernetes `cloud-controller-manager`
- Default when **no** `aws-load-balancer-type` annotation: provisions a **Classic ELB**
- When `aws-load-balancer-type: "nlb"`: provisions an **NLB** via the in-tree provider
- Recognizes `service.beta.kubernetes.io/aws-load-balancer-internal: "true"`

### AWS Load Balancer Controller (External / kubernetes-sigs)

- Standalone controller deployed as a Deployment in the cluster
- Activated when `aws-load-balancer-type` is `"nlb-ip"` or `"external"`
- On v2.5.0+, provides a mutating webhook that sets `spec.loadBalancerClass` automatically
- Uses the newer `service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"` annotation
- Supports a much richer set of annotations

> **Critical**: The `aws-load-balancer-type` annotation must be set at Service **creation time** and **never modified** on an existing Service. Doing so can result in leaked AWS resources.

---

## 2. `aws-load-balancer-type` Values

| Value | Controller | Behavior |
|-------|-----------|----------|
| *(not set)* | In-tree | Classic ELB provisioned |
| `"nlb"` | In-tree | NLB provisioned, instance-mode targets |
| `"nlb-ip"` | AWS LBC | NLB provisioned, IP-mode targets. **Deprecated** in favor of `"external"` |
| `"external"` | AWS LBC | NLB provisioned. Target type set by `aws-load-balancer-nlb-target-type` |

---

## 3. Internal vs External NLB Configuration

### Legacy Annotation (In-Tree)

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-internal: "true"
```

- Used by the in-tree cloud controller manager
- **Deprecated** starting AWS LBC v2.2.0

### Modern Annotation (AWS Load Balancer Controller)

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
```

- Valid values: `"internal"` or `"internet-facing"`
- Default: `"internal"` (for the AWS LBC)
- Takes **precedence** over the legacy annotation

---

## 4. Complete AWS Service Annotations Reference

### Traffic Routing

| Annotation | Default | Purpose |
|-----------|---------|---------|
| `aws-load-balancer-type` | — | Controller selection: `"external"`, `"nlb-ip"`, or `"nlb"` |
| `aws-load-balancer-nlb-target-type` | `instance` | Target type: `"instance"` or `"ip"` |
| `aws-load-balancer-name` | — | Custom NLB name (max 32 chars, immutable) |
| `aws-load-balancer-subnets` | auto-discovered | Subnet IDs or names |
| `aws-load-balancer-eip-allocations` | — | Elastic IP allocations (internet-facing only) |
| `aws-load-balancer-private-ipv4-addresses` | — | Static private IPs (internal only) |
| `aws-load-balancer-alpn-policy` | None | ALPN policy |
| `aws-load-balancer-target-node-labels` | — | Node labels for target registration |

*All annotations prefixed with `service.beta.kubernetes.io/`*

### Access Control

| Annotation | Default | Purpose |
|-----------|---------|---------|
| `aws-load-balancer-scheme` | `internal` | `"internal"` or `"internet-facing"` |
| `aws-load-balancer-internal` | `false` | **Deprecated** — use `scheme` instead |
| `load-balancer-source-ranges` | `0.0.0.0/0` | Allowed CIDRs |
| `aws-load-balancer-security-group-prefix-lists` | — | AWS managed prefix lists |
| `aws-load-balancer-ip-address-type` | `ipv4` | `"ipv4"` or `"dualstack"` |

### Security Groups

| Annotation | Default | Purpose |
|-----------|---------|---------|
| `aws-load-balancer-security-groups` | auto-created | Frontend SG IDs/names |
| `aws-load-balancer-manage-backend-security-group-rules` | auto | Auto-manage ingress rules on instance/ENI SGs |
| `aws-load-balancer-disable-nlb-sg` | `false` | Disable SG management |

### Health Checks

| Annotation | Default | Purpose |
|-----------|---------|---------|
| `aws-load-balancer-healthcheck-protocol` | `TCP` | Protocol: `tcp`, `http`, `https` |
| `aws-load-balancer-healthcheck-port` | `traffic-port` | Port for health check |
| `aws-load-balancer-healthcheck-path` | `"/"` (HTTP/S) | HTTP path for health check |
| `aws-load-balancer-healthcheck-healthy-threshold` | `3` | Consecutive successes for healthy |
| `aws-load-balancer-healthcheck-unhealthy-threshold` | `3` | Consecutive failures for unhealthy |
| `aws-load-balancer-healthcheck-interval` | `10` | Seconds between checks |
| `aws-load-balancer-healthcheck-timeout` | `10` | Seconds to wait for response |
| `aws-load-balancer-healthcheck-success-codes` | `200-399` | HTTP success codes |

### TLS / SSL

| Annotation | Default | Purpose |
|-----------|---------|---------|
| `aws-load-balancer-ssl-cert` | — | ACM certificate ARN(s) |
| `aws-load-balancer-ssl-ports` | all ports | Frontend ports with TLS listeners |
| `aws-load-balancer-ssl-negotiation-policy` | `ELBSecurityPolicy-2016-08` | TLS security policy |
| `aws-load-balancer-backend-protocol` | `tcp` | Backend protocol: `"tcp"` or `"ssl"` |

### Resource Attributes

| Annotation | Default | Purpose |
|-----------|---------|---------|
| `aws-load-balancer-attributes` | — | NLB-level attributes (access logs, cross-zone, deletion protection) |
| `aws-load-balancer-target-group-attributes` | — | Target group attributes (deregistration delay, stickiness, proxy protocol v2) |
| `aws-load-balancer-proxy-protocol` | — | `"*"` to enable proxy protocol v2 |
| `aws-load-balancer-additional-resource-tags` | — | Additional AWS resource tags |

---

## 5. TCP vs HTTP on NLB

- NLB supports **TCP**, **UDP**, **TLS**, and **TCP_UDP** listeners
- For pure TCP traffic (Zabbix trapper on port 10051): no TLS annotations needed — NLB creates a **TCP listener** by default
- For TLS termination at NLB: add `aws-load-balancer-ssl-cert` and `aws-load-balancer-ssl-ports`
- NLB does **not** support HTTP/HTTPS listeners natively — that requires an ALB

---

## 6. NLB Target Types

### Instance Mode

- Traffic routes to EC2 instances on the Service's NodePort
- kube-proxy forwards from NodePort to pods
- Service must be `NodePort` or `LoadBalancer` type

### IP Mode

- Traffic routes **directly to Pod IPs** — eliminates extra hop
- Supports pods on EC2 instances and **AWS Fargate**
- Requires VPC-native CNI (e.g., Amazon VPC CNI)
- Better for latency-sensitive workloads

---

## 7. DNS Name Resolution

- NLB provisioning populates `.status.loadBalancer.ingress` with the NLB DNS name
- Pattern: `<nlb-name>-<hash>.<region>.elb.amazonaws.com`
- **Internal NLBs**: DNS resolves to **private IPs** within the VPC
- EC2 instances in the **same VPC** can reach the internal NLB via this DNS name

```bash
# Retrieve the NLB DNS name
kubectl get svc <service-name> -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

## 8. OpenShift-Specific Considerations

- OpenShift 4.x on AWS uses the same `type: LoadBalancer` mechanism with the same annotations
- OpenShift includes the in-tree AWS cloud provider by default
- For the AWS Load Balancer Controller on OpenShift/ROSA:
  - Must be installed separately (Helm or OLM Operator)
  - On ROSA: available as a managed add-on
  - On self-managed OCP: install via `aws-load-balancer-controller` Helm chart
- The same `service.beta.kubernetes.io/aws-load-balancer-*` annotations work identically on OpenShift
- OpenShift IngressController objects (for Routes) are separate — these annotations apply to user-created Services

---

## 9. In-Tree to AWS LBC Transition

- Starting Kubernetes v1.20 (backported to v1.18.18+, v1.19.10+), the in-tree cloud provider ignores Services annotated with `"nlb-ip"` or `"external"`
- On AWS LBC v2.5.0+, a mutating webhook sets `spec.loadBalancerClass` automatically
- **Migration**: Do NOT change the annotation on existing Services. Create a new Service and update DNS/routing.
- In-tree annotations continue to work but are considered legacy

---

## 10. Analysis: Spec's Service vs Recommended Configuration

### Current Spec (In-Tree — Valid)

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  service.beta.kubernetes.io/aws-load-balancer-internal: "true"
```

### AWS LBC Equivalent (Modern)

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-type: "external"
  service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
```

### Recommended Additional Annotations for Zabbix Trapper

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "tcp"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "3"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"
  service.beta.kubernetes.io/aws-load-balancer-attributes: "load_balancing.cross_zone.enabled=true"
  service.beta.kubernetes.io/load-balancer-source-ranges: "10.0.0.0/8"
```

---

## 11. Official Documentation URLs

| Resource | URL |
|----------|-----|
| Kubernetes Service docs | https://kubernetes.io/docs/concepts/services-networking/service/ |
| Kubernetes Service (LoadBalancer) | https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer |
| Kubernetes Service (Internal LB) | https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer |
| Kubernetes annotations reference | https://kubernetes.io/docs/reference/labels-annotations-taints/ |
| AWS LBC — Home | https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/ |
| AWS LBC — Service Annotations | https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/service/annotations/ |
| AWS LBC — Subnet Discovery | https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/subnet_discovery/ |
| AWS NLB product docs | https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html |
| AWS NLB Target Groups | https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html |
| OpenShift Ingress & Load Balancing | https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/ingress_and_load_balancing/ |
| OpenShift Networking Overview | https://docs.openshift.com/container-platform/4.17/networking/ |
| AWS LBC GitHub | https://github.com/kubernetes-sigs/aws-load-balancer-controller |
