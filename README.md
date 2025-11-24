# DevSecOps Project: Deploy Netflix Clone on AWS using Jenkins

This project outlines the steps for deploying a Netflix Clone application (built with **React/MUI**) to AWS using **Jenkins**, incorporating security scanning (**SonarQube, Trivy, OWASP Dependency-Check**) and monitoring (**Prometheus and Grafana**).

## Table of Contents
- [Phase 1: Initial Setup and Deployment](#phase-1-initial-setup-and-deployment)
- [Phase 2: Security Tools Setup](#phase-2-security-tools-setup)
- [Phase 3: CI/CD Setup with Jenkins](#phase-3-cicd-setup-with-jenkins)
- [Phase 4: Monitoring (Prometheus and Grafana)](#phase-4-monitoring-prometheus-and-grafana)
- [Phase 5: Notification](#phase-5-notification)
- [Phase 6: Kubernetes](#phase-6-kubernetes)
- [Phase 7: Cleanup](#phase-7-cleanup)

---
## 🏗️ Architecture
```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#ff9900', 'edgeLabelBackground':'#ffffff', 'tertiaryColor': '#F6F8FA'}}}%%
graph TD
    subgraph "Source & Triggers"
        Dev[👨‍💻 Developer]
        GitHub[📂 GitHub Repository<br/>(Source Code)]
        TMDB[🎬 TMDB API]
    end

    subgraph "CI Server (AWS EC2)"
        Jenkins[⚙️ Jenkins Server]
        
        subgraph "Security & Analysis"
            OWASP[🛡️ OWASP<br/>Dependency Check]
            Sonar[🔍 SonarQube<br/>(Code Quality)]
            Trivy[u'📦 Trivy<br/>(File & Image Scan)']
        end
        
        DockerBuild[🐳 Docker Build]
    end

    subgraph "Artifact Management"
        DockerHub[🗄️ DockerHub<br/>Registry]
    end

    subgraph "CD & Deployment (Kubernetes)"
        ArgoCD[🐙 ArgoCD<br/>(GitOps Controller)]
        K8sCluster[☸️ Kubernetes Cluster]
        NetflixApp[📱 Netflix App<br/>(Pod)]
    end

    subgraph "Observability"
        Prometheus[🔥 Prometheus<br/>(Time Series DB)]
        Grafana[📊 Grafana<br/>(Visualization)]
        NodeExp[📈 Node Exporter]
        Email[📧 Email Notification]
    end

    %% Flow Connections
    Dev -->|Push Code| GitHub
    GitHub -->|Webhook/Poll| Jenkins
    
    %% CI Flow
    Jenkins -->|1. SCA Scan| OWASP
    Jenkins -->|2. SAST Scan| Sonar
    Jenkins -->|3. File Scan| Trivy
    Jenkins -->|4. Build & Tag| DockerBuild
    TMDB -.->|API Key Injection| DockerBuild
    DockerBuild -->|5. Image Scan| Trivy
    DockerBuild -->|6. Push Image| DockerHub
    
    %% CD Flow
    ArgoCD -->|Sync Manifests| GitHub
    ArgoCD -->|Pull Image| DockerHub
    ArgoCD -->|Deploy| NetflixApp
    NetflixApp -.->|Run on| K8sCluster

    %% Monitoring Flow
    NodeExp -->|Metrics| Jenkins
    Prometheus -->|Scrape Metrics| Jenkins
    Prometheus -->|Scrape Metrics| NodeExp
    Prometheus -->|Scrape Metrics| NetflixApp
    Grafana -->|Query| Prometheus
    
    %% Notifications
    Jenkins -.->|Build Status| Email

    %% Styles
    style Jenkins fill:#fff3e0,stroke:#fb8c00
    style K8sCluster fill:#e1f5fe,stroke:#039be5
    style DockerHub fill:#e3f2fd,stroke:#1565c0
    style Sonar fill:#ffebee,stroke:#e53935
    style Trivy fill:#ffebee,stroke:#e53935
    style OWASP fill:#ffebee,stroke:#e53935
    style Grafana fill:#f3e5f5,stroke:#8e24aa
  ```
  
## Phase 1: Initial Setup and Deployment

### Step 1: Launch and Connect to EC2 Instance
Provision an **EC2 instance** (Ubuntu 22.04 recommended) and connect using SSH.

### Step 2: Clone the Repository
```bash
sudo apt update -y
sudo apt upgrade -y
git clone https://github.com/Chariis/DevSecOps-project.git
```

### Step 3: Install Docker and Run the Application
#### Install Docker
```bash
sudo apt-get install docker.io -y
```

#### Add User to Docker Group
```bash
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

#### Build and Run (initial attempt will fail due to missing API key)
```bash
docker build -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest
```

### Step 4: Get the TMDB API Key
1. Create an account on **TMDB**
2. Navigate to **Settings → API**
3. Create a new **Developer API Key**
4. Build and run with the key:
```bash
docker build --build-arg TMDB_V3_API_KEY=<your-tmdb-api-key> -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest
```

---

## Phase 2: Security Tools Setup

### 1. Install SonarQube
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
Access at: `http://<PublicIP>:9000` (default: admin/admin)

### 2. Install Trivy
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```
Example image scan:
```bash
trivy image <imageid>
```

---

## Phase 3: CI/CD Setup with Jenkins

### 1. Install Jenkins
#### Install Java
```bash
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
```

#### Install Jenkins
```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```
Access Jenkins at `http://<PublicIP>:8080`.

### 2. Install Plugins
Install the following from **Manage Jenkins → Plugins**:
- Eclipse Temurin Installer
- SonarQube Scanner
- NodeJs Plugin
- Email Extension Plugin
- OWASP Dependency-Check
- Docker & Docker Pipeline Plugins
- Docker API
- docker-build-step

### 3. Configure Global Tools
- **JDK:** Install JDK 17 via Temurin
- **NodeJs:** Install Node 16
- **OWASP Dependency-Check:** Name `DP-Check`
- **SonarQube Scanner:** Add tool and congigure sonarqube container in "system"

### 4. Add Credentials
- Sonar Token → ID: `Sonar-token`
- DockerHub → ID: `docker` (username: `chariis15`)

### 5. Fix Docker Permissions
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 6. Jenkinsfile
Copy from the "Jenkinsfile" in the repo. Includes:
- Checkout
- SonarQube Scan
- Trivy Scan
- Dependency Check
- Docker Build/Push
- Deploy with API key placeholder
Remember to replace <yourapikey> with a placeholder or use Jenkins Secrets management.

---

## Phase 4: Monitoring (Prometheus and Grafana)

### 1. Install Prometheus
Create user, download, extract, and configure:
```bash
sudo useradd --system --no-create-home --shell /bin/false prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
```
(Move files, configure directories, and add systemd service as outlined)
```bash
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

Write into Prometheus Service File
```bash
sudo nano /etc/systemd/system/prometheus.service
```
Copy the text below:
```plaintext
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
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

Start Prometheus:
Start service:
```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
```
Access at: `http://<server-ip>:9090`

### 2. Install Node Exporter
(Create user, download, extract, move binary, and add service as outlined)
```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget [https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz](https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz)
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

Write into Node Exporter Service File 
```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Copy the text below:
```plaintext
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

Start service:
```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

### 3. Configure Prometheus Targets
Add jobs for **Node Exporter** and **Jenkins** to `/etc/prometheus/prometheus.yml`.

Copy:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus' # Requires Jenkins Prometheus Plugin
    static_configs:
      - targets: ['<your-jenkins-ip>:8080'] # Replace with your Jenkins IP/Port
```

Reload:
```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

### 4. Install Grafana
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo gpg --dearmor | sudo tee /usr/share/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get -y install grafana
```
Start Grafana:
```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```
Access at `http://<server-ip>:3000` (default: admin/admin).

Add Prometheus as datasource, then import dashboard (e.g., ID: 1860).

---

## Phase 5: Notification
Use **Email Extension Plugin** and configure SMTP settings. Add `mail` steps in the Jenkins pipeline `post` section.

---

## Phase 6: Kubernetes

### 1. Create Kubernetes Cluster
Use **EKS**, **kOps**, or **minikube**.

### 2. Monitor with Prometheus
Install Node Exporter:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace prometheus-node-exporter
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter
```
Add Prometheus scrape config for app metrics. Update your Prometheus configuration (prometheus.yml) to add a job for scraping metrics from your deployed application (if it exposes a metrics endpoint, e.g., on Node Exporter's default port 9100):
```bash
  - job_name: 'Netflix-Node-Metrics'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['node1Ip:9100'] # Replace with your cluster node IP
```

### 3. Deploy with ArgoCD
- Install ArgoCD
- Connect GitHub repo: `https://github.com/Chariis/DevSecOps-project.git`
- Create ArgoCD Application:
  - Name: `netflix-clone`
  - Source: repo + manifests path
  - SyncPolicy: auto

Access app at `http://<NodeIP>:30007`.

---

## Phase 7: Cleanup
Terminate all unused **AWS EC2 instances** to avoid charges.
