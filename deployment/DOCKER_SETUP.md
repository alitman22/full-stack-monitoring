# Docker Setup Guide

This guide walks you through deploying the full monitoring stack using Docker and Docker Compose.

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- Minimum 4GB RAM available for containers
- 20GB disk space for data volumes

## Quick Start

### 1. Clone Repository

```bash
git clone https://github.com/alitman22/full-stack-monitoring.git
cd full-stack-monitoring
```

### 2. Start the Stack

```bash
docker-compose up -d
```

### 3. Verify Services

```bash
docker-compose ps
```

Expected output:
```
CONTAINER ID   IMAGE                                PORTS                    STATUS
...            prom/prometheus:latest               9090/tcp                 Up 2 minutes
...            grafana/grafana:latest               3000/tcp                 Up 2 minutes
...            prom/alertmanager:latest             9093/tcp                 Up 2 minutes
...            prom/node-exporter:latest            9100/tcp                 Up 2 minutes
...            prom/blackbox-exporter:latest        9115/tcp                 Up 2 minutes
...            postgres:15-alpine                   5432/tcp                 Up 2 minutes
...            prometheuscommunity/postgres-exporter  9187/tcp               Up 2 minutes
...            mongo:latest                         27017/tcp                Up 2 minutes
...            percona/mongodb_exporter:latest      9216/tcp                 Up 2 minutes
```

### 4. Access Services

```
Prometheus: http://localhost:9090
Grafana: http://localhost:3000 (admin/admin)
AlertManager: http://localhost:9093
```

## Configuration

### Update Prometheus Configuration

Edit `prometheus/prometheus.yml` to add your monitoring targets:

```yaml
scrape_configs:
  - job_name: 'my-service'
    static_configs:
      - targets: ['your-service-ip:9090']
```

After editing, reload configuration:

```bash
docker-compose exec prometheus kill -HUP 1
```

### Update AlertManager Configuration

Edit `alertmanager.yml` to configure alerts:

```yaml
route:
  receiver: 'email'
  group_by: ['alertname', 'cluster', 'service']

receivers:
  - name: 'email'
    email_configs:
      - to: 'alerts@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'user@example.com'
        auth_password: 'password'
```

Restart AlertManager:

```bash
docker-compose restart alertmanager
```

## Adding Remote Targets

### Node Exporter on Remote Host

1. Install Node Exporter on target:

```bash
docker run -d \
  --name=node_exporter \
  -p 9100:9100 \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /:/rootfs:ro \
  prom/node-exporter:latest \
  --path.procfs=/host/proc \
  --path.sysfs=/host/sys \
  --path.rootfs=/rootfs
```

2. Add to Prometheus configuration:

```yaml
- job_name: 'node_remote'
  static_configs:
    - targets: ['remote-host-ip:9100']
```

3. Reload Prometheus:

```bash
docker-compose exec prometheus kill -HUP 1
```

## Data Management

### Backup Prometheus Data

```bash
docker-compose exec prometheus tar czf - /prometheus > prometheus_backup.tar.gz
```

### Restore Prometheus Data

```bash
cat prometheus_backup.tar.gz | docker-compose exec -T prometheus tar xzf -
```

### Backup Grafana

```bash
docker cp grafana:/var/lib/grafana grafana_backup
```

### Clean Data

```bash
# Remove all volumes
docker-compose down -v

# Restart fresh
docker-compose up -d
```

## Logging and Debugging

### View Prometheus Logs

```bash
docker-compose logs -f prometheus
```

### View Grafana Logs

```bash
docker-compose logs -f grafana
```

### View AlertManager Logs

```bash
docker-compose logs -f alertmanager
```

### Check Exporter Status

```bash
curl http://localhost:9100/metrics | head -20  # Node Exporter
curl http://localhost:9187/metrics | head -20  # PostgreSQL Exporter
curl http://localhost:9216/metrics | head -20  # MongoDB Exporter
```

## Performance Tuning

### Prometheus Memory Usage

Adjust retention policy in `docker-compose.yml`:

```yaml
command:
  - '--storage.tsdb.retention.time=7d'   # Reduce from 15d
  - '--storage.tsdb.max-block-duration=2h'
```

### Disable Unused Exporters

Comment out services in `docker-compose.yml`:

```yaml
# postgres:
# postgres_exporter:
```

### Increase Scrape Intervals

In `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 30s  # Increase from 15s
  evaluation_interval: 30s
```

## Security

### Change Default Credentials

**Grafana**:
```yaml
environment:
  GF_SECURITY_ADMIN_PASSWORD: your_strong_password
```

**Alert Manager**:
Add HTTP authentication layer (reverse proxy recommended).

### Enable SSL/TLS

Use a reverse proxy (nginx, Caddy) in front of services:

```yaml
nginx:
  image: nginx:alpine
  ports:
    - "443:443"
    - "80:80"
  volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf:ro
    - ./certs:/etc/nginx/certs:ro
  depends_on:
    - prometheus
    - grafana
```

### Network Isolation

Create a separate network for monitoring:

```yaml
networks:
  monitoring:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_ip_masquerade: "true"
```

## Troubleshooting

### Port Already in Use

```bash
# Find process using port
lsof -i :9090

# Kill process or change port in docker-compose.yml
```

### Container Won't Start

```bash
# Check logs
docker-compose logs prometheus

# Recreate container
docker-compose down prometheus
docker-compose up -d prometheus
```

### No Metrics Appearing

```bash
# Check prometheus targets
curl http://localhost:9090/api/v1/targets

# Check scrape success
curl http://localhost:9090/api/v1/query?query=up
```

### Disk Space Issues

```bash
# Check volume sizes
docker system df

# Clean up unused data
docker system prune -a

# Reduce retention period in prometheus config
```

### High Memory Usage

```bash
# Monitor container memory
docker stats prometheus grafana alertmanager

# Reduce retention and adjust scrape intervals
```

## Upgrade

### Upgrade Stack

```bash
# Pull latest images
docker-compose pull

# Restart services
docker-compose up -d
```

### Backup Before Upgrade

```bash
# Create full backup
docker-compose exec prometheus tar czf - /prometheus > backup_$(date +%s).tar.gz
docker cp grafana:/var/lib/grafana backup_grafana_$(date +%s)
```

## Environment Variables

Create `.env` file for customization:

```bash
PROMETHEUS_RETENTION=15d
GRAFANA_ADMIN_PASSWORD=admin
ALERTMANAGER_RECEIVERS=email
POSTGRES_PASSWORD=postgres
POSTGRES_USER=postgres
MONGODB_ROOT_PASSWORD=root
```

Reference in `docker-compose.yml`:

```yaml
postgres:
  environment:
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    POSTGRES_USER: ${POSTGRES_USER}
```

## Next Steps

1. Import Grafana dashboards from `grafana/dashboards/`
2. Configure AlertManager for your notification system
3. Add your monitoring targets to Prometheus
4. Set up Grafana datasources
5. Create custom dashboards
6. Configure alert routes and receivers

## Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Prometheus Docker Documentation](https://hub.docker.com/r/prom/prometheus/)
- [Grafana Docker Documentation](https://hub.docker.com/r/grafana/grafana/)
