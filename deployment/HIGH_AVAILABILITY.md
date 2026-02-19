# High Availability Setup Guide

This guide covers configuring the monitoring stack for high availability in production environments.

## Overview

High availability monitoring requires:
- Multiple Prometheus instances
- Shared storage or remote storage backend
- Load balancing
- Failover capability
- Data redundancy

## Architecture

```
┌─────────────────────────────────────────────────┐
│           Load Balancer (HAProxy)               │
│  Distributes traffic across Prometheus instances│
└──────────────────┬──────────────────────────────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
    ┌───▼──┐   ┌──▼───┐   ┌──▼───┐
    │Prom  │   │Prom  │   │Prom  │
    │  1   │   │  2   │   │  3   │
    └───┬──┘   └──┬───┘   └──┬───┘
        │         │         │
        └─────────┼─────────┘
                  │
        ┌─────────▼─────────┐
        │ Remote Storage    │
        │ (S3, MinIO, etc)  │
        └───────────────────┘
```

## Multi-Prometheus Setup

### Docker Compose Configuration

```yaml
version: '3.8'

services:
  # Load Balancer
  nginx:
    image: nginx:alpine
    container_name: nginx_lb
    ports:
      - "9090:9090"
      - "9093:9093"
      - "3000:3000"
    volumes:
      - ./nginx_ha.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - prometheus1
      - prometheus2
      - alertmanager1
      - alertmanager2
      - grafana
    restart: unless-stopped

  # Prometheus Instance 1
  prometheus1:
    image: prom/prometheus:latest
    container_name: prometheus1
    expose:
      - "9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus1_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.external-url=http://prometheus:9090'
      - '--web.enable-lifecycle'
    environment:
      - SHARD=1
      - SHARDS=2
    restart: unless-stopped

  # Prometheus Instance 2
  prometheus2:
    image: prom/prometheus:latest
    container_name: prometheus2
    expose:
      - "9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus2_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.external-url=http://prometheus:9090'
      - '--web.enable-lifecycle'
    environment:
      - SHARD=2
      - SHARDS=2
    restart: unless-stopped

  # AlertManager Instance 1
  alertmanager1:
    image: prom/alertmanager:latest
    container_name: alertmanager1
    expose:
      - "9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager1_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--cluster.listen-address=0.0.0.0:9094'
      - '--cluster.peer=alertmanager2:9094'
    restart: unless-stopped

  # AlertManager Instance 2
  alertmanager2:
    image: prom/alertmanager:latest
    container_name: alertmanager2
    expose:
      - "9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager2_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--cluster.listen-address=0.0.0.0:9094'
      - '--cluster.peer=alertmanager1:9094'
    restart: unless-stopped

  # Grafana (Single instance, backed by HA storage)
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    expose:
      - "3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_DATABASE_TYPE: mysql
      GF_DATABASE_HOST: mysql_db:3306
      GF_DATABASE_NAME: grafana
      GF_DATABASE_USER: grafana
      GF_DATABASE_PASSWORD: grafana_password
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    depends_on:
      - mysql_db
    restart: unless-stopped

  # MySQL for Grafana Storage
  mysql_db:
    image: mysql:8
    container_name: mysql_db
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: grafana
      MYSQL_USER: grafana
      MYSQL_PASSWORD: grafana_password
    volumes:
      - mysql_data:/var/lib/mysql
    restart: unless-stopped

volumes:
  prometheus1_data:
  prometheus2_data:
  alertmanager1_data:
  alertmanager2_data:
  mysql_data:
```

### Nginx Load Balancer Configuration

```nginx
# /nginx_ha.conf

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    # Prometheus Upstream
    upstream prometheus_backend {
        server prometheus1:9090 max_fails=3 fail_timeout=30s;
        server prometheus2:9090 max_fails=3 fail_timeout=30s;
    }

    # AlertManager Upstream
    upstream alertmanager_backend {
        server alertmanager1:9093 max_fails=3 fail_timeout=30s;
        server alertmanager2:9093 max_fails=3 fail_timeout=30s;
    }

    # Grafana Upstream
    upstream grafana_backend {
        server grafana:3000 max_fails=3 fail_timeout=30s;
    }

    # Prometheus
    server {
        listen 9090;
        server_name _;

        location / {
            proxy_pass http://prometheus_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Health check endpoint
            access_log off;
        }

        location /metrics {
            proxy_pass http://prometheus_backend/metrics;
        }
    }

    # AlertManager
    server {
        listen 9093;
        server_name _;

        location / {
            proxy_pass http://alertmanager_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

    # Grafana
    server {
        listen 3000;
        server_name _;

        location / {
            proxy_pass http://grafana_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

## Remote Storage Configuration

### S3 Remote Storage

Edit `prometheus/prometheus.yml`:

```yaml
remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
    write_relabel_configs:
      - source_labels: [__name__]
        regex: '.+'
        action: keep

remote_read:
  - url: "http://thanos-query:9090"
    read_recent: true
```

### MinIO Configuration

```bash
docker run -d \
  --name minio \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin \
  -p 9000:9000 \
  -p 9001:9001 \
  minio/minio:latest server /data
```

## AlertManager Clustering

AlertManager instances communicate via gossip protocol for clustering:

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

receivers:
  - name: 'default'
    email_configs:
      - to: 'alerts@example.com'
```

Start with clustering:

```bash
docker run -d \
  --name alertmanager1 \
  prom/alertmanager:latest \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager2:9094
```

## Failover Testing

### Test Prometheus Failover

```bash
# Stop prometheus1
docker-compose stop prometheus1

# Monitor should automatically use prometheus2
curl http://localhost:9090/api/v1/query?query=up
```

### Test AlertManager Failover

```bash
# Stop alertmanager1
docker-compose stop alertmanager1

# Alerts should be routed through alertmanager2
# Check cluster status
curl http://localhost:9093/api/v1/status
```

## Backup Strategy

### Regular Backups

```bash
#!/bin/bash
# backup_ha.sh

BACKUP_DIR="/backups/monitoring_$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# Backup Prometheus data
docker-compose exec -T prometheus1 tar czf - /prometheus > $BACKUP_DIR/prometheus1.tar.gz
docker-compose exec -T prometheus2 tar czf - /prometheus > $BACKUP_DIR/prometheus2.tar.gz

# Backup AlertManager data
docker-compose exec -T alertmanager1 tar czf - /alertmanager > $BACKUP_DIR/alertmanager1.tar.gz
docker-compose exec -T alertmanager2 tar czf - /alertmanager > $BACKUP_DIR/alertmanager2.tar.gz

# Backup configurations
cp prometheus/prometheus.yml $BACKUP_DIR/
cp alertmanager.yml $BACKUP_DIR/

# Upload to S3
aws s3 cp $BACKUP_DIR s3://monitoring-backups/ --recursive

echo "Backup completed: $BACKUP_DIR"
```

## Monitoring the HA Stack

### Health Check Queries

```promql
# Prometheus availability
up{job="prometheus"}

# AlertManager availability
up{job="alertmanager"}

# Cluster member count
alertmanager_cluster_members

# Gossiped messages
alertmanager_cluster_messages_sent_total
```

## Scaling Considerations

### Add More Prometheus Instances

1. Create new instance in compose file
2. Update nginx upstream configuration
3. Update alertmanager clustering configuration
4. Reload services

### Load Distribution

Use metric relabeling to shard metrics across instances:

```yaml
scrape_configs:
  - job_name: 'node_shard_1'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __tmp_shard
      - target_label: __tmp_shard
        modulus: 2
        action: mod
      - source_labels: [__tmp_shard]
        regex: '1'
        action: keep
```

## Best Practices

1. **Regular backups** - Daily to offsite storage
2. **Monitor components** - Set alerts for exporter health
3. **Test failover** - Monthly failover drills
4. **Keep configs synced** - Use Git for version control
5. **Monitor replication lag** - If using remote storage
6. **Log aggregation** - Collect logs from all instances
7. **Disaster recovery plan** - Document procedures
8. **Capacity planning** - Monitor growth over time

## Troubleshooting HA Issues

### Cluster Not Forming

```bash
# Check cluster status
curl http://localhost:9093/api/v1/status

# View clustering debug logs
docker-compose logs alertmanager1 | grep cluster
```

### Data Inconsistency

```bash
# Compare Prometheus snapshots
docker-compose exec prometheus1 ls -la /prometheus/snapshots/
```

### Network Partition

Implement alerting for cluster member changes:

```yaml
- alert: AlertManagerClusterDown
  expr: alertmanager_cluster_members < 2
  for: 5m
```

## Related Documentation

- [Prometheus Remote Storage](https://prometheus.io/docs/prometheus/latest/storage/#remote-storage-integrations)
- [AlertManager High Availability](https://prometheus.io/docs/alerting/latest/configuration/)
- [Thanos Documentation](https://thanos.io/)
