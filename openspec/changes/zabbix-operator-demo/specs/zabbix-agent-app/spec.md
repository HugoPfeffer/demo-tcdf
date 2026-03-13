## ADDED Requirements

### Requirement: Deployment Contains App Container and zabbix-agent2 Sidecar
The sample application Deployment SHALL include at least two containers per pod: the primary application container and a `zabbix-agent2` sidecar container, so that the agent runs co-located with the workload being monitored.

#### Scenario: Pod spec contains both the app container and the agent sidecar
- **WHEN** the sample-app Deployment is applied and a pod is scheduled
- **THEN** the pod spec MUST contain at least 2 containers, one of which MUST have `name: zabbix-agent2` (or equivalent agent container name), and the other MUST be the primary application container

### Requirement: Agent Configured with Correct Server and Hostname Parameters
The zabbix-agent2 sidecar SHALL be configured with `Server`, `ServerActive`, and `Hostname` parameters so it can locate the Zabbix Server for both passive and active checks.

#### Scenario: Agent environment or config reflects required parameters
- **WHEN** the zabbix-agent2 container configuration is inspected (env vars or mounted config file)
- **THEN** the `Server` parameter MUST be set to the ClusterIP DNS name of the Zabbix Server Service, `ServerActive` MUST be set to the same DNS name, and `Hostname` MUST be set to `sample-app-agent`

### Requirement: Agent Connects to Server on Port 10051 for Active Checks
The zabbix-agent2 sidecar SHALL establish outbound TCP connections to the Zabbix Server on port 10051 to submit active check results.

#### Scenario: Agent successfully delivers active check data to the server
- **WHEN** the zabbix-agent2 sidecar is running and the Zabbix Server is reachable on port 10051
- **THEN** the agent container logs MUST NOT contain persistent connection-refused or timeout errors for port 10051, and active items for `sample-app-agent` MUST appear with recent `lastvalue` timestamps in the Zabbix frontend

### Requirement: Agent Listens on Port 10050 for Passive Checks
The zabbix-agent2 sidecar SHALL bind to TCP port 10050 inside the pod to accept passive check requests initiated by the Zabbix Server.

#### Scenario: Port 10050 is bound and accessible within the pod network
- **WHEN** the zabbix-agent2 container is running
- **THEN** a TCP connection to the pod IP on port 10050 from within the cluster MUST succeed, and the container spec MUST declare `containerPort: 10050`

### Requirement: Agent Reports Standard Host Metrics
The zabbix-agent2 sidecar SHALL collect and report host-level metrics covering CPU, memory, disk, and network using the standard Zabbix agent item keys.

#### Scenario: CPU metrics are collected via system.cpu.* keys
- **WHEN** the host `sample-app-agent` is registered in Zabbix and monitoring is active
- **THEN** items with keys matching `system.cpu.*` (e.g., `system.cpu.util`) MUST have a non-empty `lastvalue` and a `lastclock` timestamp within the last 5 minutes

#### Scenario: Memory metrics are collected via vm.memory.* keys
- **WHEN** the host `sample-app-agent` is registered in Zabbix and monitoring is active
- **THEN** items with keys matching `vm.memory.*` (e.g., `vm.memory.utilization`) MUST have a non-empty `lastvalue` and a `lastclock` timestamp within the last 5 minutes

#### Scenario: Disk metrics are collected via vfs.fs.* keys
- **WHEN** the host `sample-app-agent` is registered in Zabbix and monitoring is active
- **THEN** items with keys matching `vfs.fs.*` (e.g., `vfs.fs.size[/,used]`) MUST have a non-empty `lastvalue` and a `lastclock` timestamp within the last 5 minutes

#### Scenario: Network metrics are collected via net.if.* keys
- **WHEN** the host `sample-app-agent` is registered in Zabbix and monitoring is active
- **THEN** items with keys matching `net.if.*` (e.g., `net.if.in[eth0]`) MUST have a non-empty `lastvalue` and a `lastclock` timestamp within the last 5 minutes

### Requirement: Agent Hostname Matches Registration in Zabbix Frontend
The `Hostname` value configured in the zabbix-agent2 sidecar SHALL match the host record in the Zabbix Server, either through manual configuration or auto-registration, so that data is attributed to the correct host.

#### Scenario: Host sample-app-agent appears in Zabbix with correct name
- **WHEN** the sample-app pod is running with the zabbix-agent2 sidecar and monitoring has been active for at least one check interval
- **THEN** a host with the name `sample-app-agent` MUST exist in the Zabbix frontend (visible via API or UI), its status MUST be Enabled, and its availability indicator MUST show the ZBX agent as available (green)

### Requirement: Pod Reaches Running State with Both Containers Ready
The sample-app pod SHALL reach the `Running` phase with both the application container and the zabbix-agent2 sidecar reporting `ready: true`.

#### Scenario: Pod is Running with all containers in Ready state
- **WHEN** the sample-app Deployment is applied and the rollout completes
- **THEN** every pod in the Deployment MUST report `status.phase: Running` and MUST have `containerStatuses[*].ready: true` for all containers, with no containers in `CrashLoopBackOff` or `Error` state
