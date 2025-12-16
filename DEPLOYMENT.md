# Docker Swarm Deployment Guide

## 🚀 Quick Start - Full Automated Deployment

Deploy the complete Docker Swarm cluster with a single command:

```bash
cd ansible
ansible-playbook site.yml -i inventory.yml
```

This master playbook will:
1. Install Docker on all nodes
2. Initialize Docker Swarm
3. Deploy the application stack
4. Verify the deployment

---

## 📚 Step-by-Step Manual Deployment

### Prerequisites

1. **Update inventory** with your server IPs:
   ```bash
   vi ansible/inventory.yml
   ```

2. **Test connectivity**:
   ```bash
   ansible all -i inventory.yml -m ping
   ```

### Phase 1: Install Docker

```bash
ansible-playbook playbooks/install-docker.yml -i inventory.yml
```

**What it does:**
- Installs Docker CE with specific version pinning
- Configures Docker daemon with production settings
- Adds users to docker group
- Enables Docker metrics for Prometheus

### Phase 2: Initialize Swarm

```bash
ansible-playbook playbooks/swarm-setup.yml -i inventory.yml
```

**What it does:**
- Initializes Swarm on manager node
- Joins worker nodes to the cluster
- Labels nodes for placement constraints
- Verifies cluster health

### Phase 3: Create Secrets

```bash
ansible-playbook playbooks/create-secrets.yml -i inventory.yml
```

**Interactive prompts:**
- API secret key (for API service)
- Grafana admin password

**Tip:** Use strong passwords in production!

### Phase 4: Deploy Stack

```bash
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml
```

**What it does:**
- Copies stack file and configs to manager
- Creates default secrets (if not exist)
- Deploys multi-service stack
- Waits for services to be ready (up to 5 minutes)

### Phase 5: Verify Deployment

```bash
ansible-playbook playbooks/test-swarm.yml -i inventory.yml
```

**What it tests:**
- Cluster health
- Service availability
- Endpoint accessibility
- Monitoring stack status

---

## 🔄 Update Existing Stack (Zero Downtime)

Update services without downtime:

```bash
ansible-playbook playbooks/update-stack.yml -i inventory.yml
```

**Features:**
- Rolling updates with health checks
- Automatic rollback on failure
- Prunes unused resources
- Shows failed services (if any)

---

## 🗑️ Teardown

### Remove Stack Only

```bash
ansible-playbook playbooks/teardown.yml -i inventory.yml
```

**What it does:**
- Removes Docker stack
- Removes secrets
- Leaves Swarm cluster
- Keeps Docker installed

### Complete Cleanup (including Docker)

```bash
# First teardown the swarm
ansible-playbook playbooks/teardown.yml -i inventory.yml

# Then remove Docker (if needed)
ansible all -i inventory.yml -b -m apt -a "name=docker-ce state=absent purge=yes"
```

---

## 🔐 Production Security Checklist

### Before Production Deployment:

1. **Update Secrets:**
   ```bash
   ansible-playbook playbooks/create-secrets.yml -i inventory.yml
   ```
   - Use strong, random passwords
   - Store credentials in a password manager

2. **Configure Firewall:**
   ```bash
   # On all nodes
   sudo ufw allow 22/tcp      # SSH
   sudo ufw allow 2377/tcp    # Swarm management (manager only)
   sudo ufw allow 7946/tcp    # Swarm communication
   sudo ufw allow 7946/udp    # Swarm communication
   sudo ufw allow 4789/udp    # Overlay network

   # On manager node only
   sudo ufw allow 80/tcp      # HTTP
   sudo ufw allow 443/tcp     # HTTPS
   sudo ufw allow 9090/tcp    # Prometheus
   sudo ufw allow 3000/tcp    # Grafana
   sudo ufw allow 8080/tcp    # Visualizer

   sudo ufw enable
   ```

3. **Update docker-stack.yml:**
   - Change default passwords in secrets
   - Update image tags to specific versions
   - Adjust resource limits for your workload
   - Configure SSL/TLS certificates

4. **Enable Docker Content Trust:**
   ```bash
   export DOCKER_CONTENT_TRUST=1
   ```

5. **Regular Updates:**
   ```bash
   # Update Docker
   ansible-playbook playbooks/install-docker.yml -i inventory.yml

   # Update stack
   ansible-playbook playbooks/update-stack.yml -i inventory.yml
   ```

---

## 🔍 Monitoring & Troubleshooting

### Check Service Status

```bash
# On manager node
docker service ls
docker stack ps mystack
docker service logs mystack_web
```

### View Swarm Nodes

```bash
docker node ls
docker node inspect <NODE_NAME>
```

### Access Monitoring

- **Prometheus**: http://MANAGER_IP:9090
  - Query: `rate(container_cpu_usage_seconds_total[5m])`
  - Query: `docker_swarm_service_replicas_running`

- **Grafana**: http://MANAGER_IP:3000
  - Username: admin
  - Password: (from create-secrets.yml)

- **Visualizer**: http://MANAGER_IP:8080
  - Real-time cluster visualization

### Common Issues

#### Issue: Service Won't Start

```bash
# Check service logs
docker service logs mystack_web

# Check service details
docker service ps mystack_web --no-trunc

# Common causes:
# - Missing secrets
# - Image pull failure
# - Resource constraints
# - Network issues
```

#### Issue: Node Unreachable

```bash
# Check node status
docker node ls

# Check node health
ansible swarm_workers -i inventory.yml -m ping

# Rejoin node
ansible-playbook playbooks/swarm-setup.yml -i inventory.yml
```

#### Issue: Deployment Fails

```bash
# Check Ansible logs
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml -vvv

# Verify stack file syntax
docker compose -f docker-stack.yml config

# Check for port conflicts
sudo lsof -i :80
```

---

## 📊 Performance Tuning

### Adjust Service Replicas

Edit `docker-stack.yml`:
```yaml
services:
  web:
    deploy:
      replicas: 5  # Increase from 3
```

Then update:
```bash
ansible-playbook playbooks/update-stack.yml -i inventory.yml
```

### Scale Service Manually

```bash
# On manager node
docker service scale mystack_web=10
```

### Adjust Resource Limits

Edit `docker-stack.yml`:
```yaml
deploy:
  resources:
    limits:
      cpus: '2.0'      # Increase
      memory: 1024M    # Increase
```

---

## 🔄 Backup & Recovery

### Backup Swarm Configuration

```bash
# On manager node
docker swarm ca > swarm-ca.pem
docker node ls > nodes-backup.txt
docker service ls > services-backup.txt
```

### Backup Volumes

```bash
# For each service with volumes
docker run --rm \
  -v mystack_grafana_data:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/grafana-backup.tar.gz /data
```

### Restore from Backup

```bash
# Restore volume
docker run --rm \
  -v mystack_grafana_data:/data \
  -v $(pwd):/backup \
  ubuntu tar xzf /backup/grafana-backup.tar.gz -C /
```

---

## 🚦 CI/CD Integration

This repository includes automated GitHub Actions workflows.

### Workflow Triggers

- **PR to main/dev**: Runs validation, security scans, and tests
- **Push to main**: Full deployment to production
- **Manual**: Via workflow_dispatch

### Required Secrets

Set in GitHub Settings → Secrets:

| Secret | Description |
|--------|-------------|
| `SSH_PRIVATE_KEY` | Private key for SSH access |
| `SWARM_MANAGER_IP` | Manager node public IP |
| `API_SECRET_KEY` | API service secret |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin password |
| `AWS_ACCESS_KEY_ID` | For EC2 dynamic inventory (optional) |
| `AWS_SECRET_ACCESS_KEY` | For EC2 dynamic inventory (optional) |

---

## 📝 Environment Variables

The following can be customized via environment variables:

```bash
# Ansible settings
export ANSIBLE_HOST_KEY_CHECKING=False
export ANSIBLE_FORCE_COLOR=true

# Docker settings
export DOCKER_BUILDKIT=1
export COMPOSE_DOCKER_CLI_BUILD=1
```

---

## 🎯 Best Practices

1. **Always use version tags** for Docker images (never `:latest` in production)
2. **Test updates** in staging environment first
3. **Monitor resource usage** via Prometheus/Grafana
4. **Regular backups** of persistent data
5. **Keep Docker updated** to patch security vulnerabilities
6. **Use strong secrets** (generate with `openssl rand -base64 32`)
7. **Enable log rotation** (configured in install-docker.yml)
8. **Implement health checks** for all services (included in docker-stack.yml)
9. **Use placement constraints** to ensure high availability
10. **Document changes** in git commit messages

---

## 📚 Additional Resources

- [Ansible Documentation](https://docs.ansible.com/)
- [Docker Swarm Documentation](https://docs.docker.com/engine/swarm/)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)

---

**Last Updated:** December 2024
**Version:** 2.0.0
**Status:** Production Ready ✅
