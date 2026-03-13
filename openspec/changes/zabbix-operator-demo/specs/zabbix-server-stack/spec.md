## ADDED Requirements

### Requirement: ZabbixFull CR Deploys Complete Server Stack
A ZabbixFull custom resource SHALL cause the Zabbix Operator to create all required server-side components: the Zabbix Server, Zabbix Web Frontend, Zabbix Java Gateway, and a managed PostgreSQL instance in the zabbix-demo namespace.

#### Scenario: ZabbixFull CR creates all server-side components
- **WHEN** a ZabbixFull CR is applied to the zabbix-demo namespace
- **THEN** the operator MUST create a Zabbix Server deployment, a Zabbix Web Frontend deployment, a Zabbix Java Gateway deployment, and a PostgreSQL StatefulSet within the zabbix-demo namespace

### Requirement: PostgreSQL Uses Persistent Storage via Ceph RBD
The managed PostgreSQL instance SHALL use a PersistentVolumeClaim backed by the `ocs-external-storagecluster-ceph-rbd` StorageClass to ensure durable data storage.

#### Scenario: PostgreSQL PVC is provisioned with the correct StorageClass
- **WHEN** the ZabbixFull CR is applied and the PostgreSQL StatefulSet starts
- **THEN** a PersistentVolumeClaim MUST exist in the zabbix-demo namespace with `storageClassName: ocs-external-storagecluster-ceph-rbd` and MUST reach the `Bound` phase

### Requirement: All Components Run in zabbix-demo Namespace
Every workload created by the ZabbixFull CR SHALL be scheduled and running exclusively in the `zabbix-demo` namespace.

#### Scenario: No workload pods land outside zabbix-demo namespace
- **WHEN** the ZabbixFull CR reconciliation is complete
- **THEN** all pods belonging to the Zabbix Server, Web Frontend, Java Gateway, and PostgreSQL MUST have `metadata.namespace: zabbix-demo` and MUST NOT exist in any other namespace

### Requirement: Zabbix Server Trapper Service Exposes Port 10051
The Zabbix Server SHALL be reachable within the cluster on TCP port 10051 (the trapper port) via a ClusterIP Service in the zabbix-demo namespace.

#### Scenario: ClusterIP Service forwards traffic to Zabbix Server on port 10051
- **WHEN** the ZabbixFull CR is applied and all components are running
- **THEN** a ClusterIP Service MUST exist in the zabbix-demo namespace with `port: 10051` and `targetPort: 10051`, and a TCP connection to that Service from within the cluster MUST succeed

### Requirement: Zabbix Web Frontend Service Exposes Port 8080
The Zabbix Web Frontend SHALL be reachable within the cluster on TCP port 8080 via a Service in the zabbix-demo namespace.

#### Scenario: Web frontend Service accepts connections on port 8080
- **WHEN** the ZabbixFull CR is applied and all components are running
- **THEN** a Service MUST exist in the zabbix-demo namespace with `port: 8080` targeting the Web Frontend pods, and an HTTP request to that Service on port 8080 MUST return a non-5xx HTTP response

### Requirement: All Pods Reach Running State
Every pod created by the ZabbixFull CR SHALL reach the `Running` phase with all containers in the `Ready` state within a reasonable reconciliation window.

#### Scenario: All stack pods are Running and Ready after deployment
- **WHEN** the ZabbixFull CR has been applied and sufficient time has elapsed for image pulls and startup
- **THEN** every pod in the zabbix-demo namespace associated with the ZabbixFull CR MUST report `status.phase: Running` and all container statuses MUST report `ready: true`
