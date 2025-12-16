# Production Deployment Checklist

## ✅ Pre-Deployment Checklist

### 1. Infrastructure Requirements

- [ ] **Minimum 3 EC2 instances** (1 manager, 2 workers)
- [ ] **Instance type**: t3.medium or better (2 vCPU, 4GB RAM minimum)
- [ ] **OS**: Ubuntu 22.04 LTS
- [ ] **Disk**: 20GB minimum per node
- [ ] **Network**: VPC with proper subnets configured
- [ ] **Security Groups**: Configured with required ports (see below)

### 2. AWS Security Group Configuration

**Manager Node:**
```
- SSH (22) - From your IP or bastion
- HTTP (80) - From 0.0.0.0/0
- HTTPS (443) - From 0.0.0.0/0
- Swarm (2377) - From worker nodes
- Overlay (7946 TCP/UDP) - From all swarm nodes
- Overlay (4789 UDP) - From all swarm nodes
- Prometheus (9090) - From monitoring systems
- Grafana (3000) - From trusted IPs
- Visualizer (8080) - From trusted IPs
```

**Worker Nodes:**
```
- SSH (22) - From your IP or bastion
- Swarm (2377) - From manager node
- Overlay (7946 TCP/UDP) - From all swarm nodes
- Overlay (4789 UDP) - From all swarm nodes
```

### 3. SSH Configuration

- [ ] **SSH key pair created** and saved securely
- [ ] **Key permissions**: `chmod 600 your-key.pem`
- [ ] **SSH access tested**: `ssh -i key.pem ubuntu@MANAGER_IP`
- [ ] **Ansible inventory updated** with correct IPs and key paths

### 4. GitHub Secrets Configuration

Navigate to Settings → Secrets → Actions and add:

- [ ] `SSH_PRIVATE_KEY` - Content of your private key file
- [ ] `SWARM_MANAGER_IP` - Public IP of manager node
- [ ] `API_SECRET_KEY` - Strong random string (32+ characters)
- [ ] `GRAFANA_ADMIN_PASSWORD` - Strong password (16+ characters)
- [ ] `AWS_ACCESS_KEY_ID` - (Optional, for EC2 dynamic inventory)
- [ ] `AWS_SECRET_ACCESS_KEY` - (Optional, for EC2 dynamic inventory)

**Generate strong secrets:**
```bash
# API Secret Key
openssl rand -base64 32

# Grafana Password
openssl rand -base64 24
```

### 5. Configuration Files Review

- [ ] **ansible/inventory.yml** - IPs and SSH paths updated
- [ ] **docker-stack.yml** - Image tags are specific versions (not :latest)
- [ ] **docker-stack.yml** - Resource limits appropriate for your instances
- [ ] **configs/nginx.conf** - Domain names updated (if applicable)
- [ ] **configs/prometheus.yml** - Scrape targets configured

### 6. DNS Configuration (Optional but Recommended)

- [ ] Domain purchased and configured
- [ ] A records pointing to manager node public IP
- [ ] SSL certificates obtained (Let's Encrypt recommended)
- [ ] Update nginx.conf with SSL configuration

---

## 🚀 Deployment Steps

### Step 1: Local Validation

```bash
cd ansible

# Test connectivity
ansible all -i inventory.yml -m ping

# Syntax check
ansible-playbook site.yml -i inventory.yml --syntax-check

# Dry run (check mode)
ansible-playbook site.yml -i inventory.yml --check
```

- [ ] All nodes respond to ping
- [ ] Syntax check passes
- [ ] No errors in check mode

### Step 2: Docker Installation

```bash
ansible-playbook playbooks/install-docker.yml -i inventory.yml
```

**Verify:**
- [ ] Docker installed on all nodes
- [ ] Docker service running
- [ ] Docker version consistent across nodes
- [ ] Users added to docker group

```bash
# Verify on each node
ansible all -i inventory.yml -b -m shell -a "docker --version"
ansible all -i inventory.yml -b -m shell -a "docker info"
```

### Step 3: Swarm Initialization

```bash
ansible-playbook playbooks/swarm-setup.yml -i inventory.yml
```

**Verify:**
- [ ] Swarm initialized on manager
- [ ] Workers joined successfully
- [ ] All nodes show as "Ready"
- [ ] Manager status shows "Leader"

```bash
# On manager node
docker node ls
```

Expected output:
```
ID                    HOSTNAME   STATUS   AVAILABILITY   MANAGER STATUS
xxx (self)            manager    Ready    Active         Leader
yyy                   worker1    Ready    Active
zzz                   worker2    Ready    Active
```

### Step 4: Create Secrets

```bash
ansible-playbook playbooks/create-secrets.yml -i inventory.yml
```

**Verify:**
- [ ] Secrets created successfully
- [ ] Strong passwords used (not defaults)

```bash
# On manager node
docker secret ls
```

### Step 5: Deploy Application Stack

```bash
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml
```

**Wait for deployment (5-10 minutes)**

**Verify:**
- [ ] All services created
- [ ] Replicas match desired state (e.g., 3/3)
- [ ] No services in "Pending" or "Failed" state

```bash
# On manager node
docker service ls
docker stack ps mystack
```

### Step 6: Health Checks

```bash
ansible-playbook playbooks/test-swarm.yml -i inventory.yml
```

**Verify:**
- [ ] All health checks pass
- [ ] Web service responds: `curl http://MANAGER_IP/health`
- [ ] Prometheus accessible: `http://MANAGER_IP:9090`
- [ ] Grafana accessible: `http://MANAGER_IP:3000`
- [ ] Visualizer shows cluster: `http://MANAGER_IP:8080`

---

## 📊 Post-Deployment Validation

### Application Testing

- [ ] **Web Service**: Load http://MANAGER_IP in browser
- [ ] **API Service**: Test API endpoints (if applicable)
- [ ] **Redis**: Connection test successful
- [ ] **Load Balancing**: Multiple requests hit different containers

### Monitoring Setup

- [ ] **Prometheus** - Targets showing "UP"
  - Navigate to Status → Targets
  - Verify all targets are healthy

- [ ] **Grafana** - Dashboard configured
  - Login with admin credentials
  - Add Prometheus data source: `http://prometheus:9090`
  - Import Docker Swarm dashboards

- [ ] **Visualizer** - Shows all nodes and services
  - Verify service distribution
  - Check node health status

### Security Validation

- [ ] **Secrets** - No plaintext passwords in configs
- [ ] **Firewall** - Only required ports open
- [ ] **SSH** - Password authentication disabled
- [ ] **Docker Daemon** - Not exposed publicly
- [ ] **TLS** - Swarm communication encrypted (automatic)

### Performance Baseline

```bash
# CPU usage
docker stats --no-stream

# Memory usage per service
docker service ps mystack_web --format "table {{.Name}}\t{{.Node}}"

# Network traffic
docker network ls
```

- [ ] CPU usage under 70%
- [ ] Memory usage appropriate
- [ ] No OOM errors in logs
- [ ] Network connectivity normal

---

## 🔧 Troubleshooting Guide

### Issue: Service Won't Start

**Symptoms:** Replicas show 0/3

**Debug steps:**
```bash
# Check service logs
docker service logs mystack_web

# Check tasks
docker service ps mystack_web --no-trunc

# Common causes:
# - Image pull failure
# - Missing secrets
# - Resource constraints
# - Port conflicts
```

**Solutions:**
```bash
# If image issue, manually pull
docker pull nginx:1.25-alpine

# If resource issue, increase limits in docker-stack.yml
# If port conflict, change exposed port
```

### Issue: Worker Node Unreachable

**Symptoms:** Node shows "Down" in docker node ls

**Debug steps:**
```bash
# On manager
docker node ls

# On problem worker
docker info | grep Swarm
sudo systemctl status docker
```

**Solutions:**
```bash
# Restart Docker
sudo systemctl restart docker

# Rejoin swarm
ansible-playbook playbooks/swarm-setup.yml -i inventory.yml --limit worker1
```

### Issue: Deployment Slow

**Symptoms:** Services take >10 minutes to deploy

**Debug steps:**
```bash
# Check image pull progress
docker service ps mystack_web --no-trunc

# Check network
ping worker1
```

**Solutions:**
```bash
# Pre-pull images on all nodes
ansible all -i inventory.yml -b -m shell -a "docker pull nginx:1.25-alpine"

# Check network bandwidth
# Consider using local registry for faster pulls
```

---

## 🔄 Maintenance Procedures

### Weekly Maintenance

- [ ] Check system updates: `ansible all -i inventory.yml -b -m apt -a "update_cache=yes"`
- [ ] Review service logs for errors
- [ ] Check disk usage: `df -h`
- [ ] Verify backup success
- [ ] Review Prometheus alerts

### Monthly Maintenance

- [ ] Update Docker: `ansible-playbook playbooks/install-docker.yml -i inventory.yml`
- [ ] Rotate secrets: `ansible-playbook playbooks/create-secrets.yml -i inventory.yml`
- [ ] Review and update docker-stack.yml versions
- [ ] Test disaster recovery procedures
- [ ] Review and update firewall rules

### Update Procedure

```bash
# 1. Update docker-stack.yml with new image versions
vi docker-stack.yml

# 2. Deploy update (zero downtime)
ansible-playbook playbooks/update-stack.yml -i inventory.yml

# 3. Monitor rollout
docker service ps mystack_web

# 4. Rollback if needed
docker service rollback mystack_web
```

---

## 🚨 Emergency Procedures

### Complete Stack Failure

```bash
# 1. Check cluster status
docker node ls
docker service ls

# 2. Remove failed stack
docker stack rm mystack

# 3. Redeploy
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml
```

### Manager Node Failure

```bash
# If you have multiple managers, promote worker:
docker node promote worker1

# If single manager, reinitialize:
ansible-playbook playbooks/swarm-setup.yml -i inventory.yml
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml
```

### Data Recovery

```bash
# Restore from backup (example for Grafana)
docker run --rm \
  -v mystack_grafana_data:/data \
  -v $(pwd):/backup \
  ubuntu tar xzf /backup/grafana-backup.tar.gz -C /
```

---

## 📝 Documentation

### As-Built Documentation

Document your actual deployment:

- [ ] Node IPs and instance IDs
- [ ] AWS account and region
- [ ] VPC and subnet IDs
- [ ] Security group IDs
- [ ] DNS records
- [ ] Secret rotation schedule
- [ ] Backup schedule and location
- [ ] Team contacts and escalation

### Runbook

Create team runbooks for:

- [ ] New service deployment
- [ ] Service update procedure
- [ ] Scaling up/down
- [ ] Incident response
- [ ] Backup/restore procedure
- [ ] Monitoring alerts and responses

---

## ✅ Sign-Off

**Deployment completed by:** ___________________

**Date:** ___________________

**Environment:** Production / Staging / Development

**Verification:**
- [ ] All services running
- [ ] Health checks passing
- [ ] Monitoring configured
- [ ] Backups scheduled
- [ ] Documentation updated
- [ ] Team notified

**Notes:**
___________________________________________
___________________________________________
___________________________________________

---

## 📞 Support Contacts

**Infrastructure:**
- Primary: ___________________
- Secondary: ___________________

**Application:**
- Primary: ___________________
- Secondary: ___________________

**Escalation:**
- Manager: ___________________
- On-Call: ___________________

---

**Document Version:** 1.0
**Last Updated:** December 2024
**Next Review:** ___________________
