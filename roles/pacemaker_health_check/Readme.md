# Pacemaker Cluster Health Check Role

This Ansible role performs a comprehensive, automated health audit of a Pacemaker/Corosync High Availability cluster. It generates a consolidated "Executive Report" detailing the health of both individual nodes and the global cluster state.

## Features

* **Holistic Audit:** Checks 9 distinct areas of cluster health, ranging from service status to split-brain detection.
* **Intelligent Execution:**
    * **Local Checks:** Runs on every node (Service status, Config files).
    * **Global Checks:** Elects a single "Active Leader" node to query the cluster status (CIB), preventing redundant commands.
* **Executive Reporting:** Generates a clean, text-based summary separating "Local Node Issues" from "Global Cluster Issues".
* **Safety First:** Does not alter the cluster state; strictly read-only verification.

---

## Requirements

### 1. Common Role Dependency
This role relies on a shared utility role **`pacemaker_common`** to handle the "Active Node Election" logic. Ensure `pacemaker_common` is present in your roles directory.

### 2. Supported Platforms
* RHEL / CentOS / AlmaLinux / Rocky Linux (7, 8, 9)
* SLES (12, 15)

---

## Role Variables

The following variables can be customized in `defaults/main.yml` or overridden in your playbook.

### Service Definitions
| Variable | Default | Description |
|:---|:---|:---|
| `pacemaker_service` | `pacemaker` | Name of the Pacemaker service. |
| `corosync_service` | `corosync` | Name of the Corosync service. |

### Configuration Paths
| Variable | Default | Description |
|:---|:---|:---|
| `corosync_conf_path` | `/etc/corosync/corosync.conf` | Path to the main Corosync config. |
| `cib_xml_path` | `/var/lib/pacemaker/cib/cib.xml` | Path to the Cluster Information Base XML. |

### Notification Settings
| Variable | Default | Description |
|:---|:---|:---|
| `enable_slack_notification`| `false` | Set to `true` to enable Slack alerts on failure. |
| `slack_token` | `""` | Your Slack App Bot Token (xoxb-...). |
| `slack_channel` | `"#ops-alerts"` | The Slack channel to post the report to. |

---

## Health Check Scope

The role performs checks in **9 Sections**:

### Local Node Checks (Runs on ALL Nodes)
* **Section 1: Variable Validation** - Ensures required Ansible variables are defined.
* **Section 2: Service Status** - Verifies `pacemaker` and `corosync` systemd services are `running`.
* **Section 3: Configuration Files** - Checks for the existence of `corosync.conf` and `cib.xml`.

### Global Cluster Checks (Runs on ACTIVE Node only)
* **Section 4: Node Health** - Checks `pcs status` for nodes in `OFFLINE`, `Standby`, or `UNCLEAN` states.
* **Section 5: Global Properties** - Verifies critical properties (e.g., `stonith-enabled: true`) and ensures Maintenance Mode is OFF.
* **Section 6: Stonith Configuration** - Ensures fencing devices are configured and running.
* **Section 7: Failed Actions** - Scans for "Failed Resource Actions" in the cluster status.
* **Section 8: Stonith History** - Checks the fencing history for recent errors or failed fencing attempts.
* **Section 9: Split Brain Detection** - Verifies that all nodes agree on the current DC (Designated Coordinator).

---

## Example Playbook

```yaml
---
- name: Audit Production Cluster
  hosts: cluster_nodes
  become: true
  gather_facts: true

  roles:
    - role: pacemaker_health_check
      vars:
        enable_slack_notification: true
        slack_token: "xoxb-12345-67890-abcdef"