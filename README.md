# WordPress + MySQL Stack on Kubernetes

A production-style deployment of WordPress with MySQL on a kind (Kubernetes in Docker) cluster. This project covers the majority of core Kubernetes concepts including StatefulSets, Deployments, Persistent Volumes, Secrets, ConfigMaps, Ingress, HPA, ResourceQuota, Probes, and Taints & Tolerations.

---

## Concepts Used

| Concept | Resource | Role |
|---|---|---|
| Namespace | `namespace.yml` | Isolates all resources under `wordpress` namespace |
| Secret | `secrets.yml` | Stores MySQL credentials securely (base64 encoded) |
| ConfigMap | `config-map.yml` | Stores non-sensitive WordPress config (DB host, DB name) |
| PersistentVolume | `pv.yml` | Provisions physical storage on the host for MySQL and WordPress |
| PersistentVolumeClaim | `pvc.yml` | Requests storage from PVs for use by pods |
| StatefulSet | `mysql-stateful.yml` | Runs MySQL with stable pod identity and persistent storage |
| Deployment | `wordpress-deployment.yml` | Runs WordPress as a stateless, scalable web layer |
| ClusterIP Service | `mysql-service.yml` | Exposes MySQL internally — only WordPress can reach it |
| ClusterIP Service | `wordpress-service.yml` | Exposes WordPress internally to the Ingress Controller |
| Ingress | `ingress.yml` | Routes external HTTP traffic to WordPress service |
| ResourceQuota | `resource-quota.yml` | Caps total CPU and memory usage in the namespace |
| Liveness Probe | Deployment/StatefulSet | Restarts container if it becomes unhealthy |
| Readiness Probe | Deployment/StatefulSet | Holds traffic until container is fully ready |
| Resource Requests/Limits | Deployment/StatefulSet | Reserves and caps CPU/memory per container |
| Taints & Tolerations | `wordpress-deployment.yml` | Controls which nodes pods are allowed to schedule on |
| HPA | `hpa.yml` | Auto-scales WordPress pods based on CPU usage |

---

## Prerequisites

### Install Docker
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

### Install kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### Install kind
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

---

## Cluster Setup

### config.yml — kind cluster configuration

The config defines 1 control-plane + 2 worker nodes. The control-plane has port mappings so external traffic can reach the Ingress Controller. The `ingress-ready=true` label is required so the Ingress Controller can be forced to schedule on this node.

Key fields:
- `extraPortMappings` — maps container ports to host ports (port 80 inside kind → port 880 on host, 443 → 4443)
- `node-labels: ingress-ready=true` — marks control-plane as the ingress node
- `role: worker` — adds worker nodes for workload scheduling

### Create the cluster
```bash
kind create cluster --name=k8s-project --config=config.yml
```

### Verify nodes are Ready
```bash
kubectl get nodes
```

Expected: 1 control-plane + 2 workers, all in `Ready` status.

---

## Install Nginx Ingress Controller

The Ingress Controller is the single entry point for all external HTTP/HTTPS traffic into the cluster. It reads Ingress rules and routes requests to the correct backend service — like a traffic cop inside the cluster.

### Install using the official kind manifest
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### Wait for it to be ready
```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Force Ingress Controller onto control-plane node

**Critical step for kind clusters.** The Ingress Controller must run on the control-plane node because that is the only node with port mappings to the host. By default Kubernetes may schedule it on a worker node, which breaks external access.

```bash
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/nodeSelector",
    "value": {
      "ingress-ready": "true"
    }
  }
]'
```

### Verify it moved to control-plane
```bash
kubectl get pods -n ingress-nginx -o wide
```

The `NODE` column must show `k8s-project-control-plane`, not a worker node.

---

## Install Metrics Server (required for HPA)

The Metrics Server collects CPU and memory usage from nodes and pods. HPA needs this data to make scaling decisions.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Patch for kind (disable TLS verification)
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'
```

### Verify metrics are flowing
```bash
kubectl top nodes
kubectl top pods -n wordpress
```

---

## Encoding Secrets

Secret values must be base64 encoded before putting them in `secrets.yml`. Never store plain text passwords in Kubernetes Secrets.

```bash
echo -n 'your-password-here' | base64
echo -n 'your-username-here' | base64
echo -n 'your-database-name' | base64
```

Paste the encoded output as values in `secrets.yml`.

---

## Apply Order — This Matters

Kubernetes resources have dependencies. Apply in this exact order:

```bash
# 1. Create the namespace first — everything else goes inside it
kubectl apply -f namespace.yml

# 2. ResourceQuota — caps namespace resource usage
kubectl apply -f resource-quota.yml

# 3. Secrets — must exist before pods that reference them start
kubectl apply -f secrets.yml

# 4. ConfigMap — non-sensitive config for WordPress
kubectl apply -f config-map.yml

# 5. PersistentVolumes — storage must exist before claims
kubectl apply -f pv.yml

# 6. PersistentVolumeClaims — request storage from PVs
kubectl apply -f pvc.yml

# 7. MySQL StatefulSet — database must start before WordPress
kubectl apply -f mysql-stateful.yml

# 8. MySQL Service — WordPress needs this to reach the DB
kubectl apply -f mysql-service.yml

# 9. WordPress Deployment
kubectl apply -f wordpress-deployment.yml

# 10. WordPress Service
kubectl apply -f wordpress-service.yml

# 11. Ingress — routes external traffic to WordPress service
kubectl apply -f ingress.yml

# 12. HPA — auto-scaling, applied last after deployment is stable
kubectl apply -f hpa.yml
```

---

## Taint the Node (for Taints & Tolerations demo)

A taint prevents pods from scheduling on a node unless they explicitly tolerate it. This is used to reserve nodes for specific workloads.

```bash
kubectl taint nodes k8s-project-control-plane environment=production:NoSchedule
```

The WordPress Deployment has a matching toleration in its spec so it can still schedule despite the taint. MySQL StatefulSet also needs the same toleration if it needs to run there.

### Remove taint (if needed)
```bash
kubectl taint nodes k8s-project-control-plane environment=production:NoSchedule-
```

---

## Verify Everything is Running

```bash
# Check all pods are Running (not CrashLoopBackOff or Pending)
kubectl get pods -n wordpress

# Check PVs are Bound
kubectl get pv

# Check PVCs are Bound
kubectl get pvc -n wordpress

# Check services exist with correct ports
kubectl get svc -n wordpress

# Check Ingress has an address
kubectl get ingress -n wordpress

# Check HPA is active
kubectl get hpa -n wordpress

# Check ResourceQuota usage
kubectl describe resourcequota -n wordpress
```

---

## Access WordPress

### Test from server (confirms ingress is working)
```bash
curl -v http://127.0.0.1:880 -H "Host: wordpress.local"
```

Expected response: `HTTP/1.1 302` redirecting to `/wp-admin/install.php` — this confirms WordPress is alive and connected to MySQL.

### Add to local machine hosts file

**Mac/Linux:**
```bash
sudo sh -c 'echo "YOUR_SERVER_IP wordpress.local" >> /etc/hosts'
```

**Windows** — open `C:\Windows\System32\drivers\etc\hosts` as Administrator and add:
```
YOUR_SERVER_IP wordpress.local
```

### Configure server Nginx as reverse proxy

If your server already runs Nginx on port 80, add a new site that proxies `wordpress.local` to kind's port 880:

```bash
sudo nano /etc/nginx/sites-available/wordpress
```

```nginx
server {
    listen 80;
    server_name wordpress.local;

    location / {
        proxy_pass http://127.0.0.1:880;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Now visit `http://wordpress.local` in your browser and complete the WordPress installation.

---

## Useful Debugging Commands

### Check pod logs
```bash
kubectl logs <pod-name> -n wordpress
kubectl logs <pod-name> -n wordpress --previous   # logs from crashed container
```

### Describe a pod (shows events and errors)
```bash
kubectl describe pod <pod-name> -n wordpress
```

### Exec into a pod
```bash
kubectl exec -it <pod-name> -n wordpress -- bash
```

### Check HPA status
```bash
kubectl describe hpa -n wordpress
```

### Check ingress controller logs
```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=50
```

### Check which node a pod is on
```bash
kubectl get pods -n wordpress -o wide
kubectl get pods -n ingress-nginx -o wide
```

### Check resource usage
```bash
kubectl top pods -n wordpress
kubectl top nodes
```

### Check events (great for diagnosing failures)
```bash
kubectl get events -n wordpress --sort-by='.lastTimestamp'
```

---

## Common Issues & Solutions

### Issue: MySQL pod stuck in Pending
**Cause:** ResourceQuota is blocking it — total memory limits across all pods exceed the quota.
**Fix:** Either increase `limits.memory` in `resource-quota.yml` or reduce MySQL's memory limit in `mysql-stateful.yml`. Check with:
```bash
kubectl get events -n wordpress --sort-by='.lastTimestamp' | grep quota
```

### Issue: WordPress pod in CrashLoopBackOff with HTTP 500
**Cause:** WordPress can't connect to MySQL. Either MySQL isn't running yet, or service name/credentials don't match.
**Fix:**
```bash
kubectl get pods -n wordpress           # confirm mysql pod is Running
kubectl get svc -n wordpress            # confirm mysql-service exists
kubectl logs <mysql-pod> -n wordpress   # check MySQL started cleanly
```
The `WORDPRESS_DB_HOST` value must exactly match the MySQL service name.

### Issue: curl returns "Connection reset by peer" on port 880
**Cause:** Ingress Controller is running on a worker node, not the control-plane node that has the port mapping.
**Fix:** Patch the ingress controller deployment to force it onto the control-plane node using the `ingress-ready=true` nodeSelector (see Install section above).

### Issue: 502 Bad Gateway from server Nginx
**Cause:** Server Nginx is proxying to port 880 but the Ingress Controller isn't there yet, or is on the wrong node.
**Fix:** Confirm ingress controller is on control-plane, then test with `curl -v http://127.0.0.1:880` before reloading Nginx.

### Issue: 404 from Nginx on wordpress.local
**Cause:** Request is hitting server Nginx but no server block matches `wordpress.local`.
**Fix:** Create `/etc/nginx/sites-available/wordpress` with the proxy config above and enable it.

### Issue: HPA shows `<unknown>` for CPU targets
**Cause:** Metrics Server is not running or TLS verification is blocking it on kind.
**Fix:** Patch metrics-server with `--kubelet-insecure-tls` flag (see Metrics Server section above).

### Issue: PVC stuck in Pending
**Cause:** No PV matches the PVC's `storageClassName` or `accessModes`.
**Fix:** Make sure `storageClassName: manual` and `accessModes: ReadWriteOnce` match exactly between your PV and PVC.

### Issue: kind cluster fails with "Joining worker nodes" error on AWS EC2
**Cause:** EC2's Docker networking blocks inter-container communication needed for worker nodes to join.
**Fix:** Use single-node cluster config (control-plane only). All Kubernetes concepts work the same on single node.

---

## Delete Everything

### Delete all resources in namespace
```bash
kubectl delete namespace wordpress
```

### Delete the cluster entirely
```bash
kind delete cluster --name=k8s-project
```

---

## Architecture Overview

```
Browser (wordpress.local)
        ↓
Server Nginx (port 80) — reverse proxy
        ↓
kind control-plane container (port 880)
        ↓
Nginx Ingress Controller pod
        ↓
wordpress-service (ClusterIP :80)
        ↓
WordPress pod (Deployment)
        ↓
mysql-service (ClusterIP :3306)
        ↓
MySQL pod (StatefulSet)
        ↓
PersistentVolume (hostPath storage)
```

---

## Author

[saim-devops](https://github.com/saim-devops)