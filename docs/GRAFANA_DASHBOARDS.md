# Grafana Dashboards Guide

This guide covers all available Grafana dashboards and how to use them for monitoring your infrastructure.

## 🎯 Dashboard Overview

The Grafana provisioning configuration automatically loads 4 comprehensive dashboards covering all major monitoring areas.

### 1. **Infrastructure Overview** (`infrastructure-overview`)
Monitors core system metrics using Node Exporter data.

**Key Metrics:**
- CPU Usage (%) - Real-time processor utilization
- Memory Usage (%) - RAM consumption with thresholds
- Network Traffic - Inbound/outbound bytes per interface
- Disk Available (%) - Filesystem utilization by device

**Use Cases:**
- Base infrastructure health check
- Capacity planning and trend analysis
- Identifying resource bottlenecks
- SLA tracking and alerting

**Thresholds:**
- CPU: Alert at 80%+
- Memory: Warn at 70%, critical at 85%
- Disk: Alert when available drops below 15%

---

### 2. **Endpoint Monitoring** (`endpoint-monitoring`)
Tracks external endpoint availability and performance using Blackbox Exporter.

**Key Metrics:**
- Endpoint Availability - Up/Down status for all monitored URLs
- Response Time - Latency trends with performance thresholds
- SSL Certificate Expiry - Days remaining before expiration

**Monitored Services:**
- HTTP/HTTPS endpoints (web services, APIs)
- DNS resolution quality
- TCP connectivity to critical services
- SSL/TLS certificate status

**Use Cases:**
- External SLA monitoring
- SSL certificate lifecycle management
- Multi-region endpoint health checks
- Synthetic monitoring and alerting

**Thresholds:**
- Response Time: Warn at 1s, critical at 2s
- SSL Expiry: Alert at 7 days remaining

---

### 3. **Database - PostgreSQL** (`db-postgres`)
Comprehensive PostgreSQL monitoring with replication and performance tracking.

**Key Metrics:**
- Active Connections - Current connected clients
- Transactions/sec - Write throughput and query rate
- Replication Lag - Standby replication delay in bytes
- Cache Hit Ratio - Query cache effectiveness
- Dead Rows by Table - VACUUM pressure indicator

**Monitored Components:**
- Connection pools and state
- Query performance and TPS
- Replication status and lag
- Cache efficiency
- Index and table bloat

**Use Cases:**
- Production database health (primary focus)
- Replication monitoring and failover detection
- Performance tuning (cache hit ratio optimization)
- Maintenance scheduling (VACUUM, autovacuum tuning)
- Capacity planning

**Critical Thresholds:**
- Cache Hit Ratio should be > 95% for production
- Replication Lag should be < 100MB
- Active connections trending indicates scaling needs

---

### 4. **Message Queues** (`msg-queues`)
Monitors RabbitMQ and Kafka message infrastructure.

**Key Metrics:**
- RabbitMQ Queue Depth - Messages pending processing
- Message Throughput - ACK and delivery rates
- Kafka Consumer Lag - Processing latency by consumer group
- Kafka Broker Status - Health of individual broker nodes

**Monitored Components:**
- Queue backlog and stalling
- Message ingestion vs delivery rates
- Consumer group processing lag
- Broker availability and synchronization

**Use Cases:**
- Message broker health and capacity
- Consumer/producer lag detection
- Queue backlog investigation
- Event stream processing monitoring
- Scaling decisions (consumer group sizing)

**Thresholds:**
- Queue Depth: Warn at 5k messages, critical at 10k
- Consumer Lag: Warn at 100k offsets, critical at 500k

---

## 🚀 Accessing Dashboards

### URL Format
```
http://localhost:3000/d/{uid}/{slug}
```

### Direct Links
- **Infrastructure**: http://localhost:3000/d/infra-overview/infrastructure-overview
- **Endpoints**: http://localhost:3000/d/endpoint-monitoring/endpoint-monitoring
- **PostgreSQL**: http://localhost:3000/d/db-postgres/database-postgresql
- **Message Queues**: http://localhost:3000/d/msg-queues/message-queues

### Dashboard Home
All dashboards are available from the Home page after logging in:
- Default Username: `admin`
- Default Password: `admin`

---

## 📊 Dashboard Features

### Time Range Selection
Each dashboard supports multiple time ranges:
- Last 1 hour
- Last 6 hours (default for Infrastructure & PostgreSQL)
- Last 24 hours (default for Endpoints)
- Last 7 days
- Last 30 days
- Custom range

### Legend Options
- **Show/Hide Metrics**: Click legend items to toggle visibility
- **Calculations**: Display mean, max, min values
- **Export Data**: Right-click graphs for options

### Panel Features
- **Zoom**: Click and drag on graphs to zoom into a time range
- **Inspect**: View raw data behind visualizations
- **Annotations**: System events appear as vertical lines

---

## 🔧 Configuring Dashboards

### Provisioning Configuration
Dashboards are auto-loaded from files in `grafana/provisioning/`:

**Datasources** (`datasources/prometheus.yml`):
- Configures Prometheus connection
- Update URL if Prometheus is on different host

**Dashboards** (`dashboards/dashboards.yml`):
- Specifies dashboard file locations
- Set `disableDeletion: false` to allow dashboard editing without git conflicts

### Adding New Dashboards

1. Create dashboard JSON in `grafana/dashboards/`
2. Reference UID and slug in dashboard JSON
3. Restart Grafana or wait 10 seconds for auto-reload:
   ```bash
   docker restart grafana
   ```

### Dashboard Best Practices
- Use consistent color schemes for metric types (green=good, yellow=warning, red=critical)
- Include descriptive titles and legends
- Set appropriate time ranges for data analysis
- Use stat panels for KPIs and timeseries for trends
- Document custom metrics in panel descriptions

---

## 📈 Example Queries

### Node Exporter
```promql
# CPU Usage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory Used
100 * (1 - ((node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)))
```

### PostgreSQL
```promql
# Active Connections
pg_stat_activity_count{state="active"}

# Cache Hit Ratio
pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)
```

### RabbitMQ
```promql
# Queue Depth
rabbitmq_queue_messages_ready

# Message Rate
rate(rabbitmq_queue_messages_acked_total[5m])
```

---

## 🎨 Customization

### Changing Thresholds
Edit JSON files in `grafana/dashboards/` and modify the `thresholds` section in panel configs.

### Adding New Panels
Use Grafana UI to create panels, then export as JSON:
1. Edit dashboard in Grafana
2. Save changes
3. Use "Share Dashboard" → "Export" → "Save external" to get updated JSON
4. Replace JSON files and restart

### Dashboard Variables
Add variables for dynamic filtering:
```json
"templating": {
  "list": [
    {
      "name": "instance",
      "type": "query",
      "datasource": "Prometheus",
      "query": "label_values(node_up, instance)"
    }
  ]
}
```

---

## 🔍 Troubleshooting

### Dashboards Not Appearing
1. Check Grafana logs: `docker logs grafana`
2. Verify datasource connection in Grafana UI (Settings → Data Sources)
3. Ensure `grafana/provisioning/` has correct permissions
4. Restart container: `docker-compose restart grafana`

### Metrics Not Showing
1. Verify Prometheus scrape config includes the job
2. Check Prometheus targets: http://localhost:9090/targets
3. Confirm exporters are running: `docker-compose ps`
4. Check metric availability: http://localhost:9090/graph

### Query Errors
1. Validate PromQL syntax in Prometheus UI
2. Check metric names with `autocomplete` in Prometheus
3. Verify label filters match available data

---

## � Setting Up Alerts

This project uses **Grafana's native alerting system** for alert management and notifications.

### Quick Alert Setup

1. Navigate to **Alerting** → **Alert rules**
2. Click **+ New alert rule**
3. Configure:
   - Query: Define condition (e.g., CPU > 80%)
   - Condition: Set threshold
   - Contact point: Choose notification channel
   - Name: Descriptive alert name
4. Save alert rule

### Contact Points (Notifications)

Configure notification channels in **Alerting** → **Contact points**:

- **Email**: SMTP-based email notifications
- **Webhook**: JSON POST to custom endpoint
- **Slack**: Direct Slack channel integration
- **Teams**: Microsoft Teams webhooks
- **PagerDuty**: On-call incident routing

### Example Alert Rules

**CPU Alert**:
- Condition: CPU > 80%
- For: 5 minutes
- Notification: Email

**SSL Certificate Expiry**:
- Condition: Days remaining < 7
- For: Immediately
- Notification: Critical alerts Slack

**Service Unavailable**:
- Condition: probe_success < 1
- For: 2 minutes
- Notification: High priority email

**[📖 Complete Alerting Guide](GRAFANA_ALERTING.md)** - Detailed setup, examples, and best practices.

---

## 📚 Related Documentation
- [GRAFANA_ALERTING.md](GRAFANA_ALERTING.md) - Complete alerting setup guide
- [MONITORING_MATRIX.md](../docs/MONITORING_MATRIX.md) - Detailed exporter coverage
- [metric_queries.md](../examples/metric_queries.md) - 50+ PromQL examples
- [TROUBLESHOOTING.md](../docs/TROUBLESHOOTING.md) - Common issues and solutions
