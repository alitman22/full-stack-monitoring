# ARCHITECTURE.md

## Full-Stack Monitoring Architecture

Comprehensive reference architecture for enterprise-grade monitoring infrastructure using Prometheus, Grafana, and 20+ specialized exporters.

## System Components

### Core Stack

1. **Prometheus** (9090)
   - Time-series database for metrics collection
   - Multi-DC scraping capability
   - Built-in alerting rules engine
   - 15-day data retention (configurable)

2. **Grafana** (3000)
   - Metrics visualization and dashboarding
   - Multi-datasource support
   - Native alert rules and routing
   - Multi-channel notifications (Email, Webhook, Slack, Teams, etc.)
   - User authentication and RBAC

## Data Flow

```
┌─────────────────────────────────────────────────────────┐
│                   MONITORING TARGETS                     │
├────────────┬──────────────┬──────────────┬───────────────┤
│ Servers    │ Databases    │ Services     │ Applications  │
│ (Node)     │ (PostgreSQL) │ (RabbitMQ)   │ (JMX)         │
│ (Windows)  │ (MongoDB)    │ (Kafka)      │ (Custom)      │
│            │ (ClickHouse) │ (Redis)      │ (Telegraf)    │
└────────────┴──────────────┴──────────────┴───────────────┘
             │                    │               │
             ▼                    ▼               ▼
     ┌───────────────┐   ┌──────────────┐   ┌──────────┐
     │   Exporters   │   │  Blackbox    │   │ Pushgw   │
     │  9100-9400    │   │   9115       │   │ 9091     │
     └───────────────┘   └──────────────┘   └──────────┘
             │                    │               │
             └────────────────────┼───────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │      Prometheus (9090)     │
                    │                            │
                    │ - Scrape targets           │
                    │ - Store metrics            │
                    │ - Evaluate rules           │
                    │ - Trigger alerts           │
                    └─────────────┬──────────────┘
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
             ▼                    ▼                    ▼
      ┌────────────┐       ┌─────────────┐      ┌──────────┐
      │  Grafana   │       │Remote       │      │ Remote   │
      │  (3000)    │       │Storage      │      │ Storage  │
      └────────────┘       └─────────────┘      └──────────┘
             │
      ┌──────┴────────────────────────┐
      │   Dashboards, Alerts &         │
      │   Notifications (Email,        │
      │   Webhook, Slack, Teams,       │
      │   PagerDuty)                   │
      └────────────────────────────────┘
```

## Monitoring Layers

### Layer 1: Infrastructure Monitoring
```
┌─────────────────────────────────────┐
│    Infrastructure Layer              │
├─────────────────────────────────────┤
│ Node Exporter → Linux/Unix nodes     │
│ Windows Exporter → Windows servers   │
│ Blackbox → HTTP/DNS/TCP endpoints    │
│ Netdata → Real-time system stats     │
└─────────────────────────────────────┘
        ↓
   System Metrics:
   • CPU, Memory, Disk, Network
   • Processes, File descriptors
   • Load average, Uptime
```

### Layer 2: Data & Storage Layer
```
┌─────────────────────────────────────┐
│    Data Layer                        │
├─────────────────────────────────────┤
│ PostgreSQL Exporter → RDBMS         │
│ MongoDB Exporter → Document DB      │
│ ClickHouse Exporter → Analytics     │
└─────────────────────────────────────┘
        ↓
   Database Metrics:
   • Connections, Query performance
   • Replication lag, Cache hit ratio
   • Storage usage, Slow queries
```

### Layer 3: Message & Event Layer
```
┌─────────────────────────────────────┐
│    Message Layer                     │
├─────────────────────────────────────┤
│ RabbitMQ Exporter → Message queue   │
│ Kafka Exporter → Event streaming    │
│ Telegraf → Custom metrics           │
└─────────────────────────────────────┘
        ↓
   Application Metrics:
   • Queue depth, Throughput
   • Consumer lag, Broker health
   • Custom application metrics
```

### Layer 4: Network & Services Layer
```
┌─────────────────────────────────────┐
│    Network Layer                     │
├─────────────────────────────────────┤
│ HAProxy Exporter → Load balancer    │
│ Keepalived → High availability      │
│ Fortigate → Firewall metrics        │
│ DNS Prober → DNS resolution         │
└─────────────────────────────────────┘
        ↓
   Service Metrics:
   • Load balancer stats
   • Network throughput
   • Availability, Latency
```

### Layer 5: Specialized Monitoring
```
┌─────────────────────────────────────┐
│    Specialized Layer                 │
├─────────────────────────────────────┤
│ VMware Exporter → Virtualization    │
│ TrueNAS Exporter → Storage          │
│ iLO Exporter → Hardware health      │
│ Chrony → NTP synchronization        │
│ JMX Exporter → Java applications    │
└─────────────────────────────────────┘
        ↓
   Domain-Specific Metrics:
   • VM/host metrics
   • Storage pool health
   • Server BMC metrics
   • Application JVM metrics
```

## Metric Collection Strategy

### Pull Model (Prometheus Native)
```
Prometheus → Exporter → Target
    120s scrape interval
    15s default
    Automatic service discovery
```

### Push Model (Pushgateway)
```
Application → Pushgateway → Prometheus
    For batch jobs
    For short-lived processes
    For firewall-restricted targets
```

### Federation
```
Prometheus (Local) → Prometheus (Remote)
    Multi-datacenter support
    Hierarchical metrics collection
    Regional aggregation
```

## High Availability Architecture

```
┌──────────────────┐          ┌──────────────────┐
│   Prometheus 1   │          │   Prometheus 2   │
│    (Primary)     │          │   (Secondary)    │
└────────┬─────────┘          └────────┬─────────┘
         │                             │
         └─────────────┬───────────────┘
                       │
                    ┌──▼──┐
                    │ LB   │
                    └─┬────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
    ┌───▼───┐  ┌─────▼────┐  ┌────▼───┐
    │Grafana│  │AlertMgr 1│  │AlertMgr2│
    └───────┘  └──────────┘  └────────┘
        │             │              │
        └─────────────┴──────────────┘
                      │
         ┌────────────▼────────────┐
         │  Remote Storage (S3)    │
         │  Backup location        │
         │  Long-term retention    │
         └─────────────────────────┘
```

## Alert Flow

```
1. Rule Evaluation
   Prometheus evaluates alert rules every 30s

2. Alert Generation
   {alertname, instance, severity, ...} created
   Fired/Resolved states tracked

3. Alert Deduplication
   Grafana groups similar alerts
   group_by: [alertname, instance]
   group_wait: 10s (batch collection)

4. Route Matching
   Grafana routing rules applied
   Critical → Immediate dispatch
   Warning → Batch in 30s
   Info → Daily digest

5. Notification
   Severity->Contact point mapping
   Email, Slack, Teams, PagerDuty, Webhook, etc.
   Escalation policies applied

6. Silencing/Inhibition
   Built-in silencing feature
   Prevents alert storms
   Maintenance window support
```

## Metric Retention Policies

```
Real-time Metrics (Prometheus)
├─ 15 days (default)
├─ Resolution: 15-30 seconds
└─ Use case: Dashboards, immediate alerting

Recording Rules (Pre-aggregated)
├─ 5-minute intervals
├─ Resolution: 5 minutes
└─ Use case: Historical analysis, dashboards

Remote Storage (Long-term)
├─ 1+ years (configurable)
├─ Resolution: 1-hour average
└─ Use case: Trend analysis, compliance
```

## Scalability Considerations

### Horizontal Scaling

**Add Prometheus Instances**:
- Use metric relabeling to shard targets
- Load balance via Nginx/HAProxy
- Separate scrapers for different target groups

**Add Grafana Instances**:
- Use external database (MySQL/PostgreSQL)
- Multiple Grafana servers behind load balancer
- Session store in database
- Centralized alerting configuration

### Vertical Scaling

**Prometheus Memory**:
- ~1GB per million metrics/day
- Adjust based on scrape frequency
- Monitor via `prometheus_tsdb_symbol_table_size_bytes`

**Storage Requirements**:
- Adjust retention: `--storage.tsdb.retention.time`
- Use compression: `--storage.tsdb.max-block-chunks`

## Security Architecture

### Network Security
```
Internet (Blocked by Firewall)
    ↓
LoadBalancer/Nginx (TLS termination)
    ↓
Internal Network
    ├─ Prometheus (no auth)
    └─ Grafana (HTTP Auth + LDAP/OIDC)
```

### Data Security
- Encrypt secrets in transit (TLS)
- Store credentials in secure storage (Vault)
- Audit access logs
- Regular backups with encryption

### Access Control
- Prometheus: Enforce via reverse proxy
- Grafana: RBAC with organizations and teams
- AlertManager: Protected endpoint configuration

## Integration Points

### Observability Stack
- Prometheus → Jaeger/Zipkin (traces)
- Logs → ELK/Loki (centralized logging)
- Metrics → Prometheus (monitoring)
- Events → AlertManager (alerting)

### External Systems
- OIDC/OAuth for authentication
- LDAP for user directory
- Slack/Teams for notifications
- PagerDuty for on-call routing
- SMTP for email alerts

## Disaster Recovery

### Backup Strategy
```
Daily Backups
├─ Prometheus data (tar.gz)
├─ Grafana datasources, dashboards & alerts
├─ Exporter configurations
└─ Docker volumes
    ↓
    Stored in S3/Off-site
```

### Recovery Time Objectives
- RTO: < 1 hour (restore from backup)
- RPO: < 1 day (daily backups)
- Verify: Weekly backup restoration tests

## Operational Procedures

### Deployment
1. Update configurations (Git)
2. Validate syntax
3. Dry-run in staging
4. Rolling update to production
5. Health checks

### Maintenance
- Monthly: Review alert rules
- Quarterly: Update exporters
- Annually: Capacity planning review
- As-needed: Configuration updates

### Monitoring the Monitors
```
Grafana Health
├─ Alert rule evaluation
├─ Notification delivery
├─ Dashboard render time
├─ Datasource connectivity
└─ User sessions

Prometheus Health
├─ Scrape success rate
├─ Ingestion rate
├─ Storage utilization
└─ Query latency
```

## Best Practices Summary

1. **Metrics**: Use consistent naming, appropriate cardinality
2. **Retention**: Right-size data retention policy
3. **Performance**: Use recording rules, optimize queries
4. **Reliability**: Implement HA, regular backups
5. **Security**: TLS, RBAC, secret management
6. **Operations**: IaC, documentation, runbooks
7. **Alerting**: Meaningful thresholds, routing hierarchy
8. **Scalability**: Federation, storage optimization, sharding

---

**For detailed setup instructions, see deployment guides in `deployment/` directory**
