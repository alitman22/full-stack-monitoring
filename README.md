# Full-Stack Monitoring with Prometheus & Grafana

A comprehensive, production-ready monitoring stack showcasing expertise in observability, infrastructure monitoring, and data visualization using industry-leading tools.

## 🎯 Project Overview

This project demonstrates a **multi-layer, enterprise-grade monitoring solution** that integrates:

- **Prometheus** - Metrics collection and storage
- **Grafana** - Visualization and dashboarding
- **20+ Specialized Exporters** - Deep infrastructure and application monitoring
- **Alerting** - AlertManager with custom alert rules
- **High Availability** - Redundant monitoring infrastructure

## 📊 Monitoring Coverage

### Infrastructure Monitoring
- **Node Exporter** - Linux/Unix system metrics (CPU, memory, disk, network)
- **Windows Exporter** - Windows system metrics
- **Blackbox Exporter** - HTTP/HTTPS endpoint monitoring and SSL certificate tracking
- **Ping/ICMP Monitoring** - Network connectivity and latency

### Database Monitoring
- **PostgreSQL Exporter** - Database statistics, connections, replication lag
- **MongoDB Exporter** - Database performance and replica set status
- **ClickHouse Exporter** - Analytics database metrics

### Message Queue & Cache
- **RabbitMQ Exporter** - Queue depth, message rates, connection metrics
- **Kafka Exporter** - Topic metrics, consumer lag, broker health

### Application Monitoring
- **JMX Exporter** - Java application metrics (Kafka, etc.)
- **Process Exporter** - Individual process-level metrics
- **Telegraf** - Flexible metrics collection for custom applications

### Infrastructure Services
- **eTCD Exporter** - Distributed consensus monitoring
- **HAProxy Exporter** - Load balancer statistics
- **Keepalived Exporter** - High availability monitoring
- **Chrony Exporter** - NTP synchronization and time metrics

### Specialized Monitoring
- **Netdata** - Real-time system monitoring
- **VMware Exporter** - vSphere and VM monitoring
- **TrueNAS Exporter** - Storage system metrics
- **Fortigate Exporter** - Firewall and SNMP metrics
- **HP iLO Exporter** - Physical server health and IPMI
- **MTR Exporter** - Network route analysis
- **DNS Prober** - DNS resolution monitoring
- **vSAN** - VMware vSAN monitoring

### Push-Based Monitoring
- **Pushgateway** - Support for batch jobs and ephemeral containers

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Monitoring Targets                        │
│  ┌────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │Nodes   │ │Databases │ │Services  │ │Endpoints │         │
│  └────────┘ └──────────┘ └──────────┘ └──────────┘         │
└──────────────────────────┬──────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
      ┌───▼────┐   ┌──────▼─────┐   ┌─────▼────┐
      │Exporters│   │Blackbox    │   │Pushgw    │
      └────┬────┘   └──────┬─────┘   └─────┬────┘
           │               │              │
           └───────────────┼──────────────┘
                           │
                    ┌──────▼──────┐
                    │ Prometheus  │
                    │  (Storage)  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
           ┌──▼──┐    ┌────▼───┐  ┌───▼────┐
           │ Grafana │  │Alerting  │  │Rules    │
           │Dashboards │  │       │  │Engine   │
           └────────┘   └────────┘   └────────┘
```

## 📁 Project Structure

```
.
├── README.md                          # This file
├── ARCHITECTURE.md                    # Detailed architecture documentation
├── prometheus/
│   ├── prometheus.yml                # Main Prometheus configuration
│   ├── alerts.yml                    # Alert rules
│   └── recording_rules.yml           # Recording rules for optimization
├── exporters/
│   ├── node_exporter.md              # Linux/Unix monitoring setup
│   ├── postgres_exporter.md          # PostgreSQL monitoring
│   ├── mongodb_exporter.md           # MongoDB monitoring
│   ├── rabbitmq_exporter.md          # RabbitMQ monitoring
│   ├── kafka_exporter.md             # Kafka monitoring
│   ├── blackbox_exporter.md          # HTTP/DNS endpoint monitoring
│   ├── custom_exporters.md           # Specialized exporters guide
│   └── exporter_matrix.md            # Comparison of all exporters
├── grafana/
│   ├── dashboards/
│   │   ├── infrastructure-overview.json    # CPU, memory, disk, network
│   │   ├── endpoint-monitoring.json       # HTTP endpoints, SSL certificates
│   │   ├── database-monitoring.json       # PostgreSQL connection, queries, replication
│   │   └── message-queues.json            # RabbitMQ & Kafka metrics
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── prometheus.yml            # Prometheus datasource auto-config
│   │   └── dashboards/
│   │       └── dashboards.yml            # Dashboard provisioning config
│   └── docs/GRAFANA_DASHBOARDS.md       # Dashboard guide (this file)
├── docker-compose.yml                # Full stack orchestration
├── docker-compose.prod.yml           # Production HA setup
├── alertmanager.yml                  # AlertManager configuration
├── deployment/
│   ├── DOCKER_SETUP.md              # Docker deployment guide
│   ├── KUBERNETES_SETUP.md          # Kubernetes deployment
│   ├── MANUAL_SETUP.md              # Manual installation
│   └── HIGH_AVAILABILITY.md         # HA setup guide
├── scripts/
│   ├── generate_dashboards.sh        # Generate dashboard JSON
│   ├── backup_configuration.sh       # Configuration backup
│   └── health_check.sh               # Health check script
└── examples/
    ├── custom_exporter_python.md    # Example custom exporter
    ├── metric_queries.md             # Common PromQL queries
    └── alerting_rules_examples.md   # Alert rule examples
```

## 🚀 Quick Start

### Prerequisites
- Docker & Docker Compose (recommended)
- Or: Prometheus, Grafana, exporters installed manually
- Linux/Unix system (most exporters support Linux)

### 1. Clone and Navigate
```bash
git clone https://github.com/alitman22/full-stack-monitoring.git
cd full-stack-monitoring
```

### 2. Start the Stack (Docker)
```bash
docker-compose up -d
```

### 3. Access Services

| Service | URL | Default Credentials |
|---------|-----|-------------------|
| Prometheus | http://localhost:9090 | - |
| Grafana | http://localhost:3000 | admin / admin |
| AlertManager | http://localhost:9093 | - |
| Node Exporter | http://localhost:9100 | - |

### 4. Configure Targets

Update `prometheus/prometheus.yml` with your actual target IPs/hostnames:

```yaml
scrape_configs:
  - job_name: 'your_node'
    static_configs:
      - targets: ['your-node-ip:9100']
```

### 5. Access Grafana Dashboards

Dashboards are automatically provisioned on startup. Log in to Grafana:
- URL: http://localhost:3000
- Username: admin
- Password: admin

Available dashboards:
1. **Infrastructure Overview** - CPU, memory, disk, network metrics
2. **Endpoint Monitoring** - External service availability and SSL expiry
3. **Database - PostgreSQL** - Connections, replication, cache hit ratio
4. **Message Queues** - RabbitMQ and Kafka metrics

**[📖 Complete Dashboard Guide](docs/GRAFANA_DASHBOARDS.md)** - Detailed information about each dashboard, metrics, and customization options.

## 📚 Exporter Documentation

Each exporter has detailed setup guides in the `exporters/` directory:

- **[Node Exporter](exporters/node_exporter.md)** - System metrics
- **[PostgreSQL Exporter](exporters/postgres_exporter.md)** - Database monitoring
- **[MongoDB Exporter](exporters/mongodb_exporter.md)** - MongoDB metrics
- **[RabbitMQ Exporter](exporters/rabbitmq_exporter.md)** - Message queue monitoring
- **[Kafka Exporter](exporters/kafka_exporter.md)** - Kafka broker monitoring
- **[Blackbox Exporter](exporters/blackbox_exporter.md)** - Endpoint monitoring
- **[Custom Exporters](exporters/custom_exporters.md)** - VMware, DNS, etc.

## 📊 Key Metrics Monitored

### System Metrics
- CPU usage, load average, context switches
- Memory: free, used, cached, available
- Disk: I/O operations, latency, utilization
- Network: throughput, packets, errors

### Database Metrics
- Connection count and state
- Query performance and cache hit ratio
- Replication lag and status
- Transaction rate and duration

### Application Metrics
- Request rate, latency, error rate
- Queue depth and processing time
- Worker/thread utilization
- Memory and resource consumption

### Infrastructure Metrics
- Service availability and uptime
- SSL certificate expiration
- DNS resolution time
- Network latency and packet loss

## 🔔 Alerting

Pre-configured alert rules for:
- High CPU/memory utilization
- Disk space running low
- Service down alerts
- Database replication lag
- Certificate expiration warnings
- Network connectivity issues

View and customize in `prometheus/alerts.yml`

## 🎯 Use Cases Demonstrated

1. **Microservices Monitoring** - Track multiple services and their dependencies
2. **Database Monitoring** - Multi-database environment (PostgreSQL, MongoDB, ClickHouse)
3. **Distributed Systems** - ETcd, Kafka, RabbitMQ monitoring
4. **Load Balancer Monitoring** - HAProxy and Keepalived metrics
5. **Endpoint Monitoring** - HTTP/HTTPS checks and SSL validation
6. **Infrastructure Monitoring** - Comprehensive system metrics
7. **High Availability** - Redundant monitoring infrastructure
8. **Custom Applications** - JMX, process-level, and push-based monitoring

## 💡 Best Practices Implemented

- ✅ **Metric Naming** - Follows Prometheus conventions
- ✅ **Recording Rules** - PreComputed high-level metrics
- ✅ **Service Discovery** - Dynamic target detection
- ✅ **Data Retention** - Optimal storage and retention policies
- ✅ **Alerting Strategy** - Multi-level alerts with escalation
- ✅ **Dashboard Design** - Organized, hierarchical dashboards
- ✅ **Security** - TLS support and credential management
- ✅ **Scalability** - Federation and remote storage ready

## 📖 Documentation Files

- [ARCHITECTURE.md](ARCHITECTURE.md) - Detailed system architecture
- [prometheus/alerts.yml](prometheus/alerts.yml) - Alert rules documentation
- [docs/MONITORING_MATRIX.md](docs/MONITORING_MATRIX.md) - Coverage matrix
- [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) - Common issues

## 🛠️ Advanced Topics

### High Availability
See [deployment/HIGH_AVAILABILITY.md](deployment/HIGH_AVAILABILITY.md) for:
- Multi-Prometheus setup with replication
- Remote storage configuration
- Load balancing Prometheus instances

### Kubernetes Deployment
See [deployment/KUBERNETES_SETUP.md](deployment/KUBERNETES_SETUP.md) for:
- ServiceMonitor CRDs
- PrometheusRule CRDs
- Helm chart examples

### Custom Exporters
See [examples/custom_exporter_python.md](examples/custom_exporter_python.md) for:
- Writing custom Python exporters
- Publishing metrics to Pushgateway

## 🔍 Monitoring Matrix

| Component | Exporter | Metrics | Documentation |
|-----------|----------|---------|---|
| Linux Nodes | Node Exporter | 200+ | [node_exporter.md](exporters/node_exporter.md) |
| PostgreSQL | postgres_exporter | 100+ | [postgres_exporter.md](exporters/postgres_exporter.md) |
| MongoDB | mongodb_exporter | 150+ | [mongodb_exporter.md](exporters/mongodb_exporter.md) |
| RabbitMQ | rabbitmq_exporter | 200+ | [rabbitmq_exporter.md](exporters/rabbitmq_exporter.md) |
| Kafka | kafka_exporter | 100+ | [kafka_exporter.md](exporters/kafka_exporter.md) |
| HTTP Endpoints | Blackbox | SSL/HTTP checks | [blackbox_exporter.md](exporters/blackbox_exporter.md) |
| Load Balancer | haproxy_exporter | 300+ | [custom_exporters.md](exporters/custom_exporters.md) |
| ClickHouse | clickhouse_exporter | 80+ | [custom_exporters.md](exporters/custom_exporters.md) |
| Java Apps | jmx_exporter | Custom | [custom_exporters.md](exporters/custom_exporters.md) |
| DNS | dns_prober | Latency/availability | [custom_exporters.md](exporters/custom_exporters.md) |
| VMware | vmware_exporter | VM/host metrics | [custom_exporters.md](exporters/custom_exporters.md) |
| Storage | truenas_exporter | Storage metrics | [custom_exporters.md](exporters/custom_exporters.md) |
| Firewall | fortigate_exporter | Traffic/rules | [custom_exporters.md](exporters/custom_exporters.md) |
| Hardware | hpilo_exporter | Hardware health | [custom_exporters.md](exporters/custom_exporters.md) |

## 🎓 Learning Resources

This project demonstrates:
- Prometheus architecture and operations
- PromQL query language and optimization
- Grafana dashboard design patterns
- Exporter development and integration
- Alert rule creation and management
- Service discovery patterns
- High availability monitoring infrastructure
- Infrastructure-as-Code practices

## 🤝 Contributing

Contributions welcome! Please:
1. Review existing documentation
2. Follow Prometheus naming conventions
3. Include dashboard templates for new exporters
4. Add to exporter matrix documentation

## 📞 Support & Questions

For issues or questions:
1. Check [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)
2. Review exporter-specific documentation
3. Check Prometheus/Grafana official docs

## 📜 License

MIT License - See LICENSE file

---

**Project Status**: Production-Ready ✅

*Showcasing expertise in observability engineering, infrastructure monitoring, and DevOps best practices.*
