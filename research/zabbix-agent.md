# Zabbix 7.0 Agent — Installation & Configuration on RHEL 9

> **Scope**: Installing Zabbix Agent on RHEL 9, configuring `zabbix_agentd.conf` for communication via AWS internal NLB, active vs passive checks, port 10051/10050 usage, SELinux/firewall considerations.

---

## 1. Official Repository for RHEL 9

**Repository RPM:**
```
https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-7.0-1.el9.noarch.rpm
```

**Repository base URL (configured by the RPM):**
```
https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/
```

---

## 2. Complete Installation Steps

```bash
# Step 1: Install the Zabbix 7.0 official repository
dnf install -y https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-7.0-1.el9.noarch.rpm

# Step 2: Clean dnf cache
dnf clean all

# Step 3a: Install Zabbix Agent (legacy, C-based)
dnf install -y zabbix-agent

# Step 3b: OR install Zabbix Agent 2 (modern, Go-based)
# dnf install -y zabbix-agent2

# Step 4: Edit configuration
# vi /etc/zabbix/zabbix_agentd.conf

# Step 5: Enable and start the service
systemctl enable --now zabbix-agent
```

**Additional packages available:**

| Package | Purpose |
|---------|---------|
| `zabbix-agent` | Legacy C-based agent (daemon: `zabbix_agentd`) |
| `zabbix-agent2` | Modern Go-based agent (daemon: `zabbix_agent2`) |
| `zabbix-sender` | CLI utility for sending data to Zabbix trapper |
| `zabbix-get` | CLI utility for querying agent for data (useful for testing) |

---

## 3. Key Configuration Parameters (`zabbix_agentd.conf`)

**Config file location:** `/etc/zabbix/zabbix_agentd.conf`

### Critical Parameters for NLB Setup

| Parameter | Description | Default | NLB Setup Value |
|-----------|-------------|---------|-----------------|
| `Server` | Comma-delimited list of Zabbix server IPs/DNS names. Used for **passive checks** — incoming connections accepted only from these addresses. | `127.0.0.1` | `<AWS_INTERNAL_NLB_DNS_NAME>` |
| `ServerActive` | IP:port of Zabbix server(s) for **active checks**. Agent initiates outbound connections. | `127.0.0.1` | `<AWS_INTERNAL_NLB_DNS_NAME>:10051` |
| `Hostname` | Unique, case-sensitive hostname. **Must match exactly** what is configured in the Zabbix frontend. | System hostname | e.g., `my-rhel9-ec2-instance` |
| `ListenPort` | Port the agent listens on for passive checks. | `10050` | `10050` |
| `StartAgents` | Number of pre-forked agent processes for passive checks. `0` = disable passive checks entirely. | `10` | Range: 0-100 |

### Other Important Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `LogFile` | Log file path | `/var/log/zabbix/zabbix_agentd.log` |
| `LogFileSize` | Max log file size in MB before rotation. 0 = disabled | `1` |
| `PidFile` | PID file path | `/run/zabbix/zabbix_agentd.pid` |
| `AllowKey` | Allow execution of item keys matching pattern (supports wildcards) | All keys allowed |
| `DenyKey` | Deny execution of item keys matching pattern | Not set |
| `Include` | Include additional config files/directory | `/etc/zabbix/zabbix_agentd.d/*.conf` |
| `Timeout` | Max seconds for processing | `3` |
| `BufferSend` | Max seconds to keep data in buffer (active checks) | `5` |
| `BufferSize` | Max values in memory buffer (active checks) | `100` |
| `RefreshActiveChecks` | How often to refresh active check list (seconds) | `5` (Zabbix 7.0) |
| `HostMetadata` | Metadata string for active agent autoregistration (max 2034 bytes) | Empty |
| `HostMetadataItem` | Item to use for metadata (alternative to HostMetadata) | Empty |
| `User` | User to run the agent as | `zabbix` |
| `ListenIP` | IP addresses the agent listens on | `0.0.0.0` |

### Encryption Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `TLSConnect` | How agent connects to server for active checks: `unencrypted`, `psk`, `cert` | `unencrypted` |
| `TLSAccept` | Accepted incoming connections: `unencrypted`, `psk`, `cert` (comma-separated) | `unencrypted` |
| `TLSPSKIdentity` | Pre-shared key identity string | Empty |
| `TLSPSKFile` | Full path to file containing the pre-shared key | Empty |

### Example Configuration for AWS NLB

```ini
# /etc/zabbix/zabbix_agentd.conf

Server=internal-zabbix-nlb-1234567890.us-east-1.elb.amazonaws.com
ServerActive=internal-zabbix-nlb-1234567890.us-east-1.elb.amazonaws.com:10051
Hostname=my-rhel9-ec2-instance

LogFile=/var/log/zabbix/zabbix_agentd.log

ListenIP=0.0.0.0
ListenPort=10050

# Optional: metadata for autoregistration
# HostMetadata=linux rhel9 ec2
```

---

## 4. Zabbix Agent (Legacy) vs Zabbix Agent 2 (Modern)

| Feature | zabbix-agent (Legacy) | zabbix-agent2 (Modern) |
|---------|----------------------|----------------------|
| **Language** | C | Go |
| **Threading** | Single-threaded per process; separate active check process per server/proxy | Multi-threaded single process |
| **Check execution** | Active checks for a single server executed **sequentially** | Checks from different plugins executed **concurrently** |
| **Plugin support** | Limited to UserParameters and external scripts | Native plugin system (Docker, Memcached, MySQL, PostgreSQL, Redis, systemd, etc.) |
| **Config file** | `/etc/zabbix/zabbix_agentd.conf` | `/etc/zabbix/zabbix_agent2.conf` |
| **Service name** | `zabbix-agent` | `zabbix-agent2` |
| **Daemon binary** | `zabbix_agentd` | `zabbix_agent2` |
| **Maturity** | Very stable, long history | Newer, actively developed |

**Recommendation for this demo**: `zabbix-agent` (legacy) is simpler and perfectly adequate for basic OS monitoring via NLB. Choose `zabbix-agent2` if native plugin-based monitoring is needed.

---

## 5. Active vs Passive Checks

### Passive Checks (Server-initiated)

- **Direction**: Server/proxy → Agent
- **Port**: TCP **10050** (agent's `ListenPort`)
- **Flow**: Server connects to agent, requests a specific item value, agent responds
- **Config**: `Server=<server_ip_or_dns>`
- **NLB consideration**: Requires NLB to forward traffic from server to agent's port 10050 — more complex through an NLB
- **Disable with**: `StartAgents=0`

### Active Checks (Agent-initiated)

- **Direction**: Agent → Server/proxy
- **Port**: TCP **10051** (server's trapper port)
- **Flow**:
  1. Agent connects to server on port 10051, requests list of items to monitor (identifies itself by `Hostname`)
  2. Agent collects data locally per the received item list
  3. Agent sends collected data back to server on port 10051
- **Config**: `ServerActive=<server_ip_or_dns>` and `Hostname=<exact_hostname>`
- **NLB consideration**: **Preferred mode** — agent initiates all outbound connections to port 10051 on the NLB. No inbound connectivity to the agent required.
- **Refresh interval**: `RefreshActiveChecks` parameter (default 5s in Zabbix 7.0)

### Recommendation for NLB Architecture

**Use active checks (`ServerActive`) as the primary method.** The agent initiates all connections outbound to port 10051, which is the most NAT/NLB-friendly approach.

---

## 6. Port Usage Summary

| Port | Direction | Protocol | Purpose |
|------|-----------|----------|---------|
| **10050** | Inbound to agent | TCP | Passive checks — server connects to agent |
| **10051** | Outbound from agent | TCP | Active checks — agent connects to server trapper |

For the NLB setup, port **10051** is the critical path. The NLB listener on TCP/10051 forwards to the Zabbix server's trapper port inside OpenShift.

---

## 7. Firewall (firewalld) Configuration

```bash
# For passive checks (if needed): allow inbound on port 10050
firewall-cmd --permanent --add-port=10050/tcp
firewall-cmd --reload

# For active checks only: no inbound rules needed
# (agent initiates outbound connections to port 10051)

# Verify
firewall-cmd --list-ports
```

---

## 8. SELinux Considerations on RHEL 9

RHEL 9 runs SELinux in `enforcing` mode by default. The Zabbix RPM packages include proper SELinux policies.

```bash
# Check for SELinux denials
ausearch -m avc -c zabbix_agentd --raw | audit2allow -a

# Verify Zabbix port labels
semanage port -l | grep zabbix

# If using a non-standard port, label it:
# semanage port -a -t zabbix_agent_port_t -p tcp <custom_port>

# Verify the Zabbix agent process context
ps -eZ | grep zabbix

# Key SELinux boolean for outbound network connections
getsebool -a | grep zabbix
# If zabbix_can_network is off:
setsebool -P zabbix_can_network on

# Verify config file context
ls -lZ /etc/zabbix/zabbix_agentd.conf
# Expected: system_u:object_r:zabbix_agent_conf_t:s0
```

---

## 9. Systemd Service Management

```bash
# Enable and start
systemctl enable --now zabbix-agent

# Status / start / stop / restart
systemctl status zabbix-agent
systemctl start zabbix-agent
systemctl stop zabbix-agent
systemctl restart zabbix-agent

# Reload configuration (SIGHUP)
systemctl reload zabbix-agent

# View logs
journalctl -u zabbix-agent -f
journalctl -u zabbix-agent --since today

# Agent log file
tail -f /var/log/zabbix/zabbix_agentd.log

# Test the agent (from a machine with zabbix-get)
zabbix_get -s <agent_ip> -k agent.ping
zabbix_get -s <agent_ip> -k agent.version
zabbix_get -s <agent_ip> -k system.hostname
```

---

## 10. Zabbix 7.0 Version-Specific Notes

- **Zabbix 7.0 is an LTS release** — supported until June 2029
- Latest patch: **7.0.24**
- New in 7.0: browser items, proxy load balancing/HA, cause-symptom problem linking, incremental config sync for agents
- Active checks protocol now includes `config_revision` and `session` fields for incremental sync
- Agent variant field added to active check protocol (`variant=1` for agent, `variant=2` for agent2)
- `RefreshActiveChecks` default changed from 120s (older versions) to shorter interval

---

## 11. Official Documentation URLs

| Resource | URL |
|----------|-----|
| Zabbix 7.0 Manual (root) | https://www.zabbix.com/documentation/7.0/en/manual |
| Installation from packages | https://www.zabbix.com/documentation/7.0/en/manual/installation/install_from_packages |
| Agent concept | https://www.zabbix.com/documentation/7.0/en/manual/concepts/agent |
| Agent 2 concept | https://www.zabbix.com/documentation/7.0/en/manual/concepts/agent2 |
| Agent vs Agent 2 comparison | https://www.zabbix.com/documentation/7.0/en/manual/appendix/agent_comparison |
| `zabbix_agentd.conf` reference | https://www.zabbix.com/documentation/7.0/en/manual/appendix/config/zabbix_agentd |
| `zabbix_agent2.conf` reference | https://www.zabbix.com/documentation/7.0/en/manual/appendix/config/zabbix_agent2 |
| Active/Passive checks | https://www.zabbix.com/documentation/7.0/en/manual/appendix/items/activepassive |
| Active agent autoregistration | https://www.zabbix.com/documentation/7.0/en/manual/discovery/auto_registration |
| Restricting agent checks | https://www.zabbix.com/documentation/7.0/en/manual/config/items/restrict_checks |
| Encryption (TLS/PSK) | https://www.zabbix.com/documentation/7.0/en/manual/encryption |
| RHEL upgrade procedure | https://www.zabbix.com/documentation/7.0/en/manual/installation/upgrade/packages/rhel |
| Agent item keys list | https://www.zabbix.com/documentation/7.0/en/manual/config/items/itemtypes/zabbix_agent |
| What's new in 7.0 | https://www.zabbix.com/documentation/7.0/en/manual/introduction/whatsnew700 |
| Zabbix Download page | https://www.zabbix.com/download |
| GitHub: zabbix_agentd.conf | https://github.com/zabbix/zabbix/blob/master/conf/zabbix_agentd.conf |
