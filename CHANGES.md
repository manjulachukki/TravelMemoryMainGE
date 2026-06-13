# TravelMemory - DevOps Pipeline Changes

## Project Overview
End-to-End DevOps Pipeline for a Web Application with CI/CD using Jenkins, Docker, Kubernetes (AWS EKS), Terraform, Ansible, Prometheus, and Grafana.

---

## Sprint 1: Architecture Design, Dockerization & Jenkins Setup

### Dockerfiles Created

**Backend** (`backend/dockerfile`)
- Base image: `node:18`
- Exposes port `3001` with `ENV PORT=3001`
- Entry point: `node index.js`
- Layer-optimized: copies `package*.json` first, then source

**Frontend** (`frontend/dockerfile`)
- Base image: `node:18`
- Exposes port `3000`
- Entry point: `npm start`

**Docker Ignore** (`backend/.dockerignore`, `frontend/.dockerignore`)
- Excludes `node_modules` and `.git`

**Docker Compose** (`docker-compose.yml`)
- Backend service (port 3001)
- Frontend service (port 3000)
- MongoDB service (port 27017) with persistent volume

### Jenkinsfile (`Jenkinsfile`)
Multi-stage pipeline with 7 stages:
1. **Checkout** — SCM checkout
2. **Build & Push Docker Images** — Parallel build of frontend/backend, push to ECR
3. **Terraform - Provision Infrastructure** — `init → plan → apply`
4. **Ansible - Configure Infrastructure** — Galaxy install + playbook execution
5. **Deploy to EKS** — Apply all K8s manifests
6. **Deploy Monitoring** — Apply Prometheus/Grafana manifests
7. **Verify Deployment** — Rollout status check

---

## Sprint 2: AWS Infrastructure Provisioning with Terraform

### Files Created (`terraform/`)

| File | Purpose |
|------|---------|
| `provider.tf` | AWS provider config, S3 backend for state |
| `variables.tf` | All configurable variables (region, CIDR, EKS version, instance type) |
| `vpc.tf` | VPC, Internet Gateway, NAT Gateway, public/private subnets (multi-AZ), route tables |
| `security_groups.tf` | EKS cluster SG (443 from VPC), node SG (cluster + inter-node) |
| `eks.tf` | EKS cluster, IAM roles, node group in private subnets with scaling config |
| `ecr.tf` | ECR repositories for backend & frontend with scan-on-push |
| `outputs.tf` | Exports VPC ID, cluster name/endpoint, ECR URLs, subnet IDs |
| `terraform.tfvars` | Dev environment values |

### Key Architecture Decisions
- **Region:** `ap-south-1` (Mumbai)
- **VPC:** `10.0.0.0/16` with 2 public + 2 private subnets across AZs
- **EKS:** Version 1.28, `t3.medium` nodes (2 desired, 1-3 scaling)
- **Worker nodes in private subnets** — secure, internet via NAT
- **Subnet tagging** for AWS Load Balancer Controller (`kubernetes.io/role/elb`)

---

## Sprint 3: Configuration Management with Ansible

### Enhanced Files (`ansible/`)

**`inventory/dev/group_vars/all.yml`** — Updated variables:
- `aws_region`, `eks_cluster_name`
- Extended `common_packages` with Docker prerequisites
- Added `docker_packages` list
- `kubectl_version: "1.28.0"`

**`roles/common/tasks/main.yml`** — Enhanced tasks:
1. Install common packages (curl, vim, git, unzip, apt-transport-https, etc.)
2. Set timezone (Asia/Kolkata)
3. Add Docker GPG key
4. Add Docker APT repository
5. Install Docker (docker-ce, docker-ce-cli, containerd.io)
6. Start and enable Docker service
7. Add user to docker group
8. Install kubectl (version-pinned binary)
9. Install AWS CLI v2 (idempotent with `creates`)
10. Configure kubeconfig for EKS

---

## Sprint 4: Kubernetes Deployment on EKS

### Files Created (`k8s/`)

| File | Purpose |
|------|---------|
| `namespace.yml` | `travelmemory` namespace |
| `backend-deployment.yml` | 2 replicas, resource limits, liveness/readiness probes (`/hello`), MONGO_URI from Secret |
| `backend-service.yml` | ClusterIP service (internal) on port 3001 |
| `frontend-deployment.yml` | 2 replicas, resource limits, liveness/readiness probes (`/`), backend URL env |
| `frontend-service.yml` | LoadBalancer service — exposes port 80 → 3000 (public) |
| `hpa.yml` | HorizontalPodAutoscaler for both deployments (CPU 70%, Memory 80%, max 5 pods) |

### Key Features
- **Load Balancing:** Frontend uses AWS ELB via `type: LoadBalancer`
- **Health Checks:** Liveness (30s delay, 10s interval) + Readiness (5s delay, 5s interval)
- **Auto-scaling:** 2-5 pods based on CPU/Memory utilization
- **Resource Limits:** Prevents noisy-neighbor issues

---

## Sprint 5: Monitoring with Prometheus & Grafana

### Files Created (`monitoring/`)

| File | Purpose |
|------|---------|
| `namespace.yml` | `monitoring` namespace |
| `prometheus-config.yml` | Scrape config (API server, nodes, pods, app) + alert rules |
| `prometheus-deployment.yml` | ServiceAccount, RBAC, Deployment, LoadBalancer Service (port 9090) |
| `grafana-deployment.yml` | Deployment with auto-provisioned Prometheus datasource, Service (port 3000) |

### Alert Rules Configured
| Alert | Condition | Severity |
|-------|-----------|----------|
| HighCPUUsage | > 80% for 5 min | Warning |
| PodNotReady | Not ready for 3 min | Critical |
| HighMemoryUsage | > 85% for 5 min | Warning |

### Monitoring Stack
- **Prometheus** `v2.47.0` — 15-day retention, cluster-wide RBAC
- **Grafana** `10.1.0` — Auto-connects to Prometheus datasource

---

## Project Structure (Final)

```
TravelMemoryMainGE/
├── ansible/
│   ├── ansible.cfg
│   ├── requirements.yml
│   ├── inventory/dev/
│   │   ├── hosts.ini
│   │   └── group_vars/all.yml
│   ├── playbooks/site.yml
│   └── roles/common/tasks/main.yml
├── backend/
│   ├── dockerfile
│   ├── .dockerignore
│   ├── index.js, conn.js, package.json
│   ├── controllers/, models/, routes/
├── frontend/
│   ├── dockerfile
│   ├── .dockerignore
│   ├── package.json
│   └── src/, public/
├── k8s/
│   ├── namespace.yml
│   ├── backend-deployment.yml
│   ├── backend-service.yml
│   ├── frontend-deployment.yml
│   ├── frontend-service.yml
│   └── hpa.yml
├── monitoring/
│   ├── namespace.yml
│   ├── prometheus-config.yml
│   ├── prometheus-deployment.yml
│   └── grafana-deployment.yml
├── terraform/
│   ├── provider.tf
│   ├── variables.tf
│   ├── vpc.tf
│   ├── security_groups.tf
│   ├── eks.tf
│   ├── ecr.tf
│   ├── outputs.tf
│   └── terraform.tfvars
├── docker-compose.yml
├── Jenkinsfile
├── azure-pipelines.yml
└── README.md
```

---

## Pre-Deployment Checklist

- [ ] Replace `<AWS_ACCOUNT_ID>` in `k8s/backend-deployment.yml` and `k8s/frontend-deployment.yml`
- [ ] Update `ansible/inventory/dev/hosts.ini` with actual EC2 IP
- [ ] Create S3 bucket `travelmemory-terraform-state` in `ap-south-1`
- [ ] Configure Jenkins credentials (AWS IAM role/keys)
- [ ] Create K8s Secret `travelmemory-secrets` with `mongo-uri` key
- [ ] Set `AWS_ACCOUNT_ID` environment variable in Jenkins

---

## Manual Steps Required (Step-by-Step)

### Step 1: AWS Account Setup
```bash
# Login to AWS Console and note your Account ID (12-digit number)
# Example: 123456789012
```

### Step 2: Create S3 Bucket for Terraform State
```bash
aws s3 mb s3://travelmemory-terraform-state --region ap-south-1
aws s3api put-bucket-versioning --bucket travelmemory-terraform-state --versioning-configuration Status=Enabled
```

### Step 3: Create IAM User for Jenkins
```bash
# Create IAM user with programmatic access
aws iam create-user --user-name jenkins-cicd

# Attach required policies
aws iam attach-user-policy --user-name jenkins-cicd --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
aws iam attach-user-policy --user-name jenkins-cicd --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess
aws iam attach-user-policy --user-name jenkins-cicd --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
aws iam attach-user-policy --user-name jenkins-cicd --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess
aws iam attach-user-policy --user-name jenkins-cicd --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Create access keys (save these securely)
aws iam create-access-key --user-name jenkins-cicd
```

### Step 4: Launch Jenkins EC2 Instance
```bash
# Launch Ubuntu 22.04 EC2 instance (t3.medium recommended)
# Security Group: Allow inbound port 8080 (Jenkins UI), 22 (SSH)

# SSH into the instance and install Jenkins:
sudo apt update
sudo apt install -y openjdk-17-jdk
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Install Docker on Jenkins server
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Install kubectl
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install Terraform
sudo apt install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform

# Install Ansible
sudo apt install -y ansible
```

### Step 5: Configure Jenkins (UI)
1. Access Jenkins at `http://<EC2_PUBLIC_IP>:8080`
2. Get initial password: `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
3. Install suggested plugins + these additional plugins:
   - **Docker Pipeline**
   - **Kubernetes CLI**
   - **AWS Credentials**
   - **Pipeline: AWS Steps**
   - **AnsiColor** (for colored Ansible output)
4. Go to **Manage Jenkins → Credentials → Global**:
   - Add **AWS Credentials** (Access Key + Secret Key from Step 3)
   - Add **Secret text** for `AWS_ACCOUNT_ID`
5. Go to **Manage Jenkins → System → Environment Variables**:
   - Add `AWS_ACCOUNT_ID` = your 12-digit account ID

### Step 6: Create Jenkins Pipeline Job
1. **New Item → Pipeline**
2. Name: `travelmemory-pipeline`
3. Under **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/manjulachukki/TravelMemoryMainGE.git`
   - Branch: `*/manjula-capstone-sp1`
   - Script Path: `Jenkinsfile`
4. Under **Build Triggers**:
   - Check **GitHub hook trigger for GITScm polling**

### Step 7: Replace AWS Account ID in K8s Manifests
```bash
# Replace placeholder with your actual AWS Account ID
sed -i 's/<AWS_ACCOUNT_ID>/123456789012/g' k8s/backend-deployment.yml
sed -i 's/<AWS_ACCOUNT_ID>/123456789012/g' k8s/frontend-deployment.yml
```

### Step 8: Run Terraform (First Time)
```bash
cd terraform
terraform init
terraform plan
terraform apply
```

### Step 9: Update Ansible Inventory
```bash
# After Terraform provisions EC2, get the public IP and update:
# ansible/inventory/dev/hosts.ini
# Replace 10.0.0.10 with actual EC2 IP
```

### Step 10: Create Kubernetes Secret for MongoDB
```bash
# After EKS is provisioned:
aws eks update-kubeconfig --name travelmemory-eks --region ap-south-1

# Create namespace first
kubectl apply -f k8s/namespace.yml

# Create secret with your MongoDB connection string
kubectl create secret generic travelmemory-secrets \
  --from-literal=mongo-uri="mongodb+srv://<username>:<password>@<cluster>.mongodb.net/travelmemory" \
  -n travelmemory
```

### Step 11: Configure GitHub Webhook (Auto-trigger Jenkins)
1. Go to GitHub repo → **Settings → Webhooks → Add webhook**
2. Payload URL: `http://<JENKINS_EC2_IP>:8080/github-webhook/`
3. Content type: `application/json`
4. Events: **Just the push event**

### Step 12: Verify End-to-End Pipeline
1. Make a code change and push to `manjula-capstone-sp1`
2. Jenkins should auto-trigger
3. Verify all stages pass in Jenkins Blue Ocean
4. Access the app via the LoadBalancer URL:
   ```bash
   kubectl get svc frontend-service -n travelmemory
   # Use the EXTERNAL-IP to access the app
   ```
5. Access monitoring:
   ```bash
   kubectl get svc -n monitoring
   # Prometheus: <EXTERNAL-IP>:9090
   # Grafana: <EXTERNAL-IP>:3000 (admin/admin123)
   ```

---

## Cost Optimization Notes (10% evaluation criteria)

- `t3.medium` instances — burstable, cost-effective for dev/staging
- EKS node scaling: min 1, max 3 — scales down during low traffic
- HPA scales pods 2→5 based on actual utilization
- NAT Gateway: single AZ for dev (use multi-AZ for prod)
- ECR `scan_on_push` avoids need for separate security scanning tools
- Prometheus uses `emptyDir` (no EBS volumes) for dev — switch to PVC for prod
