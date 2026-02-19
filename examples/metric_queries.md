# Common PromQL Queries

This guide provides commonly used PromQL (Prometheus Query Language) queries for monitoring different components.

## System Monitoring

### CPU Metrics

```promql
# CPU usage percentage (0-100)
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# CPU usage by mode
irate(node_cpu_seconds_total{mode!="idle"}[5m])

# Load average (5 minute)
node_load5

# Load average relative to number of CPUs
node_load5 / on(instance) count(count by (cpu) (node_cpu_seconds_total)) by (instance)

# CPU frequency
node_cpu_frequency_max_hertz / 1000000000
```

### Memory Metrics

```promql
# Memory available percentage
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Memory used percentage
((node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes) * 100

# Swap usage percentage
((node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes) / node_memory_SwapTotal_bytes) * 100

# Buffer and cache memory
node_memory_Buffers_bytes + node_memory_Cached_bytes
```

### Disk Metrics

```promql
# Disk usage percentage
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100

# Disk I/O read throughput (bytes/sec)
irate(node_disk_read_bytes_total[5m])

# Disk I/O write throughput (bytes/sec)
irate(node_disk_written_bytes_total[5m])

# Disk I/O operations per second
irate(node_disk_read_time_ms_total[5m]) + irate(node_disk_write_time_ms_total[5m])

# Root filesystem usage
(1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100
```

### Network Metrics

```promql
# Network traffic received (bytes/sec)
irate(node_network_receive_bytes_total{device!="lo"}[5m])

# Network traffic transmitted (bytes/sec)
irate(node_network_transmit_bytes_total{device!="lo"}[5m])

# Total network traffic (both directions)
irate(node_network_receive_bytes_total[5m]) + irate(node_network_transmit_bytes_total[5m])

# Network errors
irate(node_network_receive_errs_total[5m]) + irate(node_network_transmit_errs_total[5m])

# Dropped packets
irate(node_network_receive_drop_total[5m]) + irate(node_network_transmit_drop_total[5m])

# TCP connections in different states
node_netstat_Tcp_curr_estab
```

## Database Monitoring

### PostgreSQL

```promql
# Database connections
pg_stat_activity_count

# Connection limit usage percentage
(pg_stat_activity_count / pg_settings_max_connections) * 100

# Transactions per second
rate(pg_stat_database_xact_commit[5m]) + rate(pg_stat_database_xact_rollback[5m])

# Cache hit ratio percentage
(rate(pg_stat_database_blks_hit[5m]) / (rate(pg_stat_database_blks_hit[5m]) + rate(pg_stat_database_blks_read[5m]))) * 100

# Replication lag in seconds
pg_replication_lag_seconds

# Replication lag in bytes
pg_replication_slots_restart_lsn_bytes - pg_replication_slots_confirmed_flush_lsn_bytes

# Database size
pg_database_size_bytes / 1024 / 1024 / 1024

# Autovacuum runtime
pg_stat_user_tables_last_autovacuum_time
```

### MongoDB

```promql
# Document operations per second
rate(mongodb_op_total[5m])

# Memory usage
mongodb_memory_usage_bytes

# Replica set oplog window (seconds)
(mongodb_oplog_lag_seconds)

# Replication health
mongodb_repl_health

# Slow queries
increase(mongodb_slow_queries_total[5m])
```

## Application Monitoring

### RabbitMQ

```promql
# Queue depth
rabbitmq_queue_messages_ready

# Messages received per second
rate(rabbitmq_queue_messages_total[5m])

# Messages acknowledged per second
rate(rabbitmq_queue_messages_acked_total[5m])

# Active connections
rabbitmq_connections

# Consumer count
rabbitmq_consumers

# Queue ingestion rate (messages/sec)
rate(rabbitmq_queue_messages_recognized_total[5m])
```

### Kafka

```promql
# Broker leaders
kafka_broker_leader_count

# Replicas
kafka_broker_replica_count

# Partitions
kafka_topic_partitions

# Under-replicated partitions
kafka_topic_partition_replicas_in_sync < kafka_topic_partitions

# Consumer group lag
kafka_consumer_group_lag
```

## Endpoint Monitoring

### Blackbox Exporter

```promql
# Endpoint availability percentage
(count(probe_success == 1) / count(probe_success)) * 100

# Average response time
avg(probe_duration_seconds) by (instance)

# SSL certificate days until expiration
(probe_ssl_earliest_cert_expiry - time()) / 86400

# Endpoints with expiring certificates (< 30 days)
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 30 and (probe_ssl_earliest_cert_expiry - time()) > 0

# DNS query time
probe_dns_lookup_time_seconds

# HTTP response code
probe_http_status_code
```

## Prometheus Internal

```promql
# Prometheus up status
up{job="prometheus"}

# Prometheus scrape duration
increase(prometheus_tsdb_compactions_total[5m])

# Scrape target availability
count(up == 1) / count(up)

# Prometheus storage size
prometheus_tsdb_symbol_table_size_bytes + prometheus_tsdb_index_size_bytes

# Prometheus samples ingested per second
rate(prometheus_tsdb_samples_total[5m])

# Database ingestion rate
rate(prometheus_tsdb_samples_total[5m])

# Average query time
histogram_quantile(0.99, rate(prometheus_tsdb_compaction_duration_seconds_bucket[5m]))
```

## Complex Queries

### Multi-Resource Analysis

```promql
# Show instances with high CPU and memory
(100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80)
AND on(instance)
((node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.8)
```

### Top Consumers

```promql
# Top 5 instances by CPU usage
topk(5, 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))

# Top 5 instances by memory usage
topk(5, (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes)

# Top 5 network interfaces by traffic
topk(5, irate(node_network_receive_bytes_total[5m]) + irate(node_network_transmit_bytes_total[5m]))
```

### Rate of Change

```promql
# Disk space growing rate (bytes/day)
rate(node_filesystem_size_bytes[1d]) - rate(node_filesystem_avail_bytes[1d])

# Projected disk full date (rough estimate)
predict_linear(node_filesystem_avail_bytes[1h], 7*24*3600) < 0

# Memory pressure increasing
rate(node_memory_MemTotal_bytes[1h]) > 0
```

## Query Tips

### Efficiency

1. **Use specific time ranges** - Add `[5m]` or `[1h]` to rate queries
2. **Filter by labels** - Use `{label="value"}` to reduce dataset
3. **Aggregate early** - Use `sum by (label)` to reduce cardinality
4. **Use recording rules** - Pre-compute common queries

### Debugging

```promql
# Find all available metrics
{__name__=~".*"}

# Check metric types
{__name__="node_cpu_seconds_total"}

# Find metrics with specific label
{instance="localhost:9100"}

# Check instance availability
up
```

## Common Patterns

### Availability

```promql
# Service availability over time
avg_over_time(up[5m])

# Availability percentage (last 24h)
avg_over_time(probe_success[24h]) * 100
```

### Performance

```promql
# P95 latency
histogram_quantile(0.95, probe_duration_seconds)

# P99 latency
histogram_quantile(0.99, probe_duration_seconds)

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
```

### Capacity

```promql
# Days until disk full
(node_filesystem_avail_bytes / rate(node_filesystem_size_bytes[24h])) / 86400
```
