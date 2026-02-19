# Troubleshooting Guide

Common issues and their solutions when running the full-stack monitoring setup.

## Prometheus

### Prometheus Not Starting

**Symptoms**: Container fails to start, or crashes immediately

```bash
# Check logs
docker-compose logs prometheus
```

**Common Causes & Solutions**:

1. **Configuration Error**
   ```bash
   # Validate configuration
   docker run --rm -it -v $(pwd)/prometheus:/etc/prometheus prom/prometheus:latest \
     prometheus --config.file=/etc/prometheus/prometheus.yml --dry-run
   ```
   Fix any YAML syntax errors in `prometheus/prometheus.yml`

2. **Permission Denied**
   ```bash
   # Fix volume permissions
   sudo chown -R 65534:65534 prometheus_data/
   ```

3. **Port Already in Use**
   ```bash
   # Find process using port 9090
   lsof -i :9090
   
   # Or change port in docker-compose.yml
   ports:
     - "9091:9090"  # Map to different port
   ```

### No Metrics Appearing

**Symptoms**: Prometheus targets show "DOWN" or up=0

```bash
# Check target status
curl http://localhost:9090/api/v1/targets
```

**Solutions**:

1. **Exporter Not Running**
   ```bash
   # Test exporter connectivity
   curl http://exporter-host:9100/metrics
   
   # If connection refused, start the exporter
   docker-compose up -d node_exporter
   ```

2. **Firewall Blocking**
   ```bash
   # Check if port is open
   nc -zv exporter-host 9100
   
   # Open port if needed
   sudo ufw allow 9100/tcp
   ```

3. **Incorrect IP/Hostname**
   - Verify addresses in `prometheus/prometheus.yml`
   - Use IP addresses instead of hostnames if DNS resolution fails
   - Test DNS resolution: `nslookup hostname`

4. **Scrape Timeout**
   ```yaml
   # Increase timeout in prometheus.yml if needed
   scrape_configs:
     - job_name: 'node'
       scrape_timeout: 15s
       static_configs:
         - targets: ['localhost:9100']
   ```

### High Memory Usage

**Symptoms**: Docker stats show high memory consumption

```bash
docker stats prometheus
```

**Solutions**:

1. **Reduce Retention**
   ```yaml
   command:
     - '--storage.tsdb.retention.time=7d'  # Reduce from 15d
   ```

2. **Reduce Scrape Frequency**
   ```yaml
   global:
     scrape_interval: 30s  # Increase from 15s
   ```

3. **Filter Unnecessary Metrics**
   ```yaml
   metric_relabel_configs:
     - source_labels: [__name__]
       regex: 'unwanted_metric'
       action: drop
   ```

### Queries Return No Data

**Symptoms**: Valid queries return empty results

```bash
# Test if data exists
curl "http://localhost:9090/api/v1/query?query=up"
```

**Solutions**:

1. **Metrics Not Scraped Yet**
   - Wait at least one scrape interval (default 15s)
   - Check target status page: http://localhost:9090/targets

2. **Metric Name Typo**
   ```bash
   # View available metrics
   curl http://localhost:9090/api/v1/label/__name__/values | grep node_
   ```

3. **Metrics Expired**
   - Check data retention: `--storage.tsdb.retention.time=15d`
   - Old metrics may have been deleted

---

## Grafana

### Grafana Not Accessible

**Symptoms**: Connection refused on port 3000

```bash
# Check if container is running
docker-compose ps grafana

# Check logs
docker-compose logs grafana
```

**Solutions**:

1. **Container Crashed**
   ```bash
   docker-compose up -d grafana
   ```

2. **Port Conflict**
   ```bash
   lsof -i :3000
   
   # Change port in docker-compose.yml
   ports:
     - "3001:3000"
   ```

### Prometheus Datasource Not Connecting

**Symptoms**: Red X or "unable to connect" in datasource settings

**Solutions**:

1. **Wrong URL**
   - Use `http://prometheus:9090` (internal Docker network)
   - NOT `http://localhost:9090` (external)
   - NOT `http://127.0.0.1:9090` (doesn't resolve in Docker)

2. **Test Connection**
   ```bash
   docker-compose exec grafana curl http://prometheus:9090/api/v1/targets
   ```

3. **Network Issues**
   ```bash
   # Ensure containers are on same network
   docker network inspect full-stack-monitoring_monitoring
   ```

### Dashboards Not Loading

**Symptoms**: Blank or error panels in dashboards

**Solutions**:

1. **Verify Datasource**
   - Click panel → Edit → Data source
   - Ensure Prometheus is selected
   - Test connection

2. **Check Metric Names**
   ```bash
   # Verify metrics exist in Prometheus
   curl "http://localhost:9090/api/v1/query?query=up"
   ```

3. **Review Panel Query**
   - Edit panel and check PromQL syntax
   - Run query in Prometheus directly

### Failed to Load Plugin

**Symptoms**: Error about missing plugin like "grafana-piechart-panel"

**Solutions**:

```bash
# Either remove unused plugin from compose
# OR install it
docker-compose exec grafana grafana-cli plugins install grafana-piechart-panel

# Restart Grafana
docker-compose restart grafana
```

---

## AlertManager

### AlertManager Not Sending Alerts

**Symptoms**: Alerts fire in Prometheus but don't reach notification channels

```bash
# Check AlertManager status
curl http://localhost:9093/api/v1/status
```

**Solutions**:

1. **Check AlertManager Logs**
   ```bash
   docker-compose logs alertmanager
   ```

2. **Verify Configuration**
   ```bash
   # Test YAML syntax
   docker run --rm -v $(pwd):/etc/alertmanager prom/alertmanager:latest \
     amtool config routes
   ```

3. **Check Alert Matchers**
   ```yaml
   # Ensure alert labels match route matchers
   route:
     match:
       severity: critical  # Alert must have this label
   ```

4. **Email Configuration Issues**
   ```yaml
   receivers:
     - name: 'email'
       email_configs:
         - to: 'your-email@example.com'
           from: 'alertmanager@example.com'
           smarthost: 'smtp.gmail.com:587'
           auth_username: 'your-email@gmail.com'
           auth_password: 'app-password'  # Not regular password
           tls_config:
             insecure_skip_verify: false
   ```

### Alerts Duplicate or Missing

**Symptoms**: Same alert fires multiple times or some don't appear

**Solutions**:

1. **Group Wait Too Short**
   ```yaml
   route:
     group_wait: 10s  # Wait for other alerts before grouping
   ```

2. **Check Alert Rules**
   ```bash
   # Verify rules loaded
   curl http://localhost:9090/api/v1/rules
   ```

---

## Exporters

### Exporter Returns No Metrics

**Symptoms**: Exporter port responds but no metrics

```bash
# Test exporter directly
curl http://exporter-host:9100/metrics | wc -l
```

**Solutions**:

1. **Check if Metrics Endpoint is Correct**
   ```bash
   # Some exporters use different paths
   curl http://exporter-host:9100/metrics        # Standard
   curl http://exporter-host:15692/metrics       # RabbitMQ path
   curl http://exporter-host:9115/metrics        # Blackbox
   ```

2. **Insufficient Permissions**
   ```bash
   # Node Exporter needs to read /proc, /sys
   docker run prom/node-exporter:latest \
     --path.procfs=/host/proc \
     --path.sysfs=/host/sys
   ```

3. **Service Not Ready**
   - Wait a few seconds after starting
   - Check logs: `docker-compose logs exporter_name`

### PostgreSQL Exporter Connection Failed

**Symptoms**: "failed to connect to database"

```bash
docker-compose logs postgres_exporter
```

**Solutions**:

1. **Test Database Connection**
   ```bash
   # Inside postgres_exporter container
   psql -U postgres -h postgres -c "SELECT 1"
   ```

2. **Wrong Connection String**
   ```yaml
   # Verify format
   DATA_SOURCE_NAME: "postgresql://user:password@host:5432/dbname?sslmode=disable"
   ```

3. **PostgreSQL Not Running**
   ```bash
   docker-compose logs postgres
   ```

### MongoDB Exporter Connection Issues

**Symptoms**: "connection refused" or "authentication failed"

**Solutions**:

```bash
# Test connection
docker-compose exec mongodb_exporter \
  curl "mongodb://user:password@mongodb:27017"

# Check MongoDB is running
docker-compose exec mongodb mongo --eval "db.adminCommand('ping')"

# Verify credentials in environment
docker-compose exec mongodb_exporter env | grep MONGODB
```

---

## Docker Issues

### Containers Won't Start Together

**Symptoms**: Some containers fail due to dependencies

**Solution**: Use health checks and depends_on:

```yaml
depends_on:
  postgres:
    condition: service_healthy  # Wait until healthy
  prometheus:
    condition: service_started    # Just wait until started
```

### Volume Permission Errors

**Symptoms**: "Permission denied" when writing to volumes

```bash
# Fix ownership
sudo chown -R 1000:1000 prometheus_data/
docker-compose restart prometheus
```

### Out of Disk Space

**Symptoms**: Containers fail, error messages about disk

```bash
# Check disk usage
df -h
docker system df

# Clean up old images and volumes
docker system prune -a
```

---

## Network Issues

### Can't Reach Exporter from Prometheus

**Symptoms**: Target shows DOWN in Prometheus

```bash
# Test from Prometheus container
docker-compose exec prometheus curl http://node_exporter:9100/metrics

# Check if containers are on same network
docker network inspect full-stack-monitoring_monitoring

# Check firewall
sudo iptables -L | grep 9100
```

### DNS Resolution Fails

**Symptoms**: Unknown host errors

**Solution**: Use container names (automatic DNS in Docker Compose)

```yaml
# Correct
targets: ['postgres_exporter:9187']

# Incorrect
targets: ['postgres_exporter.local:9187']
targets: ['localhost:9187']  # Only works for host container
```

---

## Performance Issues

### Prometheus Slow Queries

**Symptoms**: Dashboard panels take >5 seconds to load

```bash
# Check query performance
curl "http://localhost:9090/api/v1/query?query=up" \
  -w "Time: %{time_total}\n"
```

**Solutions**:

1. **Use Recording Rules**
   ```yaml
   groups:
     - name: high-load-queries
       rules:
         - record: instance:cpu_usage:5m
           expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total[5m])) * 100)
   ```

2. **Add Query Timeouts**
   ```yaml
   global:
     query_timeout: 30s
   ```

### Grafana Slow Rendering

**Symptoms**: Dashboards take long to load

**Solutions**:

1. **Reduce Panel Count**
   - Move panels to multiple dashboards
   - Use variables to filter data

2. **Increase Grafana Memory**
   ```yaml
   memswap_limit: 2g
   mem_limit: 1g
   ```

---

## General Debugging

### Enable Debug Logging

**Prometheus**:
```yaml
command:
  - '--log.level=debug'
```

**Grafana**:
```yaml
environment:
  GF_LOG_LEVEL: debug
```

**AlertManager**:
```yaml
command:
  - '--log.level=debug'
```

### Common Log Messages

```bash
# Check all logs
docker-compose logs -f

# Specific service
docker-compose logs -f prometheus
docker-compose logs -f grafana
docker-compose logs -f alertmanager
```

### Health Check

```bash
# Comprehensive health check
docker-compose exec prometheus curl http://localhost:9090/-/healthy
docker-compose exec grafana curl http://localhost:3000/api/health
docker-compose exec alertmanager curl http://localhost:9093/-/healthy
```

---

## Getting Help

1. **Check logs first**: `docker-compose logs service_name`
2. **Search documentation**: Check exporter-specific guides
3. **Validate configurations**: Use built-in validation tools
4. **Test connectivity**: Use curl/nc to verify service access
5. **Review examples**: Check provided configurations

---

## Emergency Procedures

### Complete Reset

```bash
# Stop everything
docker-compose down

# Remove all volumes
docker-compose down -v

# Clean Docker system
docker system prune -a

# Start fresh
docker-compose up -d
```

### Restore from Backup

```bash
# Restore Prometheus data
cat prometheus_backup.tar.gz | docker-compose exec -T prometheus tar xzf -

# Restart Prometheus
docker-compose restart prometheus
```

### Temporary Disable Components

Comment out in `docker-compose.yml` while troubleshooting:

```yaml
# mongodb:
# mongodb_exporter:
```

Then: `docker-compose up -d`

---

## Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Docker Troubleshooting](https://docs.docker.com/config/containers/logging/)
