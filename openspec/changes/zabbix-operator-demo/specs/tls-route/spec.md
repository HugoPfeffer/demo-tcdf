## ADDED Requirements

### Requirement: Route Exposes Zabbix Web UI at the Designated Hostname
An OpenShift Route SHALL expose the Zabbix Web Frontend at the hostname `zabbix-demo.apps.cluster-d45v2.dynamic.redhatworkshops.io` so that users can reach the UI from outside the cluster.

#### Scenario: Route is reachable at the configured hostname
- **WHEN** the Route resource is applied and admitted by the OpenShift router
- **THEN** an HTTPS request to `https://zabbix-demo.apps.cluster-d45v2.dynamic.redhatworkshops.io` MUST receive a response with HTTP status 200 and the response body MUST contain the Zabbix login page markup (e.g., the string `Zabbix`)

### Requirement: Route Uses Edge-Terminated TLS
The Route SHALL use edge TLS termination so that TLS is handled at the router and traffic is forwarded to the backend service over HTTP.

#### Scenario: Route termination type is edge
- **WHEN** the Route resource is inspected
- **THEN** `spec.tls.termination` MUST equal `edge` and `spec.tls.insecureEdgeTerminationPolicy` MUST be set to `Redirect` so that plain HTTP requests are not served

### Requirement: Certificate Issued by cert-manager via ClusterIssuer
The TLS certificate for the Route SHALL be issued by cert-manager using a ClusterIssuer configured with a self-signed CA, triggered by cert-manager annotations on the Route.

#### Scenario: cert-manager annotations are present and certificate is issued
- **WHEN** the Route is applied with the appropriate `cert-manager.io/cluster-issuer` annotation referencing a self-signed ClusterIssuer
- **THEN** cert-manager MUST create a Certificate resource for the Route, the Certificate MUST reach `Ready: True` status, and the resulting Secret MUST contain a valid `tls.crt` and `tls.key`

### Requirement: openshift-routes Controller Injects Certificate into Route
The `openshift-routes` cert-manager integration controller SHALL read the issued certificate Secret and inject the TLS fields into the Route object so the OpenShift router serves the correct certificate.

#### Scenario: Route TLS fields are populated by the openshift-routes controller
- **WHEN** the Certificate issued by cert-manager reaches `Ready: True`
- **THEN** the Route's `spec.tls.certificate` and `spec.tls.key` fields MUST be non-empty, populated by the openshift-routes controller, without manual intervention

### Requirement: Route Targets Zabbix Web Service on Port 8080
The Route SHALL forward traffic to the Zabbix Web Frontend Service on port 8080.

#### Scenario: Route backend points to the web service on port 8080
- **WHEN** the Route resource is inspected
- **THEN** `spec.to.kind` MUST be `Service`, `spec.to.name` MUST reference the Zabbix Web Frontend Service, and `spec.port.targetPort` MUST be `8080`

### Requirement: HTTPS Access Returns Zabbix Login Page
An end-to-end HTTPS request to the Route hostname SHALL return the Zabbix Web Frontend login page, confirming TLS, routing, and backend connectivity all function correctly.

#### Scenario: End-to-end HTTPS request returns Zabbix login page
- **WHEN** a client sends `GET https://zabbix-demo.apps.cluster-d45v2.dynamic.redhatworkshops.io/` with TLS verification against the cluster CA
- **THEN** the HTTP response status MUST be 200 and the response body MUST include a recognizable Zabbix login element (e.g., `id="login-form"` or `<title>Zabbix</title>`)
