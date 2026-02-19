# Blackbox Exporter Setup Guide

**Blackbox Exporter** is a versatile exporter that allows probing of endpoints over HTTP, HTTPS, DNS, and TCP to verify they are up and to measure response times and SSL certificate validity.

## Overview

- **Metrics**: Endpoint availability, latency, SSL expiration
- **Target Port**: `:9115`
- **Setup Complexity**: ⭐⭐ (Moderate)
- **Use Case**: Endpoint monitoring, SLA tracking, SSL certificate expiration alerts

## What It Monitors

### HTTP/HTTPS
- Response code and status
- Response time (latency)
- SSL certificate validity and expiration
- Content verification (regex matching)
- Redirect chains

### DNS
- Query response time
- DNS resolution success/failure
- Authority and additional records

### TCP
- Port connectivity
- Response time
- TLS handshake time and details

### ICMP (Ping)
- Packet loss
- Round trip time (RTT)

## Installation

### Docker

```bash
docker run -d \
  --name=blackbox_exporter \
  -p 9115:9115 \
  -v /path/to/blackbox.yml:/etc/blackbox_exporter/blackbox.yml:ro \
  prom/blackbox-exporter:latest \
  --config.file=/etc/blackbox_exporter/blackbox.yml
```

### Docker Compose

```yaml
blackbox_exporter:
  image: prom/blackbox-exporter:latest
  container_name: blackbox_exporter
  ports:
    - "9115:9115"
  volumes:
    - ./blackbox.yml:/etc/blackbox_exporter/blackbox.yml:ro
  command:
    - '--config.file=/etc/blackbox_exporter/blackbox.yml'
  restart: unless-stopped
```

### Systemd Service

1. Download binary:
```bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-amd64.tar.gz
tar xvfz blackbox_exporter-0.24.0.linux-amd64.tar.gz
sudo cp blackbox_exporter-0.24.0.linux-amd64/blackbox_exporter /usr/local/bin/
```

2. Create config (`/etc/blackbox_exporter/blackbox.yml`):
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      follow_redirects: true
      preferred_ip_protocol: "ip4"

  https_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      follow_redirects: true
      preferred_ip_protocol: "ip4"
      tls_config:
        insecure_skip_verify: false

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp_ipv4:
    prober: icmp
    timeout: 5s
```

3. Systemd service:
```ini
[Unit]
Description=Blackbox Exporter
After=network.target

[Service]
Type=simple
User=blackbox
ExecStart=/usr/local/bin/blackbox_exporter \
  --config.file=/etc/blackbox_exporter/blackbox.yml \
  --web.listen-address=:9115
Restart=always

[Install]
WantedBy=multi-user.target
```

## Configuration

### blackbox.yml Example

```yaml
modules:
  # HTTP check for 2xx status codes
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200, 201, 202, 204, 206, 301, 302, 304, 307, 308]
      follow_redirects: true
      preferred_ip_protocol: "ip4"

  # HTTPS check with certificate validation
  https_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200]
      follow_redirects: true
      preferred_ip_protocol: "ip4"
      tls_config:
        insecure_skip_verify: false

  # TCP connection check
  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      tls: false

  # TCP with TLS
  tcp_tls:
    prober: tcp
    timeout: 5s
    tcp:
      tls: true

  # ICMP ping
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"

  # DNS check
  dns_query:
    prober: dns
    timeout: 5s
    dns:
      preferred_ip_protocol: "ip4"
      query_name: "example.com"
      query_type: "A"
```

## Prometheus Configuration

### Basic HTTP Monitoring

```yaml
- job_name: 'blackbox_http'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://api.example.com
      - https://www.example.com
      - https://service.example.com
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: localhost:9115
```

### ICMP Monitoring

```yaml
- job_name: 'blackbox_icmp'
  metrics_path: /probe
  params:
    module: [icmp]
  static_configs:
    - targets:
      - 8.8.8.8
      - 1.1.1.1
      - gateway.local
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: localhost:9115
```

## Key PromQL Queries

### Availability
```promql
# Endpoint availability percentage
avg_over_time(probe_success[5m]) * 100

# Endpoints down
probe_success == 0

# Average response time
avg(probe_duration_seconds)
```

### SSL Certificates
```promql
# Days until SSL certificate expiration
(probe_ssl_earliest_cert_expiry - time()) / 86400

# Certificates expiring within 30 days
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 30

# Certificate validity check
probe_ssl_last_chain_expiry_timestamp_seconds > 0
```

### Performance
```promql
# HTTP duration
probe_http_duration_seconds

# Connect duration
probe_http_connect_duration_seconds

# TLS duration
probe_tls_duration_seconds
```

## Alerting Rules

```yaml
- alert: EndpointDown
  expr: probe_success == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Endpoint {{ $labels.instance }} is down"

- alert: SSLCertificateExpiring
  expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 7
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "SSL cert expiring in {{ $value }}d for {{ $labels.instance }}"

- alert: SSLCertificateExpired
  expr: probe_ssl_earliest_cert_expiry - time() < 0
  labels:
    severity: critical
  annotations:
    summary: "SSL cert expired for {{ $labels.instance }}"

- alert: HighResponseTime
  expr: probe_duration_seconds > 5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High response time: {{ $value }}s for {{ $labels.instance }}"
```

## Testing

### Manual Test
```bash
# Test HTTP endpoint
curl "http://localhost:9115/probe?module=http_2xx&target=https://example.com"

# Test ICMP
curl "http://localhost:9115/probe?module=icmp&target=8.8.8.8"

# Test DNS
curl "http://localhost:9115/probe?module=dns_query&target=example.com"
```

## Troubleshooting

### ICMP/Ping Not Working
Blackbox exporter may need `CAP_NET_RAW` capability:
```bash
sudo setcap cap_net_raw=ep /usr/local/bin/blackbox_exporter
```

### SSL Verification Failures
- Check certificate chain
- May need intermediate certificates
- Use `insecure_skip_verify: false` carefully

### DNS Resolution Issues
- Ensure DNS server is accessible
- Check firewall rules
- Verify `query_name` configuration

### High Memory Usage
- Limit concurrent probes
- Increase timeout values
- Review blackbox.yml configuration

## Best Practices

1. **Monitor critical endpoints** - Prioritize SLAs
2. **Set appropriate timeouts** - Usually 5-10 seconds
3. **Use modules** - Separate HTTP, ICMP, DNS configs
4. **Alert on SSL expiration** - Prevent certificate outages
5. **Monitor from multiple locations** - Detect regional issues
6. **Test regularly** - Validate checks are working
7. **Use TLS verification** - Catch certificate issues early

## Related Documentation

- [Official Blackbox Exporter GitHub](https://github.com/prometheus/blackbox_exporter)
- [Prometheus HTTP Probing](https://prometheus.io/docs/guides/node-exporter/#monitoring-node-exporter)
