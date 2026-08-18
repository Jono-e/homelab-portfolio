# ⚙️ DevOps / Kubernetes Lab

A hands-on DevOps lab covering containerisation with Docker and container orchestration with Kubernetes (k3s). Built a real multi-container application from scratch, containerised it, deployed it to a Kubernetes cluster, and practised production workflows including scaling, self-healing, rolling updates and rollbacks.

---

## Environment

| Tool | Details |
|---|---|
| **Docker Desktop** | v29.7.2 on Windows 11 (WSL2 backend) |
| **Kubernetes** | k3s v1.36.3 on Ubuntu Server 26.04 LTS (VirtualBox VM) |
| **Container Runtime** | containerd v2.3.2 (k3s built-in) |
| **App Stack** | Python Flask + Redis |

---

## Phase 1: Docker Fundamentals

### What I Built

A Python Flask web app with a Redis-backed visit counter — a realistic two-service application that demonstrates real Docker concepts rather than just running pre-built images.

**App stack:**
- `web` — Flask app serving a visit counter
- `redis` — Redis instance persisting the counter across restarts

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

**Key design decisions:**
- `python:3.12-slim` — minimal base image, no unnecessary packages (smaller attack surface, faster pulls)
- Dependencies copied and installed **before** app code — Docker layer caching means if only `app.py` changes, pip install is skipped entirely on rebuild, saving significant time
- `EXPOSE 5000` documents the port; actual publishing happens at runtime with `-p`

### Docker Compose

```yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - REDIS_HOST=redis
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    restart: unless-stopped
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

volumes:
  redis-data:
```

**Concepts demonstrated:**

- **Inter-container DNS** — Flask reaches Redis via hostname `redis`, not an IP address. Docker Compose creates a shared network automatically and provides DNS resolution between services. This mirrors how Kubernetes service discovery works.
- **`depends_on`** — Redis starts before the web container
- **Named volumes** — `redis-data` persists data outside the container lifecycle. Confirmed by running `docker compose down` and `docker compose up -d` — visit counter continued from where it left off rather than resetting to 0.
- **`restart: unless-stopped`** — containers survive Docker Desktop restarts automatically

### Key Learning — Slim Images

Discovered that `python:3.12-slim` intentionally omits diagnostic tools (`ps`, `ping`, `curl`). When exec'd into the container:

```bash
docker exec -it homelab-app /bin/bash
ps aux  # bash: ps: command not found
```

In practice: use host-side commands instead of modifying the running container:

```bash
docker top homelab-app      # Process list from host
docker stats homelab-app    # Resource usage
docker logs -f homelab-app  # Live log stream
```

This is actually better practice — it doesn't modify the running container and works regardless of what tools are installed inside.

---

## Phase 2: Kubernetes with k3s

### Cluster Setup

```bash
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes
```

```
NAME               STATUS   ROLES           AGE   VERSION
ubuntutestserver   Ready    control-plane   9s    v1.36.3+k3s1
```

k3s installs as a systemd service, sets up kubectl, configures containerd, and starts a fully functional Kubernetes cluster in under 60 seconds. It ships with CoreDNS, Traefik ingress, local-path-provisioner, and metrics-server pre-installed.

### Manifests

**Deployment** (`deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homelab-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: homelab-app
  template:
    metadata:
      labels:
        app: homelab-app
    spec:
      containers:
      - name: homelab-app
        image: docker.io/library/homelab-app:v1
        imagePullPolicy: Never
        ports:
        - containerPort: 5000
        env:
        - name: REDIS_HOST
          value: redis-service
```

**Redis** (`redis.yaml`) — Deployment + ClusterIP Service:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

**Service** (`service.yaml`) — NodePort for external access:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: homelab-app-service
spec:
  type: NodePort
  selector:
    app: homelab-app
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30080
```

**Service types explained:**
- `ClusterIP` (Redis) — internal only, no external access. Databases should never be directly exposed externally.
- `NodePort` (Flask app) — exposes on a port on the node's IP, reachable from outside the cluster. Used here for simplicity; in production you'd use an Ingress instead.

### Deployment

```bash
sudo kubectl apply -f ~/k8s-app/
sudo kubectl get all
```

```
NAME                               READY   STATUS    RESTARTS   AGE
pod/homelab-app-55664d58bd-ts87c   1/1     Running   0          2m13s
pod/homelab-app-55664d58bd-vnbdh   1/1     Running   0          2m13s
pod/redis-88f6ffbc8-8ghgp          1/1     Running   0          2m13s

NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/homelab-app-service   NodePort    10.43.190.18   <none>        5000:30080/TCP   2m14s
service/redis-service         ClusterIP   10.43.30.220   <none>        6379/TCP         2m14s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/homelab-app   2/2     2            2           2m14s
deployment.apps/redis         1/1     1            1           2m14s
```

App accessible at `http://<node-ip>:30080`.

### Scaling

```bash
# Scale up
sudo kubectl scale deployment homelab-app --replicas=5

# Scale back down
sudo kubectl scale deployment homelab-app --replicas=2
```

Watched pods appear and terminate in real time via `kubectl get pods -w`. Kubernetes maintained availability throughout — no downtime during either scale operation.

### Self-Healing

```bash
sudo kubectl delete pod homelab-app-55664d58bd-ts87c
sudo kubectl get pods -w
```

Kubernetes created a replacement pod (`zb7hz`) **before** the deleted pod finished terminating — maintaining 2 running replicas at all times without any manual intervention. This is declarative state in action: you declared "I want 2 replicas" and Kubernetes continuously enforces that.

---

## Phase 3: Rolling Updates & Rollbacks

### Zero-Downtime Deployment

Updated the Flask app (added version number, new response content) and built `homelab-app:v2`. Transferred to k3s containerd and deployed:

```bash
sudo kubectl set image deployment/homelab-app homelab-app=docker.io/library/homelab-app:v2
sudo kubectl rollout status deployment/homelab-app
```

```
deployment "homelab-app" successfully rolled out
```

Kubernetes replaced pods one at a time — old pod terminated only after new pod was `Running` and ready. No downtime, no manual intervention.

### Instant Rollback

```bash
sudo kubectl rollout undo deployment/homelab-app
sudo kubectl describe deployment homelab-app | grep Image
# Image: docker.io/library/homelab-app:v1
```

One command reverted the entire deployment to the previous version. Kubernetes maintains a rollout history for exactly this purpose.

Rolled forward again to v2:

```bash
sudo kubectl set image deployment/homelab-app homelab-app=docker.io/library/homelab-app:v2
sudo kubectl rollout status deployment/homelab-app
```

---

## Complete Workflow Summary

```
Code change (app.py)
    ↓
docker build -t homelab-app:v2 .     # Build new image
    ↓
docker save → scp → k3s ctr import   # Transfer to cluster
    ↓
kubectl set image                     # Trigger rolling update
    ↓
kubectl rollout status                # Confirm successful rollout
    ↓
kubectl rollout undo                  # Rollback if needed
```

---

## Interview Talking Points

| Question | Answer |
|---|---|
| "What's the difference between a container and a VM?" | Containers share the host kernel and are isolated at the process level — much lighter than VMs which emulate full hardware. A Docker container starts in milliseconds; a VM takes minutes. |
| "Explain Docker layer caching" | Each Dockerfile instruction creates a layer. If a layer's inputs haven't changed, Docker reuses the cached version. Putting `COPY requirements.txt` and `RUN pip install` before `COPY app.py` means dependency installation is skipped on rebuilds when only code changes. |
| "What's the difference between NodePort and ClusterIP?" | ClusterIP is internal only — other pods can reach it but nothing outside the cluster can. NodePort exposes the service on a port on every node, reachable externally. In production you'd typically use Ingress instead of NodePort. |
| "How does Kubernetes handle a failing pod?" | The ReplicaSet controller continuously reconciles actual state against desired state. When a pod dies, it creates a replacement immediately — before the old one even finishes terminating. |
| "What's a rolling update?" | Kubernetes replaces pods one at a time, waiting for each new pod to be ready before terminating the old one. This gives zero-downtime deployments — the app stays available throughout. |
| "Have you done a rollback?" | Yes — `kubectl rollout undo` reverts to the previous ReplicaSet. Kubernetes keeps rollout history specifically for this. I practised doing this in the lab and confirmed the image tag switched back to v1. |

---

## Skills Demonstrated

`Docker` `Dockerfile` `Docker Compose` `Container networking` `Docker volumes` `Kubernetes` `k3s` `kubectl` `Deployments` `Services` `NodePort` `ClusterIP` `Pod scheduling` `Horizontal scaling` `Self-healing` `Rolling updates` `Rollbacks` `containerd` `YAML manifests` `Python Flask` `Redis`
