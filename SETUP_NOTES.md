# Prometheus System Dashboard — Setup Notes

## Prerequisites

- **Grafana** v9.0+ (tested up to v11.x)
- **Prometheus** data source configured in Grafana
- **Node Exporter** (`node_exporter`) running on target machines

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

The dashboard is organized into **5 sections**:

### 1. System Overview (Top Row)
| Panel | Type | Description |
|-------|------|-------------|
| System Uptime | Stat | Time since last boot |
| CPU Cores | Stat | Total logical CPU cores |
| Total RAM | Stat | Total physical memory |
| CPU Usage % | Gauge | Current CPU utilization (thresholds: 70% yellow, 85% red) |
| Memory Usage % | Gauge | Current memory utilization (thresholds: 70% yellow, 85% red) |
| Filesystem Usage % | Gauge | Root filesystem utilization (thresholds: 60% yellow, 80% red) |

### 2. CPU Section
| Panel | Description |
|-------|-------------|
| CPU Usage — Overall | Total CPU % with 85% alert threshold line |
| CPU Usage — By Mode | Stacked breakdown: user, system, iowait, steal, softirq, nice |
| CPU Usage — Per Core | Individual core utilization for load distribution analysis |
| System Load Average | 1m, 5m, 15m load averages |

### 3. Memory Section
| Panel | Description |
|-------|-------------|
| Memory Breakdown (Stacked) | Used, Buffers, Cached, Free |
| Memory Usage % | Percentage with 85% red alert zone |
| Swap Usage | Swap used, free, cached |
| Page Faults | Major and minor fault rates |

### 4. Disk I/O Section
| Panel | Description |
|-------|-------------|
| Disk Throughput | Read/write bytes per second |
| Disk IOPS | Read/write operations per second |
| Filesystem Usage by Mount | Bar gauge per mount point |
| Disk Latency | Average I/O time per operation |

### 5. Network Section
| Panel | Description |
|-------|-------------|
| Network Bandwidth | Receive/transmit in bits/sec |
| Network Packets | Packets per second |
| Network Errors & Drops | Error and drop counters |

## Template Variables

The dashboard includes **3 dropdown filters** at the top:

- **Instance** — Filter by Prometheus target (supports multi-select)
- **Disk Device** — Filter disk panels by device (e.g., sda, nvme0n1)
- **Network Interface** — Filter network panels by interface (e.g., eth0, ens3)

## Alert Thresholds

Built-in visual thresholds (displayed as colored lines/zones on charts):

| Metric | Warning (Yellow) | Critical (Red) |
|--------|-------------------|-----------------|
| CPU Usage % | 70% | 85% |
| Memory Usage % | — | 85% |
| Filesystem Usage % | 60% | 85% |

> **Note:** These are visual thresholds only. To receive actual alert notifications, configure Grafana Alerting rules based on these same queries.

## Customization Tips

- **Change refresh interval:** Top-right corner dropdown (default: 30s)
- **Adjust time range:** Use the time picker (default: Last 1 hour)
- **Add Grafana alerts:** Edit any panel → Alert tab → Create alert rule
- **Change thresholds:** Edit panel → Field config → Thresholds

## Required Node Exporter Metrics

The dashboard uses these metric families (all standard in `node_exporter`):

- `node_cpu_seconds_total`
- `node_load1`, `node_load5`, `node_load15`
- `node_memory_*` (MemTotal, MemFree, MemAvailable, Buffers, Cached, Swap*)
- `node_vmstat_pgfault`, `node_vmstat_pgmajfault`
- `node_disk_*` (read_bytes, written_bytes, reads_completed, writes_completed, io_now, read_time, write_time)
- `node_filesystem_*` (avail_bytes, size_bytes)
- `node_network_*` (receive_bytes, transmit_bytes, receive/transmit packets/errs/drop)
- `node_boot_time_seconds`, `node_time_seconds`
- `node_uname_info`
