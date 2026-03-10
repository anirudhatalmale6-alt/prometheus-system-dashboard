# Prometheus System Dashboard — Setup Notes (v2)

## Prerequisites

- **Grafana** v9.0+ (tested up to v11.x)
- **Prometheus** data source configured in Grafana
- **Node Exporter** (`node_exporter`) running on target machines
- **Pushgateway** (for backup status panel)

## Import Instructions

### Option 1: Import via Grafana UI

1. Open Grafana → **Dashboards** → **New** → **Import**
2. Click **"Upload dashboard JSON file"** and select `prometheus-system-dashboard.json`
3. Under **"Prometheus"** dropdown, select your Prometheus data source
4. Click **Import**

### Option 2: Import via API

```bash
curl -X POST http://your-grafana:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d @prometheus-system-dashboard.json
```

## Dashboard Structure

The dashboard is organized into **6 sections**:

### 1. Backup Status (Top Row)
| Panel | Type | Description |
|-------|------|-------------|
| Backup Age | Stat | Green "OK" if backup completed within 24h, Red "FAIL" otherwise. Uses `backup_timestamp` metric from Pushgateway. |

**Query:** `(time() - backup_timestamp) < bool 86400`
**Value mapping:** 1 = OK (green), 0 = FAIL (red)
**Legend:** `{{host}}` — shows one box per host pushing to Pushgateway

### 2. Per-Instance Overview (Repeating Row)
Each instance gets its own row with three stat panels:

| Panel | Description |
|-------|-------------|
| DISK | Root filesystem usage % (green/yellow/orange/red thresholds) |
| CPU | Current CPU utilization % with sparkline |
| RAM | Current memory utilization % with sparkline |

This row **repeats automatically** for each value of the `$instance` variable.

### 3. CPU Detail
| Panel | Description |
|-------|-------------|
| CPU Usage — All Instances | Time series with 85% threshold line |
| System Load Average | 1m, 5m, 15m load averages per instance |

### 4. Memory Detail
| Panel | Description |
|-------|-------------|
| Memory Usage % — All Instances | Time series with 85% red alert zone |
| Swap Usage | Swap used/free per instance |

### 5. Disk Detail
| Panel | Description |
|-------|-------------|
| Filesystem Usage by Mount | Bar gauge per mount point (color-coded) |
| Disk I/O Throughput | Read/write bytes per second |

### 6. Network
| Panel | Description |
|-------|-------------|
| Network Bandwidth | Receive/transmit in bits/sec |
| Network Errors & Drops | Error and drop counters |

## Backup Status — Pushgateway Setup

The backup panel expects a `backup_timestamp` metric pushed via Pushgateway.

**Example push script (add to your cron job after backup completes):**

```bash
#!/bin/bash
# Push backup timestamp to Pushgateway after successful backup
PUSHGATEWAY_URL="http://your-pushgateway:9091"
HOST_LABEL="local"  # or "remote", etc.

cat <<EOF | curl --data-binary @- ${PUSHGATEWAY_URL}/metrics/job/backup/host/${HOST_LABEL}
# HELP backup_timestamp Unix timestamp of last successful backup
# TYPE backup_timestamp gauge
backup_timestamp $(date +%s)
EOF
```

Run this at the end of each backup cron job. The dashboard will show:
- **Green "OK"** if `backup_timestamp` is within the last 24 hours (86400 seconds)
- **Red "FAIL"** if the backup is older than 24h or the metric is missing

## Template Variables

The dashboard includes **3 dropdown filters** at the top:

- **Instance** — Filter by Prometheus target (supports multi-select, repeats the overview row)
- **Disk Device** — Filter disk panels by device (e.g., sda, nvme0n1)
- **Network Interface** — Filter network panels by interface (e.g., eth0, ens3)

## Alert Thresholds

| Metric | Warning (Yellow) | Critical (Red) |
|--------|-------------------|-----------------|
| CPU Usage % | 70% | 85% |
| Memory Usage % | — | 85% |
| Filesystem Usage % | 60% | 85% |
| Backup Age | — | > 24 hours |

## Required Metrics

**From node_exporter (standard):**
- `node_cpu_seconds_total`, `node_load1/5/15`
- `node_memory_*` (MemTotal, MemFree, MemAvailable, Buffers, Cached, Swap*)
- `node_disk_*` (read_bytes, written_bytes, io_now)
- `node_filesystem_*` (avail_bytes, size_bytes)
- `node_network_*` (receive/transmit bytes, packets, errs, drop)
- `node_uname_info`

**From Pushgateway:**
- `backup_timestamp` (gauge, with `host` label)
