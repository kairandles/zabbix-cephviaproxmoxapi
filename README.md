# Proxmox Ceph API — Zabbix Template

A Zabbix 7.0 template for monitoring a Ceph cluster via the Proxmox REST API. Rather than requiring the Zabbix agent on every Ceph node, the template makes HTTP calls to a single Proxmox node (or load-balancer VIP) and derives all metrics from the API responses.

> **Recommended companion:** This template is designed to complement the official [Proxmox via HTTP](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/app/proxmox?at=release/7.4) template from the Zabbix template library. That template covers Proxmox node and VM/LXC metrics (CPU, memory, disk, network per node/guest); this template adds the Ceph-specific layer on top. Both use the same API token and host macros, so they can be applied to the same host without additional configuration.

## Requirements

- Zabbix 7.0 or later
- A Proxmox VE node reachable from the Zabbix server/proxy on port 8006
- A Proxmox API token with read-only access to the cluster and node Ceph endpoints

## Setup

### 1. Create a Proxmox API token

The easiest way to do this is via the Proxmox web UI:

1. Log in to the Proxmox web interface and navigate to **Datacenter → Permissions → Users**
2. Add a new user — e.g. `zabbix@pve` — with any realm and a strong password
3. Go to **Datacenter → Permissions → Add → User Permission**, set the path to `/`, select `zabbix@pve`, and assign the **PVEAuditor** role
4. Navigate to **Datacenter → Permissions → API Tokens → Add**, select `zabbix@pve`, give the token a name (e.g. `ZabbixMonitoring`), and uncheck **Privilege Separation**
5. Copy the token secret displayed — it is only shown once

### 2. Import the template

In Zabbix: **Configuration → Templates → Import** → select `CephZabbixAPI.yaml`.

### 3. Apply the template to a host

Create a host representing your Proxmox cluster (or use an existing Proxmox host) and link the **Proxmox Ceph API** template to it.

### 4. Set the required macros

Set the following macros on the host (or at a higher-level host group):

| Macro | Example value | Description |
|---|---|---|
| `{$PVE.URL.HOST}` | `10.181.55.15` | Proxmox host or LB address — no port, no protocol |
| `{$PVE.NODE}` | `pve01` | Proxmox node name used for node-scoped API calls |
| `{$PVE.TOKEN.ID}` | `zabbix@pve!ZabbixMonitoring` | API token in `user@realm!tokenname` format |
| `{$PVE.TOKEN.SECRET}` | `xxxxxxxx-xxxx-…` | API token secret UUID (stored as secret text) |

> **Note:** `{$PVE.NODE}` is used in the OSD, MON, MGR, and pool raw data URLs but is not pre-populated in the template. You must set it manually. Any in-cluster node works; the cluster-scoped endpoints (`/cluster/ceph/status`) are unaffected by which node is queried.

### 5. (Optional) Tune the threshold macros

The template ships with sensible defaults. Override on the host or host group as needed:

| Macro | Default | Description |
|---|---|---|
| `{$CEPH.CLUSTER.USAGE.WARN}` | `70` | Cluster raw usage % warning |
| `{$CEPH.CLUSTER.USAGE.HIGH}` | `85` | Cluster raw usage % high |
| `{$CEPH.OSD.USAGE.WARN}` | `75` | Per-OSD usage % warning |
| `{$CEPH.OSD.USAGE.HIGH}` | `85` | Per-OSD usage % high/critical |
| `{$CEPH.OSD.LATENCY.WARN}` | `50` | OSD apply/commit latency warning (ms) |
| `{$CEPH.POOL.USAGE.WARN}` | `70` | Per-pool usage % warning |
| `{$CEPH.POOL.USAGE.HIGH}` | `85` | Per-pool usage % high/critical |

## What is monitored

### Cluster-level metrics

| Item | Key | Notes |
|---|---|---|
| Health Status | `ceph.cluster.health` | `HEALTH_OK`, `HEALTH_WARN`, or `HEALTH_ERR` |
| Health Checks | `ceph.cluster.health.checks` | Raw JSON of active health check codes |
| OSDs Total / Up / In | `ceph.cluster.osds.*` | From the OSD map |
| PGs Total / Active+Clean / Degraded / Backfilling / Recovering / Stale | `ceph.cluster.pgs.*` | |
| Read/Write IOPS | `ceph.cluster.io.read_ops` / `write_ops` | ops/s |
| Read/Write Throughput | `ceph.cluster.io.read_bytes` / `write_bytes` | bytes/s |
| Storage Total / Used / Available / Used % | `ceph.cluster.storage.*` | |
| Safe Nearfull Ratio % | `ceph.cluster.safe.nearfull.pct` | Calculated — see below |
| Largest Node Raw Capacity | `ceph.cluster.node.raw.max` | Used in nearfull calculation |
| Active MGR Count | `ceph.cluster.mgr.active_count` | |
| MON Quorum Age | `ceph.cluster.mons.quorum_age` | Seconds |
| Cluster FSID | `ceph.cluster.fsid` | |

### Discovered resources

The template auto-discovers and creates per-resource items via Zabbix LLD:

**OSDs** — per OSD: status (up/down), in flag, usage %, bytes used, PG count, apply latency (ms), commit latency (ms)

**MONs** — per MON: state, in-quorum flag

**MGRs** — per MGR: state (active/standby)

**Pools** — per pool: bytes used, usage %, PG count, replication size, min size, PG autoscale mode

## Triggers

### Cluster triggers

| Severity | Trigger |
|---|---|
| DISASTER | Cluster health is `HEALTH_ERR` |
| DISASTER | Cluster storage usage exceeds safe nearfull ratio |
| HIGH | One or more OSDs are down |
| HIGH | No active MGR |
| HIGH | One or more stale PGs |
| AVERAGE | Cluster health is `HEALTH_WARN` |
| AVERAGE | Cluster storage approaching safe nearfull ratio (>85% of limit) |
| AVERAGE | One or more degraded PGs |

### Per-OSD triggers (LLD)

| Severity | Trigger |
|---|---|
| HIGH | OSD is DOWN |
| HIGH | OSD usage exceeds `{$CEPH.OSD.USAGE.HIGH}` % |
| AVERAGE | OSD is UP but marked OUT |
| WARNING | OSD apply or commit latency exceeds `{$CEPH.OSD.LATENCY.WARN}` ms |
| WARNING | OSD usage exceeds `{$CEPH.OSD.USAGE.WARN}` % |

### Per-MON triggers (LLD)

| Severity | Trigger |
|---|---|
| HIGH | MON is not running |
| HIGH | MON is not in quorum |

### Per-MGR triggers (LLD)

| Severity | Trigger |
|---|---|
| INFO | MGR state changed (active ↔ standby failover) |

### Per-pool triggers (LLD)

| Severity | Trigger |
|---|---|
| AVERAGE | Pool usage exceeds `{$CEPH.POOL.USAGE.HIGH}` % |
| AVERAGE | Pool usage exceeds `{$CEPH.POOL.USAGE.WARN}` % |
| AVERAGE | Pool replication size is less than 3 |
| INFO | Pool PG autoscale mode changed |

## Safe nearfull ratio explained

The template calculates a topology-aware capacity ceiling:

```
safe_nearfull_pct = (total_raw - largest_single_node_raw) / total_raw × 100
```

This is the highest raw utilisation at which Ceph can still fully rebalance data after losing the largest node in the cluster. It is a better alarm point than a fixed percentage because it automatically adjusts to asymmetric clusters and changes as nodes are added or removed.

The DISASTER trigger fires when actual usage exceeds this value. The AVERAGE trigger fires at 85% of the limit as an early warning.

## How the template works

Five master items poll the Proxmox REST API every minute using `HTTP_AGENT`:

| Master item key | Proxmox endpoint |
|---|---|
| `proxmox.ceph.raw.status` | `/api2/json/cluster/ceph/status` |
| `proxmox.ceph.raw.osd` | `/api2/json/nodes/{$PVE.NODE}/ceph/osd` |
| `proxmox.ceph.raw.mon` | `/api2/json/nodes/{$PVE.NODE}/ceph/mon` |
| `proxmox.ceph.raw.mgr` | `/api2/json/nodes/{$PVE.NODE}/ceph/mgr` |
| `proxmox.ceph.raw.pool` | `/api2/json/nodes/{$PVE.NODE}/ceph/pool` |

All other items are `DEPENDENT` — they extract specific values from the master item JSON using JSONPath or JavaScript preprocessing. This means Proxmox is queried only five times per minute regardless of how many OSDs, pools, or nodes exist in the cluster. SSL peer and host verification is disabled to accommodate self-signed Proxmox certificates.
