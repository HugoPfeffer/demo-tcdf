# OpenShift Routes & Cert-Manager TLS Integration — Official Reference

> **Scope**: OpenShift Route API, TLS termination types, Red Hat Cert-Manager Operator, the openshift-routes controller for automatic certificate injection, ClusterIssuer configuration, and Route annotations.

---

## 1. OpenShift Route API Specification

**API Version**: `route.openshift.io/v1`
**Kind**: `Route`

### Full Route Spec

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: <route-name>
  namespace: <namespace>
  annotations: {}
spec:
  host: <hostname>                         # FQDN for the route
  path: <path>                             # Optional, path-based routing (not for passthrough)
  to:
    kind: Service
    name: <service-name>
    weight: 100                            # Relative weight for traffic (0-256)
  port:
    targetPort: <port-name-or-number>      # Target port on the service
  wildcardPolicy: None                     # None or Subdomain
  tls:
    termination: edge                      # edge | passthrough | reencrypt
    insecureEdgeTerminationPolicy: Redirect  # Redirect | Allow | None
    certificate: |                         # PEM-encoded certificate (edge/reencrypt)
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |                                 # PEM-encoded private key (edge/reencrypt)
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----
    caCertificate: |                       # PEM-encoded CA certificate (optional)
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    destinationCACertificate: |            # CA cert for backend (reencrypt only)
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
```

---

## 2. Cluster Environment Context

| Detail | Value |
|--------|-------|
| **OCP Version** | 4.20.15 |
| **Platform** | `None` (bare-metal with Red Hat CNV, SingleReplica) |
| **Route Domain** | `*.apps.cluster-d45v2.dynamic.redhatworkshops.io` |
| **cert-manager Operator** | v1.18.1 — INSTALLED and healthy (`cert-manager-operator.v1.18.1` in `cert-manager-operator` namespace) |
| **cert-manager namespace** | `cert-manager` (Active) |
| **cert-manager-operator namespace** | `cert-manager-operator` (Active) |

**cert-manager status**: The Red Hat cert-manager Operator is installed and running. No additional installation needed for cert-manager itself.

**openshift-routes controller**: The Red Hat cert-manager Operator does **not** include the openshift-routes controller. It must be installed separately via Helm into the `cert-manager` namespace (see Section 6 for installation instructions). Verify whether it is already deployed:

```bash
oc get pods -n cert-manager -l app.kubernetes.io/name=openshift-routes
```

---

## 3. TLS Termination Types

### Edge Termination

- TLS terminates at the **OpenShift Router** (HAProxy), before traffic reaches the Pod
- Router decrypts the connection and forwards traffic **unencrypted** (HTTP) to the backend service
- Certificate configured **on the Route** (or injected by cert-manager)
- **Best for**: Most web applications where internal cluster traffic encryption is not required
- Path-based routing IS supported

### Passthrough Termination

- TLS terminates at the **Pod itself**; the router tunnels the encrypted connection through
- No certificate configuration on the Route; the Pod must have TLS configured
- Router cannot inspect traffic or do path-based routing
- **Best for**: Applications that must handle their own TLS (mTLS, specific cipher requirements)

### Reencrypt Termination

- TLS terminates at the **Router**, then a **new TLS connection** is established to the Pod
- Requires certificate on Route AND TLS configuration on the Pod
- Uses `destinationCACertificate` to validate the backend's certificate
- **Best for**: High-security environments requiring encryption both outside and inside the cluster

---

## 4. insecureEdgeTerminationPolicy Options

| Value | Behavior |
|-------|----------|
| `Redirect` | HTTP requests on port 80 are redirected to HTTPS (port 443). **Recommended for production.** |
| `Allow` | Both HTTP and HTTPS traffic are served. No redirect. |
| `None` | (or empty) HTTP traffic is dropped. Only HTTPS is served. No redirect response. |

Only applies to `edge` and `reencrypt` termination types. Has no effect on `passthrough` routes.

---

## 5. Red Hat Cert-Manager Operator for OpenShift

### Overview

The **cert-manager Operator for Red Hat OpenShift** is a cluster-wide service for application certificate lifecycle management. It provides:
- Certificate provisioning from external CAs
- Automatic certificate renewal
- Certificate retirement
- Self-service certificate capabilities for developers

### Installation (via OperatorHub)

1. Navigate to **Operators > OperatorHub** (requires `cluster-admin`)
2. Search for **"cert-manager Operator for Red Hat OpenShift"**
3. Select the **Red Hat-provided** operator (not community version for production)
4. Click **Install**, accept defaults
5. After operator installation, create a **CertManager** instance under the operator's tab
6. Verify: `oc get pods -n cert-manager`

> **Important**: Do NOT run both the Red Hat cert-manager Operator AND the community cert-manager Operator simultaneously.

### Supported Issuer Types

- ACME (Let's Encrypt, etc.)
- CA (Certificate Authority)
- SelfSigned
- Vault (HashiCorp Vault)
- Venafi
- Google Cloud Certificate Authority Service (CAS)
- Nokia NetGuard Certificate Manager (NCM)

---

## 6. The openshift-routes Controller

### What It Is

The cert-manager/openshift-routes project is a **separate controller** (not part of cert-manager core) that watches OpenShift Route objects for cert-manager annotations and automatically manages TLS certificates.

**GitHub**: https://github.com/cert-manager/openshift-routes
**Current Release**: v0.9.0

### Why It's Separate

The cert-manager project does not support non-Kubernetes APIs in core. OpenShift Route is an OpenShift-specific CRD, so support is maintained as a separate project.

### How It Works

1. Detects annotated Routes
2. Creates a `CertificateRequest` resource referencing the specified Issuer/ClusterIssuer
3. Obtains the signed certificate from the CA
4. **Populates `route.spec.tls` fields automatically** (certificate, key, caCertificate)
5. Handles automatic renewal (2/3 through certificate lifetime, or per `cert-manager.io/renew-before`)

### Installation

Via Helm (must be in the same namespace as cert-manager, default: `cert-manager`):

```bash
helm install openshift-routes -n cert-manager \
  oci://ghcr.io/cert-manager/charts/openshift-routes
```

Or via static manifests:

```bash
oc apply -f <(helm template openshift-routes -n cert-manager \
  oci://ghcr.io/cert-manager/charts/openshift-routes \
  --set omitHelmLabels=true)
```

---

## 7. Route Annotations for Automatic Certificate Issuance

### Minimum Required

```yaml
annotations:
  cert-manager.io/issuer-name: <issuer-name>   # REQUIRED
```

### Full Annotation Set

```yaml
metadata:
  annotations:
    # --- Core ---
    cert-manager.io/issuer-name: my-issuer          # REQUIRED — name of Issuer/ClusterIssuer
    cert-manager.io/issuer-kind: ClusterIssuer       # Optional, defaults to "Issuer"
    cert-manager.io/issuer-group: cert-manager.io    # Optional, defaults to "cert-manager.io"

    # --- Certificate Lifecycle ---
    cert-manager.io/duration: 2160h                  # Optional, defaults to 90 days
    cert-manager.io/renew-before: 720h               # Optional, defaults to 1/3 of duration

    # --- Subject / SAN Configuration ---
    cert-manager.io/common-name: "My Certificate"
    cert-manager.io/alt-names: "a.com,b.com"         # Additional DNS SANs (comma-separated)
    cert-manager.io/ip-sans: "10.0.0.1,10.0.0.2"
    cert-manager.io/uri-sans: "spiffe://td/wl"
    cert-manager.io/email-sans: "a@b.com,c@d.com"

    # --- Private Key ---
    cert-manager.io/private-key-algorithm: ECDSA     # Defaults to RSA
    cert-manager.io/private-key-size: "384"           # Defaults to 256 (ECDSA) or 2048 (RSA)

    # --- Subject Fields ---
    cert-manager.io/subject-organizations: "company"
    cert-manager.io/subject-organizationalunits: "division"
    cert-manager.io/subject-countries: "US"
    cert-manager.io/subject-provinces: "California"
    cert-manager.io/subject-localities: "San Francisco"
    cert-manager.io/subject-postalcodes: "94105"
    cert-manager.io/subject-streetaddresses: "1 Main St"
    cert-manager.io/subject-serialnumber: "123456"
```

### Shorthand Annotations (Ingress-style)

```yaml
cert-manager.io/cluster-issuer: "my-cluster-issuer"  # Shorthand for ClusterIssuer
cert-manager.io/issuer: "my-issuer"                   # Shorthand for namespace Issuer
```

> The Route's `spec.host` is automatically added to the certificate's Subject Alternative Names (SANs).

---

## 8. ClusterIssuer vs Issuer

| Feature | Issuer | ClusterIssuer |
|---------|--------|---------------|
| **Scope** | Namespace-scoped | Cluster-scoped |
| **Can issue certs for** | Routes in the **same namespace** only | Routes in **any namespace** |
| **Secret access** | Same namespace | Cluster Resource Namespace (default: `cert-manager`) |
| **Annotation** | `cert-manager.io/issuer-kind: Issuer` | `cert-manager.io/issuer-kind: ClusterIssuer` |
| **When to use** | Per-team/per-namespace isolation | Shared CA across all namespaces |

### ClusterIssuer Examples

**Let's Encrypt ACME:**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress: {}
```

**Self-Signed:**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

---

## 9. Prerequisites Checklist

Before the cert-manager annotations on a Route will work:

1. **cert-manager installed** — via Red Hat Cert-Manager Operator (OperatorHub) or Helm
   > **This cluster**: Already done. cert-manager Operator v1.18.1 is installed and healthy in `cert-manager-operator` namespace. Pods are running in `cert-manager` namespace.
2. **openshift-routes controller installed** — separate controller, installed via Helm in the cert-manager namespace
   > **Important**: The Red Hat cert-manager Operator does **not** bundle the openshift-routes controller. It must be installed separately via the Helm chart from `oci://ghcr.io/cert-manager/charts/openshift-routes` into the `cert-manager` namespace (see Section 6).
3. **Issuer or ClusterIssuer created and Ready**:
   ```bash
   oc get clusterissuers
   oc get issuers -n <namespace>
   ```
4. **DNS configured** — Route's `spec.host` must resolve to the OpenShift router (on this cluster: `*.apps.cluster-d45v2.dynamic.redhatworkshops.io`)
5. **Network access** — For ACME/HTTP-01: port 80 must be accessible from the internet; ACME server must be reachable from the cluster

---

## 10. Complete Example for Zabbix Web UI

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: admin@your-domain.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress: {}
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: zabbix-web-secure
  namespace: zabbix-system
  annotations:
    cert-manager.io/issuer-name: letsencrypt-prod
    cert-manager.io/issuer-kind: ClusterIssuer
spec:
  host: zabbix-zabbix-system.apps.cluster-d45v2.dynamic.redhatworkshops.io
  to:
    kind: Service
    name: zabbix-demo-web
  port:
    targetPort: 8080
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

After applying, the openshift-routes controller will:
1. Detect the annotations
2. Create a CertificateRequest
3. Populate `spec.tls.certificate`, `spec.tls.key`, `spec.tls.caCertificate` automatically
4. Renew the certificate before expiry

---

## 11. OpenShift-Specific Notes vs Kubernetes Ingress

- **Routes are NOT Kubernetes Ingress**: Routes are OpenShift-specific (`route.openshift.io/v1`). Standard K8s uses `networking.k8s.io/v1 Ingress`.
- **Separate controller needed**: For K8s Ingress, cert-manager's built-in ingress-shim is used. For OpenShift Routes, the separate openshift-routes controller is required.
- **Annotation format differs**: Ingress uses `cert-manager.io/cluster-issuer` (shorthand). Routes canonical format is `cert-manager.io/issuer-name` + `cert-manager.io/issuer-kind`.
- **No `secretName` needed**: Unlike Ingress (which requires `tls.secretName`), the openshift-routes controller injects the certificate directly into the Route's `spec.tls` fields. No separate Secret is created.
- **Route TLS fields auto-populated**: The controller fills `spec.tls.certificate`, `spec.tls.key`, and `spec.tls.caCertificate`. Do NOT pre-populate these.
- **Alternative approach**: You can create a K8s Ingress with cert-manager annotations in OpenShift — it auto-transforms to a Route. However, this gives less control over Route-specific features.

---

## 12. Official Documentation URLs

| Resource | URL |
|----------|-----|
| Red Hat: cert-manager Operator (OCP 4.21) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift |
| Red Hat: cert-manager Operator (OCP 4.20) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift |
| Red Hat: cert-manager Operator (OCP 4.18) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift |
| Red Hat: OpenShift Routes (OCP 4.21) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/ingress_and_load_balancing/routes |
| Red Hat: OpenShift Routes (OCP 4.20) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ingress_and_load_balancing/routes |
| Red Hat Blog: cert-manager Operator | https://www.redhat.com/en/blog/cert-manager-operator-openshift |
| cert-manager.io: Issuer Configuration | https://cert-manager.io/docs/configuration/ |
| cert-manager.io: Requesting Certificates | https://cert-manager.io/docs/usage/ |
| cert-manager.io: Securing Ingress | https://cert-manager.io/docs/usage/ingress/ |
| GitHub: openshift-routes controller | https://github.com/cert-manager/openshift-routes |
| OpenShift Route API Reference (4.15) | https://docs.openshift.com/container-platform/4.15/rest_api/network_apis/route-route-openshift-io-v1.html |
