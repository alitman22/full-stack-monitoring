# Custom & Specialized Exporters Setup Guide

This guide covers the setup of specialized exporters for various infrastructure components not covered by standard exporters.

## MongoDB Exporter

Monitors MongoDB replica sets, sharded clusters, and standalone instances.

### Installation

```bash
docker run -d \
  --name=mongodb_exporter \
  -p 9216:9216 \
  percona/mongodb_exporter:latest \
  --mongodb.uri=mongodb://user:password@localhost:27017/admin \
  --collect-all
```

### Key Metrics
- Replica set status and oplog
- Database and collection stats
- Query operations and latency
- Memory and CPU usage
- Replication lag

### Prometheus Config
```yaml
- job_name: 'mongodb_exporter'
  static_configs:
    - targets: ['localhost:9216']
```

## RabbitMQ Exporter

Monitors message queue depth, throughput, and connection metrics.

### Installation

```bash
docker run -d \
  --name=rabbitmq_exporter \
  -p 15692:15692 \
  kbudde/rabbitmq-exporter:latest \
  RABBITMQ_URL=http://localhost:15672 \
  RABBITMQ_USER=guest \
  RABBITMQ_PASSWORD=guest
```

### Key Metrics
- Queue depth and message rates
- Connection count and state
- Channel count
- Consumer count
- Broker health

### Prometheus Config
```yaml
- job_name: 'rabbit_mq'
  metrics_path: /metrics/per-object
  static_configs:
    - targets: ['localhost:15692']
```

## Kafka Exporter

Monitors Kafka brokers, topics, and consumer groups.

### Installation

```bash
docker run -d \
  --name=kafka_exporter \
  -p 9308:9308 \
  danielqsj/kafka-exporter:latest \
  --kafka.servers=kafka:9092 \
  --web.listen-address=:9308
```

### Key Metrics
- Topic partition metrics
- Broker metrics
- Consumer group lag
- Replica status
- Leader elections

### Prometheus Config
```yaml
- job_name: 'kafka_exporter'
  metrics_path: /metrics
  static_configs:
    - targets: ['localhost:9308']
```

## JMX Exporter

Collects metrics from Java applications via JMX.

### Installation

Download and configure:
```bash
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_exporter/0.19.0/jmx_exporter-0.19.0-jar-with-dependencies.jar
```

### JMX Config Example

```yaml
---
lowercaseOutputName: true
lowercaseOutputLabelNames: true
rules:
  # Generic heap memory
  - pattern: "java.lang<type=Memory>(.+):"
    name: jvm_memory_$1
  # Garbage collection
  - pattern: "java.lang<name=(.+), type=GarbageCollector>CollectionCount:"
    name: jvm_gc_collection_count
    labels:
      gc: "$1"
  # Threading
  - pattern: "java.lang<type=Threading>(.+):"
    name: jvm_threads_$1
```

### Run Command

```bash
java -javaagent:/path/to/jmx_exporter.jar=7071:/path/to/config.yml \
  -jar your-app.jar
```

## Telegraf

Flexible metrics collection for custom applications and infrastructure.

### Docker Installation

```bash
docker run -d \
  --name=telegraf \
  -p 9280:9280 \
  -v /path/to/telegraf.conf:/etc/telegraf/telegraf.conf:ro \
  telegraf:latest
```

### Configuration Example

```toml
[agent]
  interval = "10s"
  round_interval = true

[[outputs.prometheus_client]]
  listen = ":9280"
  path = "/metrics"

[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false

[[inputs.mem]]

[[inputs.net]]

[[inputs.netstat]]

[[inputs.processes]]

[[inputs.system]]
```

## VMware vSphere Exporter

Monitors vSphere clusters, hosts, and VMs.

### Installation

```bash
docker run -d \
  --name=vmware_exporter \
  -p 9272:9272 \
  pryorda/vmware_exporter:latest \
  -c /etc/vmware_exporter/config.yml
```

### Config Example

```yaml
vsphere:
  - host: vcenter.example.com
    user: monitoring@vsphere.local
    password: password
    validate_certs: false
    metrics_per_query: 500
    max_query_metrics: 250000
```

## HAProxy Exporter

Monitors load balancer metrics.

### Installation

```bash
docker run -d \
  --name=haproxy_exporter \
  -p 8105:8105 \
  prom/haproxy-exporter:latest \
  --haproxy.scrape-uri=http://localhost:8404/stats
```

## DNS Prober

Custom DNS latency and success monitoring.

### Docker Setup

```bash
docker run -d \
  --name=dns_prober \
  -p 9115:9115 \
  prom/blackbox-exporter:latest \
  --config.file=/etc/blackbox/config.yml
```

### Config for DNS

```yaml
modules:
  dns_probe:
    prober: dns
    timeout: 5s
    dns:
      preferred_ip_protocol: "ip4"
      query_name: "example.com"
      query_type: "A"
```

## TrueNAS Exporter

Monitors TrueNAS storage systems.

### Installation

```bash
docker run -d \
  --name=truenas_exporter \
  -p 9210:9210 \
  solarnz/truenas_exporter:latest \
  --truenas-api-key YOUR_API_KEY \
  --truenas-host truenas.local
```

## Fortigate (Firewall) Exporter

Monitors Fortinet firewall metrics.

### Installation

```bash
docker run -d \
  --name=fortigate_exporter \
  -p 9710:9710 \
  bluecmd/fortigate_exporter:latest \
  --target firewall.example.com \
  --api-token YOUR_TOKEN
```

## HP iLO Exporter

Monitors physical server hardware via iLO.

### Installation

```bash
docker run -d \
  --name=hpilo_exporter \
  -p 9416:9416 \
  mlowery/hpilo_exporter:latest
```

### Configuration

```yaml
targets:
  - 10.0.110.21
  - 10.0.110.22
  - 10.0.110.23
```

## Keepalived Exporter

Monitors high availability with Keepalived.

### Installation

```bash
docker run -d \
  --name=keepalived_exporter \
  -p 9165:9165 \
  noxiouz/keepalived_exporter:latest \
  --config /etc/keepalived_exporter.conf
```

## Chrony Exporter

Monitors NTP time synchronization.

### Installation

```bash
docker run -d \
  --name=chrony_exporter \
  -p 9109:9109 \
  ptimof/chrony_exporter:latest
```

## Netdata

Real-time system monitoring with metrics export.

### Installation

```bash
docker run -d \
  --name=netdata \
  -p 19999:19999 \
  --cap-add SYS_PTRACE \
  --security-opt apparmor=unconfined \
  netdata/netdata:latest
```

## MTR Exporter

Network route analysis and latency measurement.

### Installation

```bash
docker run -d \
  --name=mtr_exporter \
  -p 9570:9570 \
  ghcr.io/adinhodovic/mtr_exporter:latest
```

## ClickHouse Exporter

Analytics database monitoring.

### Installation

```bash
docker run -d \
  --name=clickhouse_exporter \
  -p 9363:9363 \
  clickhouse/clickhouse-exporter:latest \
  --clickhouse-server=localhost:8123
```

## Exporter Comparison Matrix

| Exporter | Component | Metrics | Complexity | Port | Setup |
|----------|-----------|---------|-----------|------|-------|
| Node Exporter | Linux/Unix | 200+ | Simple | 9100 | Easy |
| postgres_exporter | PostgreSQL | 100+ | Moderate | 9187 | Moderate |
| MongoDB Exporter | MongoDB | 150+ | Moderate | 9216 | Moderate |
| RabbitMQ | Message Queue | 200+ | Moderate | 15692 | Moderate |
| Kafka | Streaming | 100+ | Complex | 9308 | Complex |
| JMX Exporter | Java Apps | Custom | Complex | 7071 | Complex |
| Blackbox | Endpoints | Availability | Simple | 9115 | Easy |
| HAProxy | Load Balancer | 300+ | Simple | 8105 | Easy |
| Telegraf | Custom | Flexible | Moderate | 9280 | Moderate |
| VMware | vSphere | 500+ | Complex | 9272 | Complex |
| TrueNAS | Storage | 200+ | Moderate | 9210 | Moderate |
| Fortigate | Firewall | 300+ | Complex | 9710 | Complex |
| iLO | Hardware | 200+ | Moderate | 9416 | Moderate |
| Keepalived | HA | 100+ | Simple | 9165 | Easy |

## Best Practices

1. **Resource Management**: Monitor exporter memory and CPU usage
2. **Security**: Use API tokens and credentials securely
3. **Scrape Intervals**: Adjust per component requirements
4. **Aggregation**: Use recording rules for high-cardinality metrics
5. **Testing**: Verify metrics before production deployment
6. **Documentation**: Document custom exporter configurations
7. **Redundancy**: Run multiple exporter instances for critical services

## Troubleshooting Common Issues

### Connection Refused
- Verify service is running
- Check firewall rules
- Confirm port is correct

### No Metrics Returned
- Check exporter configuration
- Verify Prometheus can reach exporter
- Review exporter logs

### High Memory Usage
- Disable unnecessary collectors
- Reduce cardinality of metrics
- Increase scrape intervals

### Authentication Failures
- Verify credentials
- Check user permissions
- Review authentication logs

#### Related Documentation

- [Prometheus Official Exporters](https://prometheus.io/docs/instrumenting/exporters/)
- [Awesome Prometheus Exporters](https://github.com/prometheus-community/awesome-prometheus-alerts)
