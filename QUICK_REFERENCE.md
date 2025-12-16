# Docker Swarm Quick Reference

## 🚀 One-Command Deployment

```bash
cd ansible && ansible-playbook site.yml -i inventory.yml
```

---

## 📝 Common Commands

### Deployment

```bash
# Full deployment
ansible-playbook site.yml -i inventory.yml

# Individual phases
ansible-playbook playbooks/install-docker.yml -i inventory.yml
ansible-playbook playbooks/swarm-setup.yml -i inventory.yml
ansible-playbook playbooks/create-secrets.yml -i inventory.yml
ansible-playbook playbooks/deploy-stack.yml -i inventory.yml

# Update stack (zero downtime)
ansible-playbook playbooks/update-stack.yml -i inventory.yml

# Teardown
ansible-playbook playbooks/teardown.yml -i inventory.yml
```

### Docker Swarm

```bash
# View nodes
docker node ls

# View services
docker service ls
docker stack services mystack

# View tasks/containers
docker stack ps mystack
docker service ps mystack_web

# Service logs
docker service logs -f mystack_web

# Scale service
docker service scale mystack_web=5

# Update service
docker service update --image nginx:1.26 mystack_web

# Rollback service
docker service rollback mystack_web
```

### Monitoring

```bash
# Container stats
docker stats

# Node resources
docker node inspect manager --pretty

# Service health
docker service inspect mystack_web

# Network list
docker network ls

# Volume list
docker volume ls
```

---

## 🔧 Troubleshooting

### Quick Diagnostics

```bash
# Check cluster health
docker node ls
docker service ls

# View service errors
docker service ps mystack_web --no-trunc
docker service logs mystack_web --tail 100

# Check network
docker network inspect mystack_backend

# Check secrets
docker secret ls
```

### Common Fixes

```bash
# Restart service
docker service update --force mystack_web

# Remove and redeploy service
docker service rm mystack_web
docker stack deploy -c docker-stack.yml mystack

# Force recreate all services
docker stack rm mystack
sleep 30
docker stack deploy -c docker-stack.yml mystack

# Node maintenance
docker node update --availability drain worker1
# ... perform maintenance ...
docker node update --availability active worker1
```

---

## 🔐 Security

### Secrets Management

```bash
# Create secret
echo "my-secret" | docker secret create api_key -

# List secrets
docker secret ls

# Remove secret
docker secret rm api_key

# Rotate secret (requires service update)
echo "new-secret" | docker secret create api_key_v2 -
docker service update --secret-rm api_key --secret-add api_key_v2 mystack_api
docker secret rm api_key
```

### Firewall (UFW)

```bash
# Enable firewall
sudo ufw enable

# View rules
sudo ufw status

# Add rule
sudo ufw allow 80/tcp

# Delete rule
sudo ufw delete allow 80/tcp
```

---

## 📊 Monitoring URLs

```bash
# Replace MANAGER_IP with your manager node IP

# Web Application
http://MANAGER_IP

# Prometheus
http://MANAGER_IP:9090

# Grafana
http://MANAGER_IP:3000

# Swarm Visualizer
http://MANAGER_IP:8080
```

---

## 🔄 Update Workflow

1. **Update docker-stack.yml** with new versions
2. **Test locally**: `docker compose -f docker-stack.yml config`
3. **Deploy update**: `ansible-playbook playbooks/update-stack.yml -i inventory.yml`
4. **Monitor**: `watch docker service ls`
5. **Rollback if needed**: `docker service rollback SERVICE_NAME`

---

## 📦 Backup

```bash
# Backup Grafana data
docker run --rm \
  -v mystack_grafana_data:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/grafana-$(date +%Y%m%d).tar.gz /data

# Backup Prometheus data
docker run --rm \
  -v mystack_prometheus_data:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/prometheus-$(date +%Y%m%d).tar.gz /data

# Backup all volumes
for vol in $(docker volume ls -q | grep mystack); do
  docker run --rm \
    -v $vol:/data \
    -v $(pwd):/backup \
    ubuntu tar czf /backup/${vol}-$(date +%Y%m%d).tar.gz /data
done
```

---

## 🚨 Emergency Commands

```bash
# Stop all services (emergency)
docker service scale $(docker service ls -q --format "{{ '{{' }}.Name{{ '}}' }}=0")

# Remove stuck tasks
docker service update --force mystack_web

# Leave swarm (worker)
docker swarm leave --force

# Leave swarm (manager - destroys cluster!)
docker swarm leave --force

# Prune everything (careful!)
docker system prune -af --volumes
```

---

## 📱 Useful Aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# Docker Swarm aliases
alias ds='docker stack'
alias dss='docker stack services'
alias dsp='docker stack ps'
alias dsvc='docker service'
alias dsl='docker service logs'
alias dsc='docker service scale'
alias dnl='docker node ls'
alias dps='docker ps'
alias dimg='docker images'

# Ansible aliases
alias ap='ansible-playbook'
alias ai='ansible-inventory'
alias ag='ansible-galaxy'
```

---

## 📞 Quick Help

**Issue:** Service won't start
→ `docker service logs SERVICE_NAME`

**Issue:** Node unreachable
→ `docker node inspect NODE_NAME`

**Issue:** Out of resources
→ `docker system df` & `docker system prune`

**Issue:** Network problems
→ `docker network inspect NETWORK_NAME`

**Issue:** Stack won't deploy
→ `docker compose -f docker-stack.yml config`

---

## 🎯 Performance Tips

```bash
# Check resource usage
docker stats

# Limit service resources
docker service update \
  --limit-cpu 0.5 \
  --limit-memory 512M \
  mystack_web

# Reserve resources
docker service update \
  --reserve-cpu 0.25 \
  --reserve-memory 256M \
  mystack_web

# Set update parallelism
docker service update \
  --update-parallelism 2 \
  --update-delay 10s \
  mystack_web
```

---

## 📖 Documentation Links

- Full Guide: `README.md`
- Deployment Steps: `DEPLOYMENT.md`
- Production Checklist: `PRODUCTION_CHECKLIST.md`
- Workflow Guide: `WORKFLOW_GUIDE.md` (if exists)

---

**Last Updated:** December 2024
