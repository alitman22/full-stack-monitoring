# Quick Start Guide

Get the monitoring stack running in 5 minutes.

## Prerequisites

- Docker Desktop or Docker Engine 20.10+
- Docker Compose 2.0+
- 4GB RAM available
- Port availability: 9090, 3000, 9093

## 1. Clone Repository

```bash
git clone https://github.com/alitman22/full-stack-monitoring.git
cd full-stack-monitoring
```

## 2. Start Stack

```bash
docker-compose up -d
```

Wait 30 seconds for services to start.

## 3. Access Services

| Service | URL | Credentials |
|---------|-----|---|
| Prometheus | http://localhost:9090 | - |
| Grafana | http://localhost:3000 | admin / admin |
| AlertManager | http://localhost:9093 | - |

## 4. Verify Targets

1. Go to http://localhost:9090/targets
2. All targets should show "UP" (give 30s for first scrape)

## 5. View Dashboards

Dashboards are automatically provisioned on startup. In Grafana:

1. Home page shows available dashboards
2. Four main dashboards ready to use:
   - **Infrastructure Overview** - System metrics
   - **Endpoint Monitoring** - External services
   - **Database - PostgreSQL** - Database health
   - **Message Queues** - RabbitMQ & Kafka

**[📖 Full Dashboard Guide](docs/GRAFANA_DASHBOARDS.md)**

## 6. Add Your Servers

Edit `prometheus/prometheus.yml`:

```yaml
- job_name: 'my-server'
  static_configs:
    - targets: ['your-server-ip:9100']
```

Reload Prometheus:

```bash
docker-compose exec prometheus kill -HUP 1
```

## Common Issues

**No metrics appearing?**
- Wait 30 seconds for first scrape
- Check: http://localhost:9090/api/v1/query?query=up
- Verify exporter is running on target

**Can't reach Prometheus from Grafana?**
- Use URL: `http://prometheus:9090` (not localhost)

**Port already in use?**
- Edit docker-compose.yml ports
- Or: `lsof -i :9090` to find process

## Next Steps

1. **Configure Exporters**: See [exporter guides](exporters/)
2. **Setup Alerts**: Edit [alertmanager.yml](alertmanager.yml)
3. **Create Dashboards**: Use [dashboard guides](grafana/)
4. **High Availability**: See [HA setup](deployment/HIGH_AVAILABILITY.md)

## Documentation

- [Full README](README.md)
- [Architecture](ARCHITECTURE.md)
- [Docker Setup](deployment/DOCKER_SETUP.md)
- [Troubleshooting](docs/TROUBLESHOOTING.md)
- [Exporter References](exporters/)

## Get Help

Check logs:
```bash
docker-compose logs prometheus
docker-compose logs grafana
docker-compose logs alertmanager
```

Or search [troubleshooting guide](docs/TROUBLESHOOTING.md).

---

**Ready to explore?** Start with Prometheus at http://localhost:9090
