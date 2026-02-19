# PostgreSQL Exporter Setup Guide

**postgres_exporter** collects metrics from PostgreSQL databases, providing insights into database performance, replication status, and query statistics.

## Overview

- **Metrics**: 100+ database metrics
- **Target Port**: `:9187`
- **Setup Complexity**: ⭐⭐ (Moderate)
- **Use Case**: Database monitoring, replication tracking, performance analysis

## What It Monitors

### Database Statistics
- Number of databases and tables
- Database size and table sizes
- Dead tuples and autovacuum status

### Connections
- Active connections
- Idle connections
- Connection limits and usage percentage
- Connection by state

### Performance
- Transactions per second (TPS)
- Query cache hit ratio
- Sequential vs index scans
- Full table scans
- Disk read/write metrics

### Replication
- Replication lag (in bytes and seconds)
- Standby status
- WAL (Write-Ahead Log) metrics
- Replication slot status

### Indexes
- Index bloat
- Index size and usage
- Missing indexes detection
- Unused indexes

## Installation

### Docker

```bash
docker run -d \
  --name=postgres_exporter \
  -p 9187:9187 \
  -e DATA_SOURCE_NAME="postgresql://user:password@localhost:5432/postgres?sslmode=disable" \
  prometheuscommunity/postgres-exporter:latest
```

### Docker Compose

```yaml
postgres_exporter:
  image: prometheuscommunity/postgres-exporter:latest
  container_name: postgres_exporter
  ports:
    - "9187:9187"
  environment:
    DATA_SOURCE_NAME: "postgresql://user:password@postgres:5432/postgres?sslmode=disable"
  depends_on:
    - postgres
  restart: unless-stopped
```

### Manual Installation

1. Download binary:
```bash
wget https://github.com/prometheuscommunity/postgres_exporter/releases/download/v0.13.2/postgres_exporter-0.13.2.linux-amd64.tar.gz
tar xvfz postgres_exporter-0.13.2.linux-amd64.tar.gz
sudo cp postgres_exporter-0.13.2.linux-amd64/postgres_exporter /usr/local/bin/
```

2. Create `.env` file:
```bash
cat > ~/.postgres_exporter << EOF
export DATA_SOURCE_NAME="postgresql://user:password@localhost:5432/postgres?sslmode=disable"
EOF
```

3. Systemd service:
```ini
[Unit]
Description=PostgreSQL Exporter
After=network.target

[Service]
EnvironmentFile=/home/postgres_exporter/.postgres_exporter
ExecStart=/usr/local/bin/postgres_exporter --web.listen-address=":9187"
Restart=always
User=postgres_exporter

[Install]
WantedBy=multi-user.target
```

## PostgreSQL User Setup

Create a monitoring user with necessary permissions:

```sql
-- Create monitoring user
CREATE USER pg_monitor WITH PASSWORD 'strong_password';

-- Grant necessary permissions
GRANT CONNECT ON DATABASE postgres TO pg_monitor;
GRANT pg_monitor TO postgres;
GRANT EXECUTE ON FUNCTION pg_database_size(oid) TO pg_monitor;

-- Enable required extensions (optional but recommended)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

## Connection String Format

```
postgresql://username:password@host:port/database?sslmode=disable
```

### Options
- `sslmode=disable` - No SSL (dev only)
- `sslmode=require` - Require SSL
- `sslmode=verify-ca` - Verify certificate
- `sslmode=verify-full` - Full verification

## Prometheus Configuration

```yaml
- job_name: 'postgres_exporter'
  static_configs:
    - targets: ['localhost:9187']
```

### Multiple Databases

```yaml
- job_name: 'postgres_exporter_primary'
  relabel_configs:
    - source_labels: [__address__]
      target_label: instance
      replacement: 'primary-db'
  static_configs:
    - targets: ['primary-db:9187']

- job_name: 'postgres_exporter_standby'
  relabel_configs:
    - source_labels: [__address__]
      target_label: instance
      replacement: 'standby-db'
  static_configs:
    - targets: ['standby-db:9187']
```

## Key PromQL Queries

### Connections
```promql
# Current connections
pg_stat_activity_count

# Connection limit usage
pg_stat_activity_count / pg_settings_max_connections

# Idle connections
pg_stat_activity_count{state="idle"}
```

### Performance
```promql
# Transactions per second
rate(pg_stat_database_xact_commit[5m]) + rate(pg_stat_database_xact_rollback[5m])

# Cache hit ratio
rate(pg_stat_database_blks_hit[5m]) / (rate(pg_stat_database_blks_hit[5m]) + rate(pg_stat_database_blks_read[5m]))

# Full table scans
rate(pg_stat_user_tables_seq_scan[5m])
```

### Replication
```promql
# Replication lag in bytes
pg_replication_slots_restart_lsn_bytes - pg_replication_slots_confirmed_flush_lsn_bytes

# Replication lag in seconds
pg_replication_lag_seconds

# Standby behind primary
pg_wal_lsn_last_removed_segfile - pg_wal_lsn_last_archove_wal_file
```

### Database Size
```promql
# Database size
pg_database_size_bytes

# Table size
pg_total_relation_size_bytes{schemaname="public"}

# Index size
pg_indexes_size_bytes
```

## Alerting Rules

```yaml
- alert: PostgreSQLDown
  expr: pg_up == 0
  for: 1m
  labels:
    severity: critical

- alert: PostgreSQLReplicationLag
  expr: pg_replication_lag_seconds > 10
  for: 5m
  labels:
    severity: warning

- alert: PostgreSQLConnectionLimit
  expr: (pg_stat_activity_count / pg_settings_max_connections) > 0.8
  for: 5m
  labels:
    severity: warning

- alert: PostgreSQLSlowQueries
  expr: rate(pg_stat_statements_mean_exec_time{query!="UNKNOWN"}[5m]) > 1000
  labels:
    severity: warning

- alert: PostgreSQLCacheHitRatioLow
  expr: rate(pg_stat_database_blks_hit[5m]) / (rate(pg_stat_database_blks_hit[5m]) + rate(pg_stat_database_blks_read[5m])) < 0.99
  for: 5m
  labels:
    severity: warning
```

## Troubleshooting

### Connection Refused
```bash
# Test connection
psql -U pg_monitor -h localhost -c "SELECT 1"
```

### SSL Error
Use `sslmode=disable` for testing:
```bash
export DATA_SOURCE_NAME="postgresql://user:password@localhost/postgres?sslmode=disable"
```

### Permission Denied
Ensure monitoring user has necessary permissions:
```sql
GRANT pg_monitor TO monitoring_user;
```

### High Memory Usage
Disable PG stat statements collection:
```bash
postgres_exporter --disable-default-metrics
```

## Best Practices

1. **Create dedicated user**: Never use superuser
2. **Use SSL in production**: Set `sslmode=require`
3. **Monitor replication**: Set alerts for replication lag
4. **Track cache hit ratio**: Target > 99% for production
5. **Monitor query performance**: Use pg_stat_statements
6. **Regular backups**: Test recovery procedures
7. **Connection pooling**: Use PgBouncer for connection management

## Related Documentation

- [Official postgres_exporter GitHub](https://github.com/prometheuscommunity/postgres_exporter)
- [PostgreSQL Metrics Documentation](https://www.postgresql.org/docs/current/monitoring-stats.html)
