# 🐮 Wisecow Application — Accuknox DevOps Assessment


## Problem Statement 1

### Objective
Containerise and deploy the Wisecow application on a Kubernetes environment (Minikube) with secure TLS communication and an automated CI/CD pipeline.

---

### Dockerisation

A `Dockerfile` was created to containerise the Wisecow application.

```dockerfile
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    cowsay fortune-mod netcat-openbsd \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

ENV PATH="/usr/games:${PATH}"

WORKDIR /app
COPY wisecow.sh .
RUN chmod +x wisecow.sh

EXPOSE 4499
CMD ["./wisecow.sh"]
```

**Build and run locally:**
```bash
docker build -t wisecow .
docker run -p 4499:4499 wisecow
```

The image is published to Docker Hub: `sandeepreddymunagala/wisecow`

---

### Kubernetes Deployment

The application is deployed with 2 replicas and exposed via a `ClusterIP` service. External access is handled by the Nginx Ingress controller with TLS termination.

```bash
# Start Minikube
minikube start --driver=docker

# Apply deployment and service
kubectl apply -f wisecow-deployment.yaml

# Verify pods are running
kubectl get pods -l app=wisecow
kubectl get svc wisecow-service
```

**`wisecow-deployment.yaml` creates:**
- `Deployment` — 2 replicas of the wisecow container
- `Service` — ClusterIP on port 80 → container port 4499

---

### TLS Implementation

TLS is implemented using **cert-manager** with a self-signed `ClusterIssuer` and **Nginx Ingress** for TLS termination.

#### Step 1 — Enable Nginx Ingress Addon
```bash
minikube addons enable ingress

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

#### Step 2 — Install cert-manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml

kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=120s
```

#### Step 3 — Apply TLS Manifests
```bash
kubectl apply -f cluster-issuer.yaml
kubectl apply -f certificate.yaml
kubectl apply -f wisecow-deployment.yaml
kubectl apply -f ingress.yaml

# Verify certificate is issued
kubectl get certificate -n default
# NAME          READY   SECRET               AGE
# wisecow-tls   True    wisecow-tls-secret   30s
```

#### Step 4 — Patch Ingress Controller to LoadBalancer
```bash
kubectl patch svc ingress-nginx-controller \
  -n ingress-nginx \
  -p '{"spec":{"type":"LoadBalancer"}}'
```

#### Step 5 — Start minikube tunnel (separate terminal, keep running)
```bash
minikube tunnel
```

#### Step 6 — Configure DNS and Test
```bash
echo "127.0.0.1  wisecow.local" | sudo tee -a /etc/hosts
```
```bash
# Test HTTPS
curl -k https://wisecow.local
```

**App running over HTTPS at `https://wisecow.local`:**



> The browser shows "Not secure" because the certificate is self-signed (local dev). TLS encryption is still active.

---

### CI/CD Pipeline

The GitHub Actions pipeline (`.github/workflows/cicd.yaml`) automates build, image update, and deployment on every push to `main`.

#### Pipeline Flow

```
git push origin main
       │
       ▼
[Job 1 — Build & Push] (GitHub-hosted runner)
  docker build + push → sandeepreddymunagala/wisecow:N
  docker push → sandeepreddymunagala/wisecow:latest
       │
       ▼
[Job 2 — Update Manifest] (GitHub-hosted runner)
  sed: update image tag in wisecow-deployment.yaml → :N
  git commit + push back to repo
       │
       ▼
[Job 3 — Deploy to Minikube] (self-hosted runner)
  kubectl apply -f wisecow-deployment.yaml
  kubectl rollout status deployment/wisecow-deployment
```

**All 3 jobs passing successfully:**



#### Setup

**1. Add GitHub Secrets:**

| Secret | Value |
|---|---|
| `DOCKER_USERNAME` | Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub password or access token |

**2. Set Workflow Permissions:**

Repo → Settings → Actions → General → Workflow permissions → **Read and write permissions** → Save

**3. Set up Self-Hosted Runner:**

Repo → Settings → Actions → Runners → New self-hosted runner → Linux → follow the displayed commands.

```bash
mkdir actions-runner && cd actions-runner
./config.sh --url https://github.com/munagalasandeep99/trial --token <TOKEN>
./run.sh   # keep this running
```

---

## Problem Statement 2

Three Bash automation scripts are implemented in the `scripts/` directory.

---

### 1. System Health Monitor — `scripts/system_monitor.sh`

Monitors CPU usage, memory usage, disk space, and running process count. If any metric exceeds the predefined threshold, an alert is written to both the console and a log file (`system_health.log`).

**Thresholds:**

| Metric | Threshold |
|---|---|
| CPU Usage | > 80% |
| Memory Usage | > 80% |
| Disk Usage | > 80% |
| Running Processes | > 400 |

**Usage:**
```bash
chmod +x scripts/system_monitor.sh
./scripts/system_monitor.sh
```

**Example output:**
```
[2026-06-01 10:30:00] ALERT: CPU usage is 85% (Threshold: 80%)
[2026-06-01 10:30:00] ALERT: Memory usage is 83% (Threshold: 80%)
Logs stored in: ./system_health.log
```

**How it works:**
- CPU: parsed from `top -bn1`
- Memory: calculated from `free` (used/total × 100)
- Disk: parsed from `df /`
- Processes: counted via `ps -e`

---

### 2. Automated Backup — `scripts/backup.sh`

Backs up a specified local directory (`/home/sandeep/Documents`) to a remote server using `scp` over SSH. Reports success or failure of the backup operation to both the console and a log file (`backup.log`).

**Usage:**
```bash
chmod +x scripts/backup.sh
./scripts/backup.sh <ssh-key.pem> <user@remote-host>

# Example:
./scripts/backup.sh ~/.ssh/mykey.pem ubuntu@192.168.1.100
```

**Example output:**
```
[2026-06-01 10:30:00] Starting backup of /home/sandeep/Documents to ubuntu@192.168.1.100:/backup
[2026-06-01 10:30:05] BACKUP SUCCESSFUL
```

**How it works:**
- Validates SSH key file and source directory exist
- Uses `scp -r` to securely copy the directory to the remote server
- Logs `BACKUP SUCCESSFUL` or `BACKUP FAILED` with timestamp to `backup.log`

---

### 3. Log File Analyser — `scripts/log_file_analyser.sh`

Analyses web server log files (Apache/Nginx combined log format) and outputs a summarised report including total requests, 404 errors, most requested pages, and top IP addresses.

**Usage:**
```bash
chmod +x scripts/log_file_analyser.sh
./scripts/log_file_analyser.sh <path-to-log-file>

# Example:
./scripts/log_file_analyser.sh /var/log/nginx/access.log
```

**Example output:**
```
Total Requests:
15234

Number of 404 Errors:
342

Top 10 Requested Pages:
    523 /index.html
    410 /about
    389 /api/data
    ...

Top 10 IP Addresses:
    145 192.168.1.10
    132 10.0.0.5
    ...

Top 10 URLs Returning 404:
     89 /wp-admin
     45 /favicon.ico
    ...
```

**How it works:**
- Total requests: `wc -l` on the log file
- 404 errors: `grep ' 404 '` count
- Top pages: `awk '{print $7}'` (URL field) + sort + uniq
- Top IPs: `awk '{print $1}'` (IP field) + sort + uniq
- Top 404 URLs: combination of grep + awk + sort

---

## Problem Statement 3

### KubeArmor Zero-Trust Policy

A zero-trust `KubeArmorPolicy` (`wisecow-kubearmor-policy.yaml`) is applied to the wisecow pods to enforce runtime security using eBPF (BPFLSM enforcer).

#### Policy Rules

| Rule Type | Policy |
|---|---|
| **Process** | Allow only: `wisecow.sh`, `cowsay`, `fortune`, `bash`, `sh`, `nc`, `cat`, `mkfifo`, `rm`, `sleep`. Block all others. |
| **File** | Block `/etc/shadow`, `/root/`, `/home/`. Audit `/etc/passwd` and `/proc/`. |
| **Network** | Allow TCP only. Block UDP and ICMP. |
| **Capabilities** | Block `net_raw`, `sys_ptrace`, `sys_admin`. |

#### Install KubeArmor
```bash
# Install karmor CLI
curl -sfL https://raw.githubusercontent.com/kubearmor/kubearmor-client/main/install.sh | sudo sh -s -- -b /usr/local/bin

# Install KubeArmor on the cluster
karmor install

# Verify
kubectl get pods -n kubearmor
```

#### Apply the Policy
```bash
kubectl apply -f wisecow-kubearmor-policy.yaml

# Verify
kubectl get KubeArmorPolicy -n default
```

#### Trigger Policy Violations
```bash
# Open shell in a wisecow pod
kubectl exec -it $(kubectl get pod -l app=wisecow -o jsonpath='{.items[0].metadata.name}') -- bash
```
```bash
# Try blocked actions inside the pod:
cat /etc/shadow        # BLOCKED
ls /root/              # BLOCKED
curl http://google.com # BLOCKED
```

#### Observe Violations
```bash
# In a separate terminal
karmor log
```

**Policy violation logs captured via `karmor log`:**

![KubeArmor Violations](kubearmor-violation.png)

**Key details from the violation log:**
- `PolicyName: wisecow-zero-trust-policy`
- `Resource: CAP_SYS_ADMIN`
- `Operation: Capabilities`
- `Action: Block`
- `Result: Permission denied`
- `Enforcer: BPFLSM`

---

