# Docker Swarm Deployment with Ansible & CI/CD

[![Docker Swarm CI/CD Pipeline](https://github.com/YOUR_USERNAME/redLUIT_Dec2025_DockerSwarm/actions/workflows/ansible-deploy.yml/badge.svg)](https://github.com/YOUR_USERNAME/redLUIT_Dec2025_DockerSwarm/actions/workflows/ansible-deploy.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Production-ready Docker Swarm cluster deployment using Ansible automation with comprehensive CI/CD pipeline, monitoring, and security scanning.

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Monitoring](#monitoring)
- [Production Operations](#-production-operations)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Security](#security)
- [Production Standards](#-production-standards)
- [Contributing](#contributing)

---

## 🎯 Overview

This project automates the deployment and management of a Docker Swarm cluster using Ansible, with a complete CI/CD pipeline for testing, security scanning, and automated deployment. The setup includes:

- **Multi-node Docker Swarm cluster** with manager and worker nodes
- **Production-ready stack** with web services, API, caching, and monitoring
- **Automated CI/CD** with GitHub Actions
- **Comprehensive testing** including security scans and health checks
- **Monitoring stack** with Prometheus, Grafana, and Swarm Visualizer
- **Infrastructure as Code** using Ansible playbooks

---

## ✨ Features

### Infrastructure Management
- ✅ Automated Docker installation and configuration
- ✅ Docker Swarm cluster initialization and management
- ✅ Support for AWS EC2 dynamic inventory
- ✅ Static inventory for on-premise deployments
- ✅ Idempotent playbooks with proper error handling
- ✅ Automated backup and restore procedures
- ✅ Health check monitoring and validation
- ✅ Rollback capabilities for failed deployments

### Application Stack
- ✅ Multi-service application deployment
- ✅ Load-balanced web tier with Nginx
- ✅ API service with Node.js
- ✅ Redis caching layer
- ✅ Health checks for all services
- ✅ Secrets management with Docker Secrets

### Monitoring & Observability
- ✅ Prometheus for metrics collection
- ✅ Grafana for visualization
- ✅ Docker Swarm Visualizer
- ✅ Service health monitoring
- ✅ Automated alerting capabilities

### CI/CD Pipeline
- ✅ Branch-based deployment strategy (dev → main)
- ✅ Automated validation and linting
- ✅ Security scanning (Trivy, Gitleaks)
- ✅ Ansible playbook testing
- ✅ Docker stack validation
- ✅ Dry-run testing on dev branch
- ✅ Production deployment on main branch
- ✅ Manual Jenkins deployment workflow
- ✅ Post-deployment verification
- ✅ Pinned dependency versions

### Security
- ✅ Secret scanning
- ✅ Infrastructure as Code (IaC) scanning
- ✅ Hardcoded credential detection
- ✅ SARIF upload to GitHub Security
- ✅ Docker daemon security configuration
- ✅ Network isolation with overlay networks

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
│                      (Optional)                         │
└─────────────────────┬───────────────────────────────────┘
                      │
        ┌─────────────┴──────────────┐
        │                            │
┌───────▼────────┐          ┌────────▼───────┐
│ Swarm Manager  │          │ Swarm Worker 1 │
│                │          │                │
│ - Orchestrator │◄────────►│ - Web Service  │
│ - Prometheus   │          │ - API Service  │
│ - Grafana      │          │ - Redis        │
│ - Visualizer   │          └────────────────┘
└────────────────┘                   │
        │                   ┌────────▼───────┐
        └──────────────────►│ Swarm Worker 2 │
                            │                │
                            │ - Web Service  │
                            │ - API Service  │
                            │ - Redis        │
                            └────────────────┘
```

### Service Distribution

| Service | Replicas | Placement | Ports |
|---------|----------|-----------|-------|
| **Web (Nginx)** | 3 | Workers | 80, 443 |
| **API (Node.js)** | 2 | Workers | Internal |
| **Redis** | 1 | Workers | Internal |
| **Prometheus** | 1 | Manager | 9090 |
| **Grafana** | 1 | Manager | 3000 |
| **Visualizer** | 1 | Manager | 8080 |

---

## 📦 Prerequisites

### Local Development
- Python 3.11+
- Ansible 2.15+
- Docker & Docker Compose
- SSH access to target servers
- AWS CLI (if using EC2)

### Target Servers
- Ubuntu 22.04 LTS (recommended)
- Minimum 2GB RAM per node
- 20GB disk space
- Open ports: 22 (SSH), 80, 443, 2377, 7946, 4789

### GitHub Secrets (for CI/CD)

Navigate to **Settings → Secrets and variables → Actions** and add:

| Secret Name | Description | Required |
|-------------|-------------|----------|
| `SSH_PRIVATE_KEY` | Private key for SSH access to servers | Yes |
| `SWARM_MANAGER_IP` | Public IP of Swarm manager node | Yes |
| `API_SECRET_KEY` | Secret key for API service | Yes |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin password | Yes |
| `AWS_ACCESS_KEY_ID` | AWS access key (for EC2 inventory) | Optional |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key (for EC2 inventory) | Optional |

---

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/redLUIT_Dec2025_DockerSwarm.git
cd redLUIT_Dec2025_DockerSwarm
```

### 2. Configure Inventory

**Option A: Static Inventory** (for on-premise/known IPs)

Edit `ansible/inventory.yml`:

```yaml
all:
  children:
    swarm_manager:
      hosts:
        manager:
          ansible_host: 54.123.45.67
          ansible_user: ubuntu
          ansible_ssh_private_key_file: ~/.ssh/your_key.pem
    swarm_workers:
      hosts:
        worker1:
          ansible_host: 54.123.45.68
          ansible_user: ubuntu
          ansible_ssh_private_key_file: ~/.ssh/your_key.pem
        worker2:
          ansible_host: 54.123.45.69
          ansible_user: ubuntu
          ansible_ssh_private_key_file: ~/.ssh/your_key.pem
```

**Option B: AWS EC2 Dynamic Inventory**

The repository includes `ansible/aws_ec2.yml` for automatic EC2 discovery. Ensure your EC2 instances have these tags:

```
Role: docker-swarm
SwarmRole: manager  (for manager nodes)
SwarmRole: worker   (for worker nodes)
Environment: production
```

### 3. Install Ansible Dependencies

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

### 4. Test Connectivity

```bash
ansible all -i inventory.yml -m ping
```

### 5. Deploy the Cluster

```bash
# Install Docker on all nodes
ansible-playbook playbooks/install-docker.yml -i inventory.yml

# Initialize Swarm cluster
ansible-playbook playbooks/swarm-setup.yml -i inventory.yml

# Create Docker secrets
ansible swarm_manager -i inventory.yml -b -m shell -a "echo 'your-api-secret' | docker secret create api_secret_key -"
ansible swarm_manager -i inventory.yml -b -m shell -a "echo 'your-grafana-password' | docker secret create grafana_admin_password -"

# Deploy application stack
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml
```

### 6. Verify Deployment

```bash
# Run test suite
ansible-playbook playbooks/test-swarm.yml -i inventory.yml

# Check services manually
ansible swarm_manager -i inventory.yml -b -m shell -a "docker service ls"
```

### 7. Access Services

After deployment, access your services:

- **Web Application**: http://YOUR_MANAGER_IP
- **Prometheus**: http://YOUR_MANAGER_IP:9090
- **Grafana**: http://YOUR_MANAGER_IP:3000 (admin / YOUR_GRAFANA_PASSWORD)
- **Swarm Visualizer**: http://YOUR_MANAGER_IP:8080

---

## ⚙️ Configuration

### Docker Daemon Configuration

The playbook configures Docker with production settings (see `playbooks/install-docker.yml`):

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "metrics-addr": "0.0.0.0:9323",
  "experimental": false,
  "live-restore": true
}
```

### Stack Configuration

Modify `docker-stack.yml` to customize:
- Service replicas
- Resource limits (CPU/memory)
- Network configuration
- Volume mounts
- Health check parameters

### Ansible Configuration

`ansible/ansible.cfg` includes:
- SSH connection settings
- Privilege escalation
- Fact caching
- Performance optimizations

---

## 🔄 Deployment

### Automated CI/CD Deployment

The GitHub Actions workflows follow a branch-based deployment strategy:

1. **Dev Branch** (Push to dev):
   - Validates Ansible syntax
   - Runs ansible-lint and yamllint
   - Scans for security issues (Trivy, Gitleaks)
   - Validates Docker stack files
   - **Dry-run mode**: Tests without actual deployment
   - All tests run in check mode

2. **Main Branch** (Merge to main):
   - Runs all validation checks
   - **Production deployment**: Actual infrastructure changes
   - Deploys Swarm cluster
   - Deploys monitoring stack
   - Runs post-deployment tests
   - Verifies service health
   - Health check validation

3. **Jenkins Deployment** (Manual Only):
   - Navigate to: Actions → Jenkins CI/CD Stack Deployment
   - Click "Run workflow"
   - Requires manual trigger via `workflow_dispatch`
   - Includes pre-deployment validation
   - Post-deployment health checks

### Manual Deployment

```bash
# Deploy everything at once
cd ansible
ansible-playbook playbooks/install-docker.yml -i inventory.yml
ansible-playbook playbooks/swarm-setup.yml -i inventory.yml
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml

# Or use a master playbook (create site.yml):
ansible-playbook site.yml -i inventory.yml
```

### Workflow Dispatch

Trigger deployment manually via GitHub UI:

1. Go to **Actions** → **Docker Swarm CI/CD Pipeline**
2. Click **Run workflow**
3. Select environment (staging/production)
4. Optionally skip tests (emergency only)

---

## 📊 Monitoring

### Prometheus

Access Prometheus at `http://YOUR_MANAGER_IP:9090`

**Key Metrics**:
- Docker daemon metrics
- Swarm node health
- Service replica status
- Container resource usage

**Sample PromQL Queries**:

```promql
# Container CPU usage
rate(container_cpu_usage_seconds_total[5m])

# Service replica count
docker_swarm_service_replicas_running

# Node availability
docker_swarm_node_status
```

### Grafana

Access Grafana at `http://YOUR_MANAGER_IP:3000`

**Pre-configured Dashboards** (manual import required):
- Docker Swarm Overview
- Node Resource Usage
- Service Performance
- Application Metrics

**Add Prometheus Data Source**:
1. Configuration → Data Sources → Add data source
2. Select Prometheus
3. URL: `http://prometheus:9090`
4. Save & Test

### Swarm Visualizer

Access Visualizer at `http://YOUR_MANAGER_IP:8080`

Visual representation of:
- Node distribution
- Service placement
- Container status
- Real-time updates

---

## 🔧 Production Operations

### Backup Procedures

Automated backup of critical data runs daily at 2 AM UTC.

**Manual Backup**:
```bash
# Backup all critical data
ansible-playbook ansible/playbooks/backup.yml -i ansible/inventory.yml

# Backups include:
# - Swarm node configurations
# - Stack definitions
# - Prometheus data
# - Grafana data
# - Jenkins data
# - Monitoring configs

# Backup locations:
# /var/backups/swarm/TIMESTAMP/
# /var/backups/monitoring/TIMESTAMP/
# /var/backups/jenkins/TIMESTAMP/
```

**Backup Retention**: 30 days (configurable in playbook)

### Restore Procedures

Restore from backup in case of disaster or data corruption.

```bash
# Run restore playbook
ansible-playbook ansible/playbooks/restore.yml -i ansible/inventory.yml

# You will be prompted for:
# 1. Backup timestamp (format: YYYYMMDDTHHMMSS)
# 2. Confirmation (type "RESTORE")

# Example:
# Enter backup timestamp: 20251220T020000
# Type 'RESTORE' to confirm: RESTORE
```

**What gets restored**:
- Prometheus metrics data
- Grafana dashboards and configurations
- Jenkins jobs and build history
- Monitoring stack configurations

**Post-restore validation**:
- Prometheus health check
- Grafana accessibility
- Jenkins service status
- All services verification

### Health Checks

Comprehensive health monitoring for the entire infrastructure.

```bash
# Run full health check
ansible-playbook ansible/playbooks/health-check.yml -i ansible/inventory.yml

# Checks performed:
# ✓ Docker Swarm cluster status
# ✓ All nodes availability
# ✓ Service replicas status
# ✓ Disk usage warnings
# ✓ Memory usage warnings
# ✓ Node exporters health
# ✓ cAdvisor health
# ✓ Prometheus targets
# ✓ Grafana availability
# ✓ Jenkins status (if deployed)
```

**Schedule**: Run every 15 minutes in production (configure via cron or GitHub Actions)

**Health Check Components**:
- **Swarm Cluster**: Node status, availability, services
- **Worker Nodes**: Disk space (warn >80%), memory (warn >90%)
- **Exporters**: node-exporter and cAdvisor on all nodes
- **Monitoring**: Prometheus/Grafana health and data usage
- **Jenkins**: Service health endpoint validation

### Rollback Procedures

Revert deployments when issues are detected.

```bash
# Run rollback playbook
ansible-playbook ansible/playbooks/rollback.yml -i ansible/inventory.yml

# You will be prompted for:
# 1. Component to rollback: stack/monitoring/jenkins/all
# 2. Confirmation (type "ROLLBACK")

# Example:
# Which component to rollback? stack
# Type 'ROLLBACK' to confirm: ROLLBACK
```

**Rollback Process**:
1. Pre-rollback snapshot captured
2. Docker service rollback executed
3. Services stabilize (30-60 seconds)
4. Post-rollback validation
5. Report generated in `/var/log/rollback_TIMESTAMP.log`

**Supported Rollback Targets**:
- `stack`: All application stack services
- `monitoring`: Prometheus and Grafana stack
- `jenkins`: Jenkins CI/CD service
- `all`: Complete infrastructure rollback

### Cleanup Operations

Remove deployments while preserving cluster infrastructure.

```bash
# Cleanup via GitHub Actions
# Navigate to Actions → Cleanup Swarm Deployment
# Click "Run workflow"
# Enter "CLEANUP" to confirm

# Or run manually:
ansible-playbook ansible/playbooks/cleanup-swarm.yml -i ansible/inventory.yml
```

**Cleanup includes**:
- Docker stacks removal
- Container cleanup
- Volume pruning
- Network cleanup
- Image pruning
- Monitoring stack shutdown

**Cleanup preserves**:
- Swarm cluster structure
- Docker installation
- Node configurations
- SSH access

---

## 🧪 Testing

### Automated Tests

The project includes comprehensive testing:

```bash
# Syntax validation
ansible-playbook playbooks/*.yml --syntax-check -i inventory.yml

# Dry run (check mode)
ansible-playbook playbooks/install-docker.yml -i inventory.yml --check

# Full test suite
ansible-playbook playbooks/test-swarm.yml -i inventory.yml
```

### Test Coverage

- ✅ Ansible playbook syntax
- ✅ YAML linting
- ✅ Docker stack validation
- ✅ Security scanning (Trivy, Gitleaks)
- ✅ Swarm cluster health
- ✅ Service availability
- ✅ Endpoint health checks
- ✅ Prometheus/Grafana accessibility

### Local Testing

```bash
# Install testing tools
pip install ansible-lint yamllint molecule

# Run linters
yamllint ansible/ docker-stack.yml
ansible-lint ansible/playbooks/

# Validate Docker stack
docker compose -f docker-stack.yml config
```

---

## 🐛 Troubleshooting

### Issue 1: Ansible Connection Failed

**Error**: `Failed to connect to the host via ssh`

**Solution**:
```bash
# Verify SSH access
ssh -i ~/.ssh/your_key.pem ubuntu@YOUR_SERVER_IP

# Check SSH key permissions
chmod 600 ~/.ssh/your_key.pem

# Add to known hosts
ssh-keyscan -H YOUR_SERVER_IP >> ~/.ssh/known_hosts
```

### Issue 2: Swarm Initialization Failed

**Error**: `This node is already part of a swarm`

**Solution**:
```bash
# On affected node
docker swarm leave --force

# Re-run swarm setup playbook
ansible-playbook playbooks/swarm-setup.yml -i inventory.yml
```

### Issue 3: Service Not Starting

**Error**: Service replicas show `0/3`

**Solution**:
```bash
# Check service logs
docker service logs SERVICE_NAME

# Check service details
docker service ps SERVICE_NAME --no-trunc

# Common issues:
# - Missing secrets: create them first
# - Resource constraints: adjust limits in docker-stack.yml
# - Image pull failures: check image name/registry
```

### Issue 4: Port Already in Use

**Error**: `port is already allocated`

**Solution**:
```bash
# Find process using port
sudo lsof -i :80

# Stop conflicting service
sudo systemctl stop apache2  # or nginx, etc.

# Redeploy stack
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml
```

### Issue 5: Secrets Creation Failed

**Error**: `secret already exists`

**Solution**:
```bash
# Remove existing secrets
ansible swarm_manager -i inventory.yml -b -m shell -a "
  docker secret rm api_secret_key || true
  docker secret rm grafana_admin_password || true
"

# Recreate secrets
ansible swarm_manager -i inventory.yml -b -m shell -a "
  echo 'NEW_SECRET' | docker secret create api_secret_key -
"
```

### Issue 6: GitHub Actions Deployment Failed

**Check**:
1. Verify all required secrets are set in GitHub
2. Check workflow logs for specific error
3. Ensure SSH key has correct permissions
4. Verify firewall allows GitHub Actions IPs

---

## 🔒 Security

### Security Features

1. **Secret Management**
   - Docker Secrets for sensitive data
   - No hardcoded credentials
   - GitHub Secrets for CI/CD

2. **Network Security**
   - Overlay networks with encryption
   - Service isolation
   - No unnecessary port exposure

3. **Container Security**
   - Non-root user execution where possible
   - Resource limits to prevent DoS
   - Read-only root filesystems (where applicable)

4. **CI/CD Security**
   - Trivy security scanning
   - Gitleaks secret detection
   - SARIF upload to GitHub Security
   - Dependency vulnerability checks

### Security Best Practices

```bash
# Rotate secrets regularly
docker secret create api_secret_key_v2 - < new_secret.txt
docker service update --secret-rm api_secret_key --secret-add api_secret_key_v2 SERVICE_NAME

# Update Docker regularly
ansible-playbook playbooks/install-docker.yml -i inventory.yml

# Review security scan results
# Check GitHub Security tab after each push

# Enable Docker Content Trust (optional)
export DOCKER_CONTENT_TRUST=1
```

### Firewall Rules

```bash
# Manager Node
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw allow 2377/tcp    # Swarm management
sudo ufw allow 7946/tcp    # Swarm communication
sudo ufw allow 7946/udp    # Swarm communication
sudo ufw allow 4789/udp    # Overlay network
sudo ufw allow 9090/tcp    # Prometheus
sudo ufw allow 3000/tcp    # Grafana
sudo ufw allow 8080/tcp    # Visualizer
sudo ufw enable

# Worker Nodes
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 7946/tcp    # Swarm communication
sudo ufw allow 7946/udp    # Swarm communication
sudo ufw allow 4789/udp    # Overlay network
sudo ufw enable
```

---

## 📈 Performance Optimization

### Resource Allocation

Adjust in `docker-stack.yml`:

```yaml
deploy:
  resources:
    limits:
      cpus: '1.0'
      memory: 512M
    reservations:
      cpus: '0.5'
      memory: 256M
```

### Caching Strategy

Redis is included for application caching. Configure your services to use:
- **Host**: `redis`
- **Port**: `6379`
- **Connection**: Within `backend` network

### Docker Daemon Optimization

Modify `ansible/playbooks/install-docker.yml` daemon.json:

```json
{
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 5,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  }
}
```

---

## 🔄 CI/CD Pipeline Details

### Pipeline Stages

```
Dev Branch Flow (Dry-Run Testing):
┌─────────────┐
│  Validate   │ ── Syntax, lint, YAML validation
└──────┬──────┘
       │
┌──────▼──────┐
│  Security   │ ── Trivy, Gitleaks, credential scan
└──────┬──────┘
       │
┌──────▼──────┐
│  Dry-Run    │ ── Ansible check mode (no changes)
└──────┬──────┘
       │
┌──────▼──────┐
│   Report    │ ── Test results and validation
└─────────────┘

Main Branch Flow (Production Deployment):
┌─────────────┐
│  Validate   │ ── All validation checks
└──────┬──────┘
       │
┌──────▼──────┐
│Deploy Swarm │ ── Swarm cluster setup
└──────┬──────┘
       │
┌──────▼──────┐
│Deploy Stack │ ── Monitoring & services
└──────┬──────┘
       │
┌──────▼──────┐
│   Verify    │ ── Health checks, validation
└─────────────┘

Manual Jenkins Flow:
┌─────────────┐
│  Validate   │ ── Pre-deployment checks
└──────┬──────┘
       │
┌──────▼──────┐
│   Deploy    │ ── Jenkins stack deployment
└──────┬──────┘
       │
┌──────▼──────┐
│   Verify    │ ── Jenkins health validation
└─────────────┘
```

### Workflow Jobs

| Job | Duration | Triggers | Purpose |
|-----|----------|----------|---------|
| **CI Validation** | ~3min | Push to dev/main | Syntax, lint, security scans |
| **Swarm Setup (Dry)** | ~5min | Push to dev | Test swarm deployment |
| **Swarm Setup (Prod)** | ~10min | Push to main | Deploy swarm cluster |
| **Monitoring (Dry)** | ~3min | Push to dev | Test monitoring stack |
| **Monitoring (Prod)** | ~8min | Push to main | Deploy monitoring |
| **Jenkins Deploy** | ~5min | Manual trigger | Deploy Jenkins (manual only) |
| **Cleanup** | ~3min | Manual trigger | Remove deployments |
| **Health Checks** | ~2min | After deployment | Validate all services |

---

## 🛠️ Development

### Project Structure

```
redLUIT_Dec2025_DockerSwarm/
├── .github/
│   └── workflows/
│       ├── ci-validation.yml        # Validation pipeline (dev/main)
│       ├── swarm-setup.yml          # Swarm deployment (dev/main)
│       ├── monitoring-deploy.yml    # Monitoring deployment (dev/main)
│       ├── jenkins-deploy.yml       # Jenkins deployment (manual)
│       └── cleanup-swarm.yml        # Cleanup workflow (manual)
├── ansible/
│   ├── ansible.cfg                  # Ansible configuration
│   ├── requirements.yml             # Galaxy collections
│   ├── inventory.yml                # Static inventory
│   ├── aws_ec2.yml                  # EC2 dynamic inventory
│   └── playbooks/
│       ├── install-docker.yml       # Docker installation
│       ├── swarm-setup.yml          # Swarm initialization
│       ├── deploy-stack.yml         # Stack deployment
│       ├── setup-monitoring.yml     # Monitoring stack setup
│       ├── setup-jenkins.yml        # Jenkins deployment
│       ├── setup-exporters.yml      # Metrics exporters
│       ├── test-swarm.yml           # Swarm tests
│       ├── test-exporters.yml       # Exporter tests
│       ├── backup.yml               # Backup playbook
│       ├── restore.yml              # Restore playbook
│       ├── health-check.yml         # Health monitoring
│       ├── rollback.yml             # Rollback procedures
│       ├── cleanup-swarm.yml        # Cleanup deployments
│       └── debug-*.yml              # Debug playbooks
├── configs/
│   ├── nginx.conf                   # Nginx configuration
│   ├── prometheus.yml.j2            # Prometheus template (dynamic)
│   └── grafana/                     # Grafana configs
├── docker-stack.yml                 # Application stack
├── jenkins-stack.yml                # Jenkins stack
├── requirements.txt                 # Python dependencies (pinned)
├── PRODUCTION_STANDARDS.md          # Production best practices
├── CHANGELOG.md                     # Version history
├── .ansible-lint                    # Ansible linting rules
├── .yamllint                        # YAML linting rules
└── README.md                        # This file
```

### Adding New Services

1. **Update docker-stack.yml**:

```yaml
services:
  newservice:
    image: your-image:tag
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.role == worker
    networks:
      - backend
```

2. **Update deploy-stack.yml** if service needs initialization

3. **Update test-swarm.yml** to add health checks

4. **Test locally**:

```bash
docker compose -f docker-stack.yml config
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml --check
```

### Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Run tests (`ansible-lint`, `yamllint`)
5. Commit changes (`git commit -m 'Add amazing feature'`)
6. Push to branch (`git push origin feature/amazing-feature`)
7. Open a Pull Request

---

## 📚 Additional Resources

### Documentation
- [Docker Swarm Official Docs](https://docs.docker.com/engine/swarm/)
- [Ansible Documentation](https://docs.ansible.com/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)

### Related Projects
- [Docker Swarm Rocks](https://dockerswarm.rocks/)
- [Ansible Docker Modules](https://docs.ansible.com/ansible/latest/collections/community/docker/)

### Useful Commands

```bash
# Scale service
docker service scale mystack_web=5

# Update service
docker service update --image nginx:latest mystack_web

# View service logs
docker service logs -f mystack_web

# Drain node for maintenance
docker node update --availability drain worker1

# Promote worker to manager
docker node promote worker1

# Remove stack
docker stack rm mystack

# Leave swarm
docker swarm leave --force
```

---

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

---

## 👥 Authors

- **DevOps Team** - *Initial work*

---

## 📋 Production Standards

This project follows comprehensive production best practices documented in `PRODUCTION_STANDARDS.md`.

### Key Standards

**Version Management**:
- All Python dependencies pinned in `requirements.txt`
- Docker images use specific version tags (no `latest`)
- Semantic versioning tracked in `CHANGELOG.md`
- Git tags for release versions

**Backup & Disaster Recovery**:
- Automated daily backups (2 AM UTC)
- 30-day backup retention policy
- Tested restore procedures
- Backup verification in health checks

**Operational Excellence**:
- Health checks every 15 minutes
- Rollback procedures for all components
- Pre-deployment validation
- Post-deployment verification
- Comprehensive logging

**CI/CD Standards**:
- Dev branch for testing (dry-run)
- Main branch for production
- Manual approval for Jenkins
- Security scanning on all changes
- Automated health checks post-deployment

**Documentation**:
- Production standards guide
- Comprehensive README
- Inline playbook documentation
- Version history (CHANGELOG)
- Troubleshooting guides

### Compliance Status

Current implementation: **Phase 1 (Critical) - Complete**

✅ Version pinning for all dependencies
✅ Backup and restore procedures
✅ Health monitoring playbook
✅ Rollback capabilities
✅ Production documentation
✅ CI/CD branch strategy
✅ Security scanning

See `PRODUCTION_STANDARDS.md` for Phase 2-4 implementation roadmap.

---

## 🙏 Acknowledgments

- Docker Swarm community
- Ansible community
- RedLUIT training program participants

---

## 📞 Support

For issues and questions:
- **GitHub Issues**: [Create an issue](https://github.com/YOUR_USERNAME/redLUIT_Dec2025_DockerSwarm/issues)
- **Documentation**: This README
- **Email**: devops@your-org.com

---

**Last Updated**: December 2025
**Version**: 1.0.0
**Status**: Production Ready ✅

---

## 🔗 Quick Links

- **[Production Standards](PRODUCTION_STANDARDS.md)** - Best practices and compliance
- **[CHANGELOG](CHANGELOG.md)** - Version history and updates
- **[Requirements](requirements.txt)** - Python dependencies
