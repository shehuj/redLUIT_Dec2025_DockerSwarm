# Monitoring Setup Guide

## Architecture Overview

This deployment uses a **separate monitoring server** architecture:

- **Docker Swarm Cluster**: Runs application services (web, api, redis, visualizer)
- **Monitoring Server**: Runs Prometheus and Grafana to monitor the swarm cluster

```
┌─────────────────────────────────────┐
│     Docker Swarm Cluster            │
│  ┌──────────┐  ┌──────────┐        │
│  │ Manager  │  │ Worker 1 │        │
│  │  :9323   │  │  :9323   │        │
│  └──────────┘  └──────────┘        │
│  ┌──────────┐  ┌──────────┐        │
│  │ Worker 2 │  │ Worker 3 │        │
│  │  :9323   │  │  :9323   │        │
│  └──────────┘  └──────────┘        │
└─────────────────────────────────────┘
           │
           │ Scrapes metrics
           ▼
┌─────────────────────────────────────┐
│    Monitoring Server                │
│  ┌──────────────┐  ┌──────────────┐│
│  │ Prometheus   │  │  Grafana     ││
│  │   :9090      │  │   :3000      ││
│  └──────────────┘  └──────────────┘│
│  ┌──────────────┐  ┌──────────────┐│
│  │ Node Exporter│  │  cAdvisor    ││
│  │   :9100      │  │   :8080      ││
│  └──────────────┘  └──────────────┘│
└─────────────────────────────────────┘
```

## Prerequisites

1. **Swarm Cluster**: Docker Swarm cluster already deployed
2. **Monitoring Server**: A separate server (Ubuntu recommended)
3. **Network Access**: Monitoring server must be able to reach swarm nodes on port 9323
4. **Firewall Rules**:
   - Swarm nodes: Port 9323 open to monitoring server
   - Monitoring server: Ports 3000 (Grafana), 9090 (Prometheus) open to your network

## Quick Start

### 1. Update Inventory

Edit `ansible/inventory.yml` and add your monitoring server:

```yaml
monitoring:
  hosts:
    monitoring_server:
      ansible_python_interpreter: /usr/bin/python3
      ansible_host: YOUR_MONITORING_SERVER_IP
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

### 2. Update Prometheus Config

Edit `configs/prometheus-monitoring.yml` and replace the IPs with your actual swarm node IPs:

```yaml
- job_name: 'swarm-manager'
  static_configs:
    - targets: ['YOUR_MANAGER_IP:9323']

- job_name: 'swarm-workers'
  static_configs:
    - targets:
        - 'YOUR_WORKER1_IP:9323'
        - 'YOUR_WORKER2_IP:9323'
        - 'YOUR_WORKER3_IP:9323'
```

### 3. Deploy Everything

**Deploy swarm cluster only:**
```bash
ansible-playbook ansible/site.yml
```

**Deploy swarm + monitoring:**
```bash
ansible-playbook ansible/site.yml -e 'setup_monitoring=true'
```

**Deploy monitoring separately:**
```bash
ansible-playbook ansible/playbooks/setup-monitoring.yml
```

## Configuration Files

### Monitoring Stack (`monitoring-stack.yml`)
Docker Compose file that deploys:
- Prometheus (port 9090)
- Grafana (port 3000)
- Node Exporter (port 9100) - monitors the monitoring server itself
- cAdvisor (port 8080) - monitors Docker containers on monitoring server

### Prometheus Config (`configs/prometheus-monitoring.yml`)
Defines what to monitor:
- Prometheus itself
- Monitoring server metrics
- Docker Swarm manager node (Docker metrics)
- Docker Swarm worker nodes (Docker metrics)
- Swarm visualizer (optional)

### Grafana Datasources (`configs/grafana-datasources.yml`)
Auto-configures Prometheus as a datasource in Grafana

## Access URLs

After deployment:

| Service | URL | Credentials |
|---------|-----|-------------|
| Grafana | http://MONITORING_SERVER_IP:3000 | admin / change-me-in-production |
| Prometheus | http://MONITORING_SERVER_IP:9090 | None |
| Node Exporter | http://MONITORING_SERVER_IP:9100/metrics | None |
| cAdvisor | http://MONITORING_SERVER_IP:8080 | None |

## Firewall Configuration

### On Swarm Nodes

Allow monitoring server to scrape metrics:
```bash
# If using UFW
sudo ufw allow from MONITORING_SERVER_IP to any port 9323 proto tcp

# If using AWS Security Groups
# Add inbound rule: TCP 9323 from monitoring server IP
```

### On Monitoring Server

Allow access to Grafana and Prometheus:
```bash
# If using UFW
sudo ufw allow 3000/tcp  # Grafana
sudo ufw allow 9090/tcp  # Prometheus

# If using AWS Security Groups
# Add inbound rules:
# - TCP 3000 from your IP (Grafana)
# - TCP 9090 from your IP (Prometheus)
```

## Grafana Dashboard Setup

### 1. Login to Grafana
Navigate to `http://MONITORING_SERVER_IP:3000`
- Username: `admin`
- Password: `change-me-in-production`

### 2. Import Dashboards

Grafana has many pre-built dashboards for Docker and Prometheus:

**Recommended Dashboard IDs:**
- **1860**: Node Exporter Full
- **893**: Docker and System Monitoring
- **179**: Docker Prometheus Monitoring

**To Import:**
1. Click `+` → `Import`
2. Enter dashboard ID
3. Select "Prometheus" as datasource
4. Click Import

### 3. Explore Metrics

Some useful Prometheus queries:

**Container CPU Usage:**
```promql
rate(container_cpu_usage_seconds_total{name=~"mystack_.*"}[5m]) * 100
```

**Container Memory Usage:**
```promql
container_memory_usage_bytes{name=~"mystack_.*"} / 1024 / 1024
```

**Swarm Service Replicas:**
```promql
swarm_service_replicas{job="swarm-manager"}
```

## Troubleshooting

### Prometheus Can't Scrape Swarm Nodes

**Check network connectivity:**
```bash
# From monitoring server
curl http://SWARM_NODE_IP:9323/metrics
```

**If connection refused:**
- Check firewall rules
- Verify Docker daemon is exposing metrics (check `/etc/docker/daemon.json`)
- Ensure swarm nodes have `"metrics-addr": "0.0.0.0:9323"` in daemon config

### Grafana Shows "No Data"

1. Check Prometheus targets: `http://MONITORING_SERVER_IP:9090/targets`
2. All targets should be "UP"
3. If targets are "DOWN", check network connectivity and firewall rules

### Services Not Appearing in Grafana

1. Verify Prometheus is scraping: Go to Prometheus → Graph → Enter query
2. Check if data exists: `up{job="swarm-manager"}`
3. If no data, check Prometheus logs:
   ```bash
   docker logs prometheus
   ```

## Maintenance

### Update Monitoring Stack

```bash
# On monitoring server
cd /opt/monitoring
docker compose pull
docker compose up -d
```

### View Logs

```bash
# On monitoring server
cd /opt/monitoring
docker compose logs -f prometheus
docker compose logs -f grafana
```

### Backup Grafana

```bash
# Backup Grafana data
docker run --rm --volumes-from grafana \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/grafana-backup.tar.gz /var/lib/grafana
```

### Backup Prometheus

```bash
# Backup Prometheus data
docker run --rm --volumes-from prometheus \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/prometheus-backup.tar.gz /prometheus
```

## Security Best Practices

1. **Change Default Passwords:**
   - Update Grafana admin password immediately
   - Set in `monitoring-stack.yml` under `GF_SECURITY_ADMIN_PASSWORD`

2. **Restrict Access:**
   - Use firewall rules to limit access to monitoring ports
   - Consider using VPN for accessing Grafana

3. **Use HTTPS:**
   - Set up reverse proxy (nginx/traefik) with SSL certificates
   - Use Let's Encrypt for free SSL certificates

4. **Authentication:**
   - Enable Grafana authentication
   - Consider OAuth/LDAP integration for enterprise use

## Advanced Configuration

### Add More Swarm Nodes

Edit `configs/prometheus-monitoring.yml`:
```yaml
- job_name: 'swarm-workers'
  static_configs:
    - targets:
        - 'worker1:9323'
        - 'worker2:9323'
        - 'worker3:9323'
        - 'worker4:9323'  # Add new node
```

Then restart Prometheus:
```bash
cd /opt/monitoring
docker compose restart prometheus
```

### Custom Alerts

Create `configs/prometheus-alerts.yml` with alert rules, then mount it in Prometheus.

### High Availability

For production, consider:
- Multiple Prometheus instances
- Prometheus federation
- Thanos for long-term storage
- Separate Grafana for each environment

## Support

For issues or questions:
1. Check Prometheus logs: `docker logs prometheus`
2. Check Grafana logs: `docker logs grafana`
3. Verify network connectivity between monitoring server and swarm nodes
4. Review firewall rules and security groups
