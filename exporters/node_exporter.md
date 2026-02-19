# Node Exporter Setup Guide

**Node Exporter** is the Prometheus exporter for machine metrics. It collects CPU, memory, disk, network, and other system-level metrics from Linux and Unix systems.

## Overview

- **Metrics**: 200+ system metrics
- **Target Port**: `:9100`
- **Setup Complexity**: ⭐ (Simple)
- **Use Case**: Essential monitoring for all Linux/Unix servers

## What It Monitors

### CPU
- CPU seconds by mode (user, system, idle, iowait, etc.)
- Context switches and interrupts
- Load average (1m, 5m, 15m)
- CPU frequency scaling

### Memory
- Total, free, available, used memory
- Buffers and cached memory
- Swap usage
- Memory pressure and page faults

### Disk
- Filesystem usage (by device, mountpoint)
- Filesystem I/O (read/write operations, bytes)
- I/O time and weighted I/O time
- Disk queue length and operations in progress

### Network
- Transmit/receive bytes and packets
- Errors, dropped packets
- Network collisions
- Interface state

### System
- Boot time
- Number of processes
- Uptime

## Installation

### Docker (Recommended)

```bash
docker run -d \
  --name=node_exporter \
  -p 9100:9100 \
  --restart unless-stopped \
  prom/node-exporter:latest \
  --path.procfs=/host/proc \
  --path.rootfs=/rootfs \
  --path.sysfs=/host/sys \
  --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/) \
  --collector.netdev.device-exclude=^(veth.*|br.*)$$
```

### Docker Compose

```yaml
node_exporter:
  image: prom/node-exporter:latest
  container_name: node_exporter
  ports:
    - "9100:9100"
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    - /:/rootfs:ro
  command:
    - '--path.procfs=/host/proc'
    - '--path.sysfs=/host/sys'
    - '--path.rootfs=/rootfs'
    - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    - '--collector.netdev.device-exclude=^(veth.*|br.*)$$'
  restart: unless-stopped
```

### Systemd Service

1. Download the binary:
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.6.1.linux-amd64.tar.gz
sudo cp node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false node_exporter
```

2. Create systemd service (`/etc/systemd/system/node_exporter.service`):
```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/) \
  --collector.netdev.device-exclude=^(veth.*|br.*)$$

Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

3. Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

## Prometheus Configuration

```yaml
- job_name: 'node_exporter'
  static_configs:
    - targets: ['localhost:9100']
```

## Key PromQL Queries

### CPU Usage
```promql
# CPU usage percentage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# CPU utilization by mode
irate(node_cpu_seconds_total[5m])

# Load average
node_load5
```

### Memory Usage
```promql
# Memory available percentage
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Memory used percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Swap usage
(node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes) / node_memory_SwapTotal_bytes
```

### Disk Usage
```promql
# Disk usage percentage
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100

# Disk I/O read rate (bytes/sec)
irate(node_disk_read_bytes_total[5m])

# Disk I/O write rate (bytes/sec)
irate(node_disk_written_bytes_total[5m])
```

### Network
```promql
# Network traffic (bytes/sec)
irate(node_network_receive_bytes_total[5m])

# Network errors
irate(node_network_receive_errs_total[5m])

# Network dropped packets
irate(node_network_receive_drop_total[5m])
```

## Command Line Options

```bash
# Enable/disable collectors
--collector.loadavg  # Enable load average
--collector.meminfo  # Enable memory info
--collector.netdev   # Enable network devices
--collector.diskstats # Enable disk stats

# Exclude certain collectors
--no-collector.cpufreq

# Path configurations
--path.sysfs=/sys           # Path to sysfs filesystem
--path.procfs=/proc         # Path to procfs filesystem
--path.rootfs=/              # Path to root filesystem
```

## Alerting Rules

```yaml
- alert: HighCPUUsage
  expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
  for: 5m

- alert: HighMemoryUsage
  expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
  for: 5m

- alert: HighDiskUsage
  expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100 > 80
  for: 5m
```

## Troubleshooting

### Port Already in Use
```bash
sudo lsof -i :9100
```

### Permission Denied
Ensure `node_exporter` user has read access to `/proc` and `/sys`.

### Missing Metrics
Check enabled collectors:
```bash
curl http://localhost:9100/metrics | grep "node_" | head -20
```

### High Memory Usage
Disable unused collectors:
```bash
node_exporter --no-collector.netdev --no-collector.diskstats
```

## Best Practices

1. **Run as non-root**: Use systemd service with dedicated user
2. **Disable unused collectors**: Reduces memory footprint
3. **Set appropriate scrape intervals**: 15-60s typically
4. **Monitor the exporter itself**: Create alerts for exporter availability
5. **Use filesystem mounts**: For containerized setups, properly mount procfs

## Related Documentation

- [Official Node Exporter GitHub](https://github.com/prometheus/node_exporter)
- [Prometheus Exporters List](https://prometheus.io/docs/instrumenting/exporters/)
