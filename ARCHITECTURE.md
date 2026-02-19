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

```mermaid
graph TD
    TARGETS["MONITORING TARGETS<br/>Servers, Databases<br/>Services, Applications"]
    
    EXP["Exporters<br/>(9100-9400)<br/>Node, Custom, etc"]
    BX["Blackbox<br/>(9115)<br/>Synthetic Tests"]
    PG["Pushgateway<br/>(9091)<br/>Batch Jobs"]
    
    PROM["Prometheus (9090)<br/>- Scrape targets<br/>- Store metrics<br/>- Evaluate rules<br/>- Trigger alerts"]
    
    GRAF["Grafana<br/>(3000)<br/>Dashboards"]
    RSTOR1["Remote<br/>Storage"]
    RSTOR2["Remote<br/>Storage"]
    
    NOTIF["Dashboards, Alerts<br/>Notifications<br/>Email, Webhook, Slack,<br/>Teams, PagerDuty"]
    
    TARGETS --> EXP
    TARGETS --> BX
    TARGETS --> PG
    
    EXP --> PROM
    BX --> PROM
    PG --> PROM
    
    PROM --> GRAF
    PROM --> RSTOR1
    PROM --> RSTOR2
    
    GRAF --> NOTIF
    
    style TARGETS fill:#4285F4,color:#fff
    style EXP fill:#34A853,color:#fff
    style BX fill:#FF9800,color:#fff
    style PG fill:#2196F3,color:#fff
    style PROM fill:#607D8B,color:#fff
    style GRAF fill:#00BCD4,color:#fff
    style RSTOR1 fill:#9C27B0,color:#fff
    style RSTOR2 fill:#9C27B0,color:#fff
    style NOTIF fill:#D32F2F,color:#fff
```


    RSTOR["Remote Storage"]
    NOTIF["Dashboards, Alerts &<br/>Notifications<br/>(Email, Webhook,<br/>Slack, Teams, PagerDuty)"]
    
    EXP --> PROM
    BX --> PROM
    PG --> PROM
    PROM --> GRAF
    PROM --> RSTOR
    GRAF --> NOTIF
    
    style PROM fill:#f9f,stroke:#333,stroke-width:2px
    style GRAF fill:#bbf,stroke:#333,stroke-width:2px
```

## Monitoring Layers

### Layer 1: Infrastructure Monitoring

```mermaid
graph TD
    subgraph INF ["Infrastructure Layer"]
        NE["Node Exporter<br/>(Linux/Unix)"]
        WE["Windows Exporter<br/>(Windows)"]
        BX["Blackbox<br/>(HTTP/DNS/TCP)"]
        ND["Netdata<br/>(Real-time)"]
    end
    METRICS["System Metrics:<br/>CPU, Memory, Disk, Network<br/>Processes, File descriptors<br/>Load average, Uptime"]
    
    NE --> METRICS
    WE --> METRICS
    BX --> METRICS
    ND --> METRICS
    
    style INF fill:#e1f5ff
    style METRICS fill:#fff3e0
```

### Layer 2: Data & Storage Layer

```mermaid
graph TD
    subgraph DATA ["Data Layer"]
        PE["PostgreSQL Exporter<br/>(RDBMS)"]
        ME["MongoDB Exporter<br/>(Document DB)"]
        CE["ClickHouse Exporter<br/>(Analytics)"]
    end
    METRICS2["Database Metrics:<br/>Connections, Query performance<br/>Replication lag, Cache hit ratio<br/>Storage usage, Slow queries"]
    
    PE --> METRICS2
    ME --> METRICS2
    CE --> METRICS2
    
    style DATA fill:#f3e5f5
    style METRICS2 fill:#fff3e0
```

### Layer 3: Message & Event Layer

```mermaid
graph TD
    subgraph MSG ["Message Layer"]
        RE["RabbitMQ Exporter<br/>(Message Queue)"]
        KE["Kafka Exporter<br/>(Event Streaming)"]
        TE["Telegraf<br/>(Custom)"]
    end
    METRICS3["Application Metrics:<br/>Queue depth, Throughput<br/>Consumer lag, Broker health<br/>Custom application metrics"]
    
    RE --> METRICS3
    KE --> METRICS3
    TE --> METRICS3
    
    style MSG fill:#e8f5e9
    style METRICS3 fill:#fff3e0
```

### Layer 4: Network & Services Layer

```mermaid
graph TD
    subgraph NET ["Network Layer"]
        HE["HAProxy Exporter<br/>(Load Balancer)"]
        KV["Keepalived<br/>(High Availability)"]
        FG["Fortigate<br/>(Firewall)"]
        DP["DNS Prober<br/>(DNS Resolution)"]
    end
    METRICS4["Service Metrics:<br/>Load balancer stats<br/>Network throughput<br/>Availability, Latency"]
    
    HE --> METRICS4
    KV --> METRICS4
    FG --> METRICS4
    DP --> METRICS4
    
    style NET fill:#fce4ec
    style METRICS4 fill:#fff3e0
```

### Layer 5: Specialized Monitoring

```mermaid
graph TD
    subgraph SPEC ["Specialized Layer"]
        VM["VMware Exporter<br/>(Virtualization)"]
        TN["TrueNAS Exporter<br/>(Storage)"]
        IL["iLO Exporter<br/>(Hardware Health)"]
        CR["Chrony<br/>(NTP Sync)"]
        JX["JMX Exporter<br/>(Java Apps)"]
    end
    METRICS5["Domain-Specific Metrics:<br/>VM/host metrics<br/>Storage pool health<br/>Server BMC metrics<br/>Application JVM metrics"]
    
    VM --> METRICS5
    TN --> METRICS5
    IL --> METRICS5
    CR --> METRICS5
    JX --> METRICS5
    
    style SPEC fill:#fff3e0
    style METRICS5 fill:#fff3e0
```

## Metric Collection Strategy

### Pull Model (Prometheus Native)

```mermaid
graph LR
    PROM["Prometheus"] -->|15-120s interval| EXP["Exporter"]
    EXP -->|scrape| TARGET["Target"]
```

### Push Model (Pushgateway)

```mermaid
graph LR
    APP["Application<br/>(Batch Jobs)"] -->|push| PG["Pushgateway"]
    PG -->|scrape| PROM2["Prometheus"]
```

### Federation

```mermaid
graph LR
    PROMLOCAL["Prometheus<br/>(Local)"] -->|federate| PROMREMOTE["Prometheus<br/>(Remote)"]
    PROMREMOTE -.->|multi-datacenter| PROMLOCAL
```

## High Availability Architecture

```mermaid
graph TD
    P1["Prometheus 1<br/>(Primary)"]
    P2["Prometheus 2<br/>(Secondary)"]
    LB["Load Balancer"]
    GRAF["Grafana"]
    RS["Remote Storage<br/>(S3)"]
    
    P1 -->|replicate| LB
    P2 -->|replicate| LB
    LB --> GRAF
    P1 --> RS
    P2 --> RS
    
    style P1 fill:#e3f2fd
    style P2 fill:#e3f2fd
    style LB fill:#fff3e0
    style GRAF fill:#bbf
    style RS fill:#c8e6c9
```

## Alert Flow
```

```mermaid
sequenceDiagram
    participant Prometheus
    participant Grafana
    participant GroupingEngine as Grouping Engine
    participant Router
    participant ContactPoint as Contact Points
    
    Prometheus->>Grafana: 1. Rule Evaluation
    Grafana->>Grafana: 2. Generate Alerts
    Grafana->>GroupingEngine: 3. Deduplicate/Group
    GroupingEngine->>Router: 4. Route Matching
    Router->>ContactPoint: 5. Send Notifications
    ContactPoint->>ContactPoint: 6. Silencing
```

## Metric Retention Policies

```mermaid
graph TD
    RETENTION["Retention Strategy"]
    
    RT["Real-time<br/>(Prometheus)<br/>15 days<br/>Resolution: 15-30s<br/>Use: Dashboards, Alerts"]
    RR["Recording Rules<br/>(Pre-aggregated)<br/>5-min intervals<br/>Resolution: 5 min<br/>Use: Historical, Dashboards"]
    RS["Remote Storage<br/>(Long-term)<br/>1+ years<br/>Resolution: 1-hour avg<br/>Use: Trends, Compliance"]
    
    RETENTION --> RT
    RETENTION --> RR
    RETENTION --> RS
    
    style RT fill:#e1f5ff
    style RR fill:#f3e5f5
    style RS fill:#c8e6c9
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

```mermaid
graph TD
    INT["Internet<br/>(Blocked by Firewall)"]
    LB["LoadBalancer/Nginx<br/>(TLS Termination)"]
    INTERNAL["Internal Network"]
    PROM["Prometheus<br/>(No Auth)"]
    GRAF["Grafana<br/>(HTTP Auth +<br/>LDAP/OIDC)"]
    
    INT -->|blocked| LB
    LB --> INTERNAL
    INTERNAL --> PROM
    INTERNAL --> GRAF
    
    style INT fill:#ffcdd2
    style LB fill:#fff3e0
    style INTERNAL fill:#c8e6c9
    style PROM fill:#e3f2fd
    style GRAF fill:#e3f2fd
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

```mermaid
graph TD
    HEALTH["Monitoring Stack Health"]
    
    GH["Grafana Health<br/>✓ Alert evaluation<br/>✓ Notification delivery<br/>✓ Dashboard render time<br/>✓ Datasource connectivity<br/>✓ User sessions"]
    PH["Prometheus Health<br/>✓ Scrape success rate<br/>✓ Ingestion rate<br/>✓ Storage utilization<br/>✓ Query latency"]
    
    HEALTH --> GH
    HEALTH --> PH
    
    style GH fill:#bbf
    style PH fill:#f9f
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
