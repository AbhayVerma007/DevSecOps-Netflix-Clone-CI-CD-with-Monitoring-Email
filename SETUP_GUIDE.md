# DevSecOps Netflix Clone — Complete Setup & Version Control Guide
**Pipeline Version:** 2.0.x | **Updated:** 2026-04-20

---

## What Was Fixed in the Jenkinsfile

| # | Issue Found | Fix Applied |
|---|-------------|-------------|
| 1 | TMDB API key hardcoded in plain text | Moved to Jenkins credential `tmdb-api-key` |
| 2 | Trivy image scan targeted wrong image (`sevenajay/netflix`) | Now scans the image actually built (`rutik/netflix`) |
| 3 | No version tagging — every build overwrites `:latest` | Added `BUILD_VERSION=2.0.${BUILD_NUMBER}` — every build gets a unique tag |
| 4 | `node16` is EOL | Upgraded to `node18` LTS |
| 5 | Old container not removed before re-deploying | Added `docker rm -f netflix` before `docker run` |
| 6 | Email subject didn't include build version | Subject now shows version + result clearly |

---

## Version Control Strategy

### 1 — Initialize Git Repository

```bash
# In your project root
git init
git remote add origin https://github.com/<your-username>/netflix-devsecops.git
```

### 2 — Create a .gitignore

```bash
cat > .gitignore << 'EOF'
# Security — never commit these
.env
*.key
*secret*
*credentials*
trivyfs.txt
trivyimage.txt
dependency-check-report.xml

# Node
node_modules/
npm-debug.log*

# Build artifacts
dist/
build/

# Docker
.dockerignore
EOF
```

### 3 — Branching Strategy (Git Flow)

```
main          ← production-ready, protected branch
  └── develop ← integration branch
        ├── feature/add-monitoring
        ├── feature/update-pipeline
        └── hotfix/fix-trivy-scan
```

```bash
# Create branches
git checkout -b develop
git checkout -b feature/update-pipeline develop

# After changes, merge back
git checkout develop
git merge --no-ff feature/update-pipeline
git tag -a v2.0.0 -m "Fix Trivy scan target, add versioned Docker tags"
git push origin develop --tags
```

### 4 — Tagging Convention

| Tag Format | Example | Meaning |
|---|---|---|
| `vMAJOR.MINOR.PATCH` | `v2.0.0` | Full release |
| `vMAJOR.MINOR.BUILD` | `v2.0.42` | Per Jenkins build |
| `hotfix/vX.Y.Z` | `hotfix/v2.0.1` | Emergency fix |

---

## Step-by-Step Infrastructure Setup

---

### STEP 1 — Launch AWS EC2 Instance

- **Type:** t2.large (minimum)
- **OS:** Ubuntu 22.04 LTS
- **Storage:** 30 GB minimum
- **Security Group — open these ports:**

| Port | Service |
|------|---------|
| 22 | SSH |
| 8080 | Jenkins |
| 9000 | SonarQube |
| 9090 | Prometheus |
| 3000 | Grafana |
| 8081 | Netflix App (Docker) |
| 30000–32767 | Kubernetes NodePort |

---

### STEP 2A — Install Jenkins

```bash
# Create and run the Jenkins install script as root
sudo vi /opt/jenkins.sh
```

Paste the following content:

```bash
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | \
    tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] \
    https://packages.adoptium.net/artifactory/deb \
    $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | \
    tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
    sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/" | \
    sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

```bash
sudo chmod +x /opt/jenkins.sh
sudo /opt/jenkins.sh

# Get the initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Access Jenkins at: `http://<EC2-PUBLIC-IP>:8080`

---

### STEP 2B — Install Docker

```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
newgrp docker
sudo systemctl enable docker
sudo systemctl start docker
```

Start SonarQube as a Docker container (needs port 9000 open):

```bash
docker run -d \
    --name sonar \
    --restart unless-stopped \
    -p 9000:9000 \
    sonarqube:lts-community
```

Default SonarQube login: `admin` / `admin`

---

### STEP 2C — Install Trivy

```bash
sudo vi /opt/trivy.sh
```

```bash
#!/bin/bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
    gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
    https://aquasecurity.github.io/trivy-repo/deb \
    $(lsb_release -sc) main" | \
    sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

```bash
sudo chmod +x /opt/trivy.sh && sudo /opt/trivy.sh
trivy --version   # verify
```

---

### STEP 3 — Get TMDB API Key

1. Go to [https://www.themoviedb.org](https://www.themoviedb.org) and create a free account.
2. Navigate to: **Profile → Settings → API → Create → Developer**.
3. Accept terms and fill out the form to receive your API key.

**Important:** Store your key in Jenkins credentials — do NOT paste it into the Jenkinsfile.

In Jenkins → **Manage Jenkins → Credentials → Global → Add Credential:**
- Kind: `Secret text`
- ID: `tmdb-api-key`
- Secret: `<your TMDB API key>`

---

### STEP 4 — Install Prometheus & Grafana (Monitoring Server)

Use a **separate** EC2 instance (t2.medium recommended).

#### Prometheus

```bash
# Create system user
sudo useradd --system --no-create-home --shell /bin/false prometheus

# Download and extract (check https://github.com/prometheus/prometheus/releases for latest version)
wget https://github.com/prometheus/prometheus/releases/download/v2.51.0/prometheus-2.51.0.linux-amd64.tar.gz
tar -xvf prometheus-2.51.0.linux-amd64.tar.gz

# Move binaries and configs
sudo mkdir -p /data /etc/prometheus
cd prometheus-2.51.0.linux-amd64/
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

# Clean up
cd .. && rm -rf prometheus-2.51.0.linux-amd64*
```

Create systemd service:

```bash
sudo tee /etc/systemd/system/prometheus.service > /dev/null << 'EOF'
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/data \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.console.templates=/etc/prometheus/consoles \
    --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

#### Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | \
    sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Access Grafana at: `http://<monitoring-server-ip>:3000`
Default login: `admin` / `admin`

---

### STEP 5 — Jenkins Plugins to Install

In Jenkins → **Manage Jenkins → Manage Plugins → Available:**

| Plugin | Purpose |
|--------|---------|
| Eclipse Temurin Installer | JDK 17 toolchain |
| NodeJS Plugin | Node 18 toolchain |
| SonarQube Scanner | Static code analysis |
| OWASP Dependency-Check | Dependency vulnerability scan |
| Docker / Docker Pipeline | Docker build + push |
| Kubernetes / Kubernetes CLI | kubectl from Jenkins |
| Prometheus Metrics | Expose Jenkins metrics |
| Email Extension | Build result emails |

After installing, configure tools in **Manage Jenkins → Global Tool Configuration:**

- JDK: name = `jdk17`, install automatically from Adoptium
- NodeJS: name = `node18`, version = `18.x`
- SonarQube Scanner: name = `sonar-scanner`
- OWASP Dependency Check: name = `DP-Check`

---

### STEP 6 — Configure SonarQube in Jenkins

1. In SonarQube (`http://<ip>:9000`): **Administration → Security → Users → Generate Token**
2. Copy the token.
3. In Jenkins: **Manage Jenkins → Credentials → Add** → Secret text, ID = `Sonar-token`
4. In Jenkins: **Manage Jenkins → Configure System → SonarQube Servers:**
   - Name: `sonar-server`
   - URL: `http://<sonar-ip>:9000`
   - Token: select `Sonar-token`

---

### STEP 7 — Configure Docker Credentials

1. In Jenkins: **Manage Jenkins → Credentials → Global → Add Credential:**
   - Kind: `Username with password`
   - ID: `docker`
   - Username: your Docker Hub username
   - Password: your Docker Hub password or access token

---

### STEP 8 — Create the Jenkins Pipeline

1. Jenkins → **New Item → Pipeline**
2. Name it: `Netflix-DevSecOps`
3. Under **Pipeline → Definition:** choose `Pipeline script from SCM`
4. SCM: Git, repo URL, branch: `main`
5. Script path: `Jenkinsfile`
6. Save and click **Build Now**

---

### STEP 9 — Kubernetes Setup (Master + Worker)

Run on both master and worker nodes:

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Kernel modules and sysctl
sudo modprobe overlay
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/kubernetes.conf > /dev/null << 'EOF'
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                  = 1
EOF
sudo sysctl --system

# Install containerd
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] \
    https://apt.kubernetes.io/ kubernetes-xenial main" | \
    sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

On **master only:**

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Setup kubeconfig
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Print join command for worker nodes
kubeadm token create --print-join-command
```

On each **worker node** — run the join command from above output.

Save the kubeconfig as a Jenkins credential:

```bash
cat $HOME/.kube/config
# Copy the output → save as secret-file.txt
```

In Jenkins: **Credentials → Add → Secret file**, ID = `k8s`, upload `secret-file.txt`

---

### STEP 10 — Add Node Exporter to All Kubernetes Nodes

Run on master and worker:

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter

# Check https://github.com/prometheus/node_exporter/releases for latest
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.0.linux-amd64.tar.gz
sudo mv node_exporter-1.8.0.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*

sudo tee /etc/systemd/system/node_exporter.service > /dev/null << 'EOF'
[Unit]
Description=Node Exporter
After=network-online.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

Add to Prometheus config (`/etc/prometheus/prometheus.yml`):

```yaml
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<jenkins-ip>:8080']

  - job_name: 'node_export_master'
    static_configs:
      - targets: ['<master-ip>:9100']

  - job_name: 'node_export_worker'
    static_configs:
      - targets: ['<worker-ip>:9100']
```

Reload Prometheus (no downtime):

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

---

### STEP 11 — Access the App

```bash
# Docker container
http://<ec2-ip>:8081

# Kubernetes
kubectl get svc -n default
# Use NodePort shown → http://<worker-ip>:<nodeport>
```

---

## Quick Reference — Useful Commands

```bash
# Jenkins service
sudo systemctl status jenkins
sudo systemctl restart jenkins

# Check all running containers
docker ps -a

# Check Kubernetes pods
kubectl get pods -A
kubectl get svc

# View Trivy scan output
cat trivyfs.txt
cat trivyimage.txt

# Prometheus targets health
curl http://localhost:9090/api/v1/targets | jq .

# Git: list all version tags
git tag -l --sort=-version:refname | head -10
```
