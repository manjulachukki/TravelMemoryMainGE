# TravelMemory - DevOps Pipeline Changes

## Project Overview
End-to-End DevOps Pipeline for a Web Application with CI/CD using Jenkins, Docker, Kubernetes (AWS EKS), Terraform, Ansible, Prometheus, and Grafana.

### What This Project Does (For Beginners)
This project automates the entire lifecycle of deploying a web application:
1. **Docker** packages the app into containers (portable units that run anywhere)
2. **Jenkins** automates building, testing, and deploying the app every time code changes
3. **Terraform** creates AWS cloud infrastructure (networks, servers, Kubernetes cluster) automatically
4. **Ansible** configures the servers with required software (Docker, kubectl, AWS CLI)
5. **Kubernetes (EKS)** runs and manages the containers in AWS with auto-scaling
6. **Prometheus + Grafana** monitors everything and alerts when something goes wrong

### Prerequisites You Need Before Starting
- An **AWS Account** (Free Tier eligible — sign up at https://aws.amazon.com)
- **AWS CLI** installed on your local machine ([Install Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
- **Git** installed locally ([Download](https://git-scm.com/downloads))
- A **GitHub account** with this repository forked/cloned
- A **MongoDB Atlas** free cluster (for the database — https://www.mongodb.com/cloud/atlas)
- Basic familiarity with terminal/command-line usage

---

## Sprint 1: Architecture Design, Dockerization & Jenkins Setup

### What is Docker and Why We Use It
Docker packages your application code + dependencies into a single unit called a "container." This ensures the app runs identically on your laptop, on a test server, and in production — no more "works on my machine" issues.

### Dockerfiles Created

**Backend** (`backend/dockerfile`)
- Base image: `node:18` (pre-built image with Node.js installed)
- Exposes port `3001` with `ENV PORT=3001`
- Entry point: `node index.js` (starts the backend server)
- Layer-optimized: copies `package*.json` first, then source (speeds up rebuilds)

**Frontend** (`frontend/dockerfile`)
- Base image: `node:18`
- Exposes port `3000`
- Entry point: `npm start` (starts the React development server)

**Docker Ignore** (`backend/.dockerignore`, `frontend/.dockerignore`)
- Excludes `node_modules` and `.git` (these are recreated inside the container)

**Docker Compose** (`docker-compose.yml`)
- Runs all 3 services together locally for development/testing:
  - Backend service (port 3001)
  - Frontend service (port 3000)
  - MongoDB service (port 27017) with persistent volume

#### How to Test Docker Locally
```bash
# Navigate to the project root directory
cd TravelMemoryMainGE

# Build and start all services (frontend + backend + MongoDB)
docker-compose up --build

# Wait for all services to start (you'll see log output)
# Then open your browser:
#   Frontend: http://localhost:3000
#   Backend API: http://localhost:3001/hello

# To stop all services, press Ctrl+C in the terminal, then run:
docker-compose down
```

### Jenkinsfile (`Jenkinsfile`)
The Jenkinsfile defines an automated pipeline — think of it as a recipe that Jenkins follows every time code is pushed.

Multi-stage pipeline with 7 stages:
1. **Checkout** — Downloads the latest code from GitHub
2. **Build & Push Docker Images** — Builds Docker images for frontend & backend in parallel, then pushes them to AWS ECR (a private Docker image registry)
3. **Terraform - Provision Infrastructure** — Creates/updates all AWS resources (VPC, EKS cluster, etc.)
4. **Ansible - Configure Infrastructure** — Installs required software on the EC2 instances
5. **Deploy to EKS** — Deploys the application containers to the Kubernetes cluster
6. **Deploy Monitoring** — Deploys Prometheus and Grafana for monitoring
7. **Verify Deployment** — Checks that all deployments are healthy and running

---

## Sprint 2: AWS Infrastructure Provisioning with Terraform

### What is Terraform and Why We Use It
Terraform is an "Infrastructure as Code" (IaC) tool. Instead of manually clicking through the AWS Console to create resources, you write configuration files that describe what you want. Terraform then creates, modifies, or destroys resources to match your configuration. Benefits:
- Reproducible: Run the same config to create identical environments
- Version controlled: Track infrastructure changes in Git
- Automated: No manual clicking, less human error

### Files Created (`terraform/`)

| File | Purpose |
|------|---------|
| `provider.tf` | Tells Terraform to use AWS and where to store its state file |
| `variables.tf` | Defines all configurable values (region, network ranges, instance sizes) |
| `vpc.tf` | Creates the network (VPC, subnets, internet gateway, NAT gateway, route tables) |
| `security_groups.tf` | Firewall rules — what traffic is allowed in/out |
| `eks.tf` | Creates the Kubernetes cluster and worker nodes |
| `ecr.tf` | Creates private Docker image repositories in AWS |
| `outputs.tf` | Prints useful info after creation (cluster endpoint, ECR URLs, etc.) |
| `terraform.tfvars` | Actual values for the dev environment |

### Key Architecture Decisions (What Was Chosen and Why)
- **Region:** `ap-south-1` (Mumbai) — closest to India-based users for low latency
- **VPC:** `10.0.0.0/16` — a private network with 65,536 IP addresses
  - 2 public subnets (for load balancers, internet-facing resources)
  - 2 private subnets (for EKS worker nodes — not directly accessible from internet)
  - Spread across 2 Availability Zones for high availability
- **EKS:** Version 1.28, `t3.medium` nodes (2 vCPU, 4GB RAM each)
  - Desired: 2 nodes, Min: 1, Max: 3 (auto-scales based on load)
- **Worker nodes in private subnets** — more secure; access internet via NAT Gateway
- **Subnet tagging** — special tags that tell AWS Load Balancer Controller which subnets to use

### Understanding the Network Architecture
```
Internet
    │
    ▼
┌─────────────────────────────────────────────┐
│  VPC (10.0.0.0/16)                          │
│                                             │
│  ┌──────────────┐    ┌──────────────┐       │
│  │ Public Subnet│    │ Public Subnet│       │
│  │  10.0.1.0/24 │    │  10.0.2.0/24 │       │
│  │  (AZ-1a)     │    │  (AZ-1b)     │       │
│  │  [Load Bal.] │    │  [NAT GW]    │       │
│  └──────┬───────┘    └──────┬───────┘       │
│         │                   │               │
│  ┌──────▼───────┐    ┌──────▼───────┐       │
│  │Private Subnet│    │Private Subnet│       │
│  │ 10.0.3.0/24  │    │ 10.0.4.0/24  │       │
│  │  (AZ-1a)     │    │  (AZ-1b)     │       │
│  │ [EKS Nodes]  │    │ [EKS Nodes]  │       │
│  └──────────────┘    └──────────────┘       │
└─────────────────────────────────────────────┘
```

---

## Sprint 3: Configuration Management with Ansible

### What is Ansible and Why We Use It
Ansible automates server configuration. Instead of SSH-ing into each server and manually running install commands, Ansible does it all for you across multiple servers simultaneously. It uses simple YAML files (called "playbooks") to describe the desired state.

### Enhanced Files (`ansible/`)

**`inventory/dev/group_vars/all.yml`** — Variables shared across all hosts:
- `aws_region`, `eks_cluster_name`
- Extended `common_packages` with Docker prerequisites
- Added `docker_packages` list
- `kubectl_version: "1.28.0"`

**`roles/common/tasks/main.yml`** — What Ansible installs/configures:
1. Install common packages (curl, vim, git, unzip, apt-transport-https, etc.)
2. Set timezone (Asia/Kolkata)
3. Add Docker GPG key (verifies Docker packages are authentic)
4. Add Docker APT repository (where to download Docker from)
5. Install Docker (docker-ce, docker-ce-cli, containerd.io)
6. Start and enable Docker service (runs on boot)
7. Add user to docker group (allows running docker without sudo)
8. Install kubectl (Kubernetes command-line tool, version-pinned)
9. Install AWS CLI v2 (for AWS operations, idempotent — skips if already installed)
10. Configure kubeconfig for EKS (connects kubectl to our cluster)

---

## Sprint 4: Kubernetes Deployment on EKS

### What is Kubernetes (K8s) and Why We Use It
Kubernetes orchestrates containers — it decides which server to run them on, restarts them if they crash, scales them up/down based on traffic, and handles networking between them. EKS is AWS's managed Kubernetes service (AWS handles the control plane, you manage the worker nodes).

### Files Created (`k8s/`)

| File | Purpose |
|------|---------|
| `namespace.yml` | Creates an isolated `travelmemory` namespace (like a folder for our resources) |
| `backend-deployment.yml` | Runs 2 copies of the backend, with health checks and MongoDB connection |
| `backend-service.yml` | Internal network endpoint for backend (only accessible within the cluster) |
| `frontend-deployment.yml` | Runs 2 copies of the frontend, with health checks |
| `frontend-service.yml` | Public-facing load balancer (users access the app through this) |
| `hpa.yml` | Auto-scales pods from 2 to 5 based on CPU/Memory usage |

### Key Features Explained
- **Load Balancing:** Frontend `type: LoadBalancer` creates an AWS ELB — distributes traffic across all frontend pods
- **Health Checks:**
  - *Liveness probe* (30s delay, 10s interval): "Is the container alive?" — restarts if not responding
  - *Readiness probe* (5s delay, 5s interval): "Is the container ready for traffic?" — removes from load balancer if not ready
- **Auto-scaling (HPA):** Automatically adds more pods when CPU > 70% or Memory > 80%
- **Resource Limits:** Each container gets a max amount of CPU/RAM — prevents one container from starving others

---

## Sprint 5: Monitoring with Prometheus & Grafana

### What is Monitoring and Why It Matters
Without monitoring, you're flying blind — you won't know if the app is slow, crashing, or running out of resources until users complain. Prometheus collects metrics (CPU, memory, request counts), and Grafana visualizes them in dashboards.

### Files Created (`monitoring/`)

| File | Purpose |
|------|---------|
| `namespace.yml` | Creates isolated `monitoring` namespace |
| `prometheus-config.yml` | Tells Prometheus what to scrape (collect metrics from) + alert rules |
| `prometheus-deployment.yml` | Runs Prometheus with proper permissions to access cluster metrics |
| `grafana-deployment.yml` | Runs Grafana pre-configured to connect to Prometheus |

### Alert Rules Configured
| Alert | Condition | Severity | What It Means |
|-------|-----------|----------|---------------|
| HighCPUUsage | > 80% for 5 min | Warning | Server is heavily loaded, might need scaling |
| PodNotReady | Not ready for 3 min | Critical | A container is failing — investigate immediately |
| HighMemoryUsage | > 85% for 5 min | Warning | Running low on memory, risk of OOM kills |

### Monitoring Stack
- **Prometheus** `v2.47.0` — Collects and stores metrics for 15 days, has cluster-wide permissions
- **Grafana** `10.1.0` — Dashboarding tool, auto-connects to Prometheus on startup

---

## Project Structure (Final)

```
TravelMemoryMainGE/
├── ansible/                        # Server configuration automation
│   ├── ansible.cfg                 # Ansible settings
│   ├── requirements.yml            # External role dependencies
│   ├── inventory/dev/              # Dev environment hosts
│   │   ├── hosts.ini              # Server IP addresses
│   │   └── group_vars/all.yml     # Shared variables
│   ├── playbooks/site.yml          # Main playbook entry point
│   └── roles/common/tasks/main.yml # Installation tasks
├── backend/                        # Node.js API server
│   ├── dockerfile                  # Container build instructions
│   ├── .dockerignore               # Files excluded from container
│   ├── index.js, conn.js, package.json
│   ├── controllers/, models/, routes/
├── frontend/                       # React web application
│   ├── dockerfile
│   ├── .dockerignore
│   ├── package.json
│   └── src/, public/
├── k8s/                            # Kubernetes deployment manifests
│   ├── namespace.yml
│   ├── backend-deployment.yml
│   ├── backend-service.yml
│   ├── frontend-deployment.yml
│   ├── frontend-service.yml
│   └── hpa.yml                     # Auto-scaling rules
├── monitoring/                     # Observability stack
│   ├── namespace.yml
│   ├── prometheus-config.yml
│   ├── prometheus-deployment.yml
│   └── grafana-deployment.yml
├── terraform/                      # Infrastructure as Code
│   ├── provider.tf
│   ├── variables.tf
│   ├── vpc.tf
│   ├── security_groups.tf
│   ├── eks.tf
│   ├── ecr.tf
│   ├── outputs.tf
│   └── terraform.tfvars
├── docker-compose.yml              # Local development setup
├── Jenkinsfile                     # CI/CD pipeline definition
├── azure-pipelines.yml             # Alternative CI/CD (Azure DevOps)
└── README.md
```

---

## Pre-Deployment Checklist

Complete these items IN ORDER before running the pipeline:

- [ ] **1.** Sign up for AWS account and note your 12-digit Account ID
- [ ] **2.** Install AWS CLI on your local machine and configure it (`aws configure`)
- [ ] **3.** Create S3 bucket `travelmemory-terraform-state` in `ap-south-1` (for Terraform state)
- [ ] **4.** Create a free MongoDB Atlas cluster and get the connection string
- [ ] **5.** Replace `<AWS_ACCOUNT_ID>` in `k8s/backend-deployment.yml` and `k8s/frontend-deployment.yml`
- [ ] **6.** Log in to hosted Jenkins at https://jenkinsacademics.herovired.com
- [ ] **7.** Configure Jenkins credentials (AWS Access Key + Secret Key) via Jenkins UI
- [ ] **8.** Set `AWS_ACCOUNT_ID` environment variable in Jenkins
- [ ] **9.** Run Terraform to provision the EKS cluster
- [ ] **10.** Update `ansible/inventory/dev/hosts.ini` with the actual EC2 IP address
- [ ] **11.** Create K8s Secret `travelmemory-secrets` with `mongo-uri` key
- [ ] **12.** Configure GitHub Webhook to auto-trigger Jenkins on code push

---

## Detailed Step-by-Step Deployment Guide (Beginner-Friendly)

> **Important:** Follow these steps in exact order. Each step depends on the previous one.

---

### Step 1: AWS Account Setup

**What you're doing:** Creating an AWS account and finding your Account ID (needed everywhere).

#### 1.1 — Create an AWS Account (skip if you already have one)
1. Go to https://aws.amazon.com and click **"Create an AWS Account"**
2. Enter your email, choose a password, and pick an account name (e.g., "MyDevOps")
3. Provide payment information (you won't be charged for Free Tier usage)
4. Complete phone verification
5. Choose the **"Basic Support - Free"** plan

#### 1.2 — Find Your AWS Account ID
1. Log into the **AWS Management Console** (https://console.aws.amazon.com)
2. Click your **username** in the top-right corner
3. Your **Account ID** is the 12-digit number shown (e.g., `123456789012`)
4. **Write this down** — you'll need it in multiple steps

#### 1.3 — Set Your Default Region
1. In the AWS Console, look at the top-right corner (next to your username)
2. Click the region dropdown and select **Asia Pacific (Mumbai) ap-south-1**
3. All resources will be created in this region

---

### Step 2: Install and Configure AWS CLI on Your Local Machine

**What you're doing:** Installing the AWS command-line tool so you can interact with AWS from your terminal.

#### 2.1 — Install AWS CLI

**On Windows:**
```powershell
# Download and run the MSI installer
# Go to: https://awscli.amazonaws.com/AWSCLIV2.msi
# Double-click the downloaded file and follow the wizard

# After installation, open a NEW terminal and verify:
aws --version
# Expected output: aws-cli/2.x.x Python/3.x.x Windows/10 ...
```

**On macOS:**
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```

**On Linux (Ubuntu/Debian):**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

#### 2.2 — Create an IAM User for CLI Access

> **Why?** You should NOT use your root account for daily operations. Create a separate IAM user.

1. Go to **AWS Console → IAM → Users → Create user**
2. User name: `admin-cli`
3. Check **"Provide user access to the AWS Management Console"** (optional)
4. Click **Next**
5. Select **"Attach policies directly"**
6. Search for and check **`AdministratorAccess`** (for learning purposes; restrict in production)
7. Click **Next → Create user**
8. Click on the user → **Security credentials** tab → **Create access key**
9. Select **"Command Line Interface (CLI)"** → check the acknowledgment → **Next → Create**
10. **IMPORTANT: Download the .csv file or copy both keys NOW** — you won't see the Secret Key again!

#### 2.3 — Configure AWS CLI with Your Credentials
```bash
aws configure
```
You'll be prompted for 4 values:
```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE        ← Paste your Access Key
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG   ← Paste your Secret Key
Default region name [None]: ap-south-1                 ← Type this exactly
Default output format [None]: json                     ← Type this exactly
```

#### 2.4 — Verify AWS CLI Works
```bash
# This should return your account info without errors
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAEXAMPLEID",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/admin-cli"
# }
```

---

### Step 3: Create S3 Bucket for Terraform State

**What you're doing:** Creating a storage location where Terraform will save its "state" — a file that tracks what resources exist in AWS. This is critical so Terraform knows what it has already created.

**Why S3?** If the state file is only on your laptop, other team members (or Jenkins) can't access it. S3 makes it shared and durable.

```bash
# Create the S3 bucket (bucket names must be globally unique)
aws s3 mb s3://travelmemory-terraform-state --region ap-south-1

# Expected output:
# make_bucket: travelmemory-terraform-state

# Enable versioning (keeps history of state files — safety net for rollbacks)
aws s3api put-bucket-versioning \
  --bucket travelmemory-terraform-state \
  --versioning-configuration Status=Enabled

# Verify the bucket was created
aws s3 ls | grep travelmemory

# Expected output:
# 2024-xx-xx xx:xx:xx travelmemory-terraform-state
```

> **Troubleshooting:** If you get "BucketAlreadyExists" error, someone else already has that bucket name. Change the name in the command AND in `terraform/provider.tf` (the `backend "s3"` block).

---

### Step 4: Create a Free MongoDB Atlas Cluster

**What you're doing:** Setting up a free cloud database that the backend will connect to.

#### 4.1 — Create MongoDB Atlas Account
1. Go to https://www.mongodb.com/cloud/atlas and click **"Try Free"**
2. Sign up with your email or Google account
3. Choose **"Shared" (Free Forever)** cluster when prompted

#### 4.2 — Create a Cluster
1. After login, click **"Build a Database"** (or "Create" if you see the dashboard)
2. Select **M0 Free Tier**
3. Cloud Provider: **AWS**
4. Region: **Mumbai (ap-south-1)** — same as our EKS cluster for low latency
5. Cluster Name: `TravelMemory` (or leave default)
6. Click **"Create Deployment"**

#### 4.3 — Set Up Database Access
1. Go to **Database Access** (left sidebar) → **Add New Database User**
2. Authentication: **Password**
3. Username: `travelmemory-app`
4. Password: Click **"Autogenerate Secure Password"** → **Copy and save this password!**
5. Database User Privileges: **Read and write to any database**
6. Click **"Add User"**

#### 4.4 — Set Up Network Access
1. Go to **Network Access** (left sidebar) → **Add IP Address**
2. Click **"Allow Access from Anywhere"** (sets `0.0.0.0/0`)
   - This is acceptable for development; restrict in production
3. Click **"Confirm"**

#### 4.5 — Get Your Connection String
1. Go to **Database** (left sidebar) → Click **"Connect"** on your cluster
2. Choose **"Connect your application"**
3. Driver: **Node.js**, Version: **5.5 or later**
4. Copy the connection string. It looks like:
   ```
   mongodb+srv://travelmemory-app:<password>@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority
   ```
5. Replace `<password>` with the password you saved in step 4.3
6. Add the database name before the `?`:
   ```
   mongodb+srv://travelmemory-app:YOUR_PASSWORD@cluster0.xxxxx.mongodb.net/travelmemory?retryWrites=true&w=majority
   ```
7. **Save this complete connection string** — you'll need it in Step 10.

---

### Step 5: Configure AWS Credentials for Jenkins (Using Existing IAM User)

**What you're doing:** Generating an additional access key for your existing IAM user so Jenkins can perform AWS operations (build images, create infrastructure, deploy to EKS).

> **Why not create a separate IAM user?** Some AWS accounts (e.g., lab/sandbox accounts, AWS Academy, or organization-managed accounts) do not grant permissions to create new IAM users. In such cases, you can reuse your existing IAM user's credentials for Jenkins. This is acceptable for development/learning environments.

#### 5.1 — Verify Your Current IAM Identity and Permissions

```bash
# Check who you are currently logged in as
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAEXAMPLEID",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/your-username"
# }
```

#### 5.2 — Create an Access Key for Jenkins (Under Your Existing User)

```bash
# Create a new access key pair for your existing IAM user
# (each IAM user can have up to 2 active access keys)
aws iam create-access-key

# IMPORTANT: Save the output! You'll see something like:
# {
#     "AccessKey": {
#         "UserName": "your-username",
#         "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
#         "Status": "Active",
#         "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
#         "CreateDate": "2024-01-01T00:00:00+00:00"
#     }
# }
#
# Copy BOTH AccessKeyId and SecretAccessKey — you'll enter these in Jenkins later.
# The SecretAccessKey is shown ONLY ONCE. If lost, you must delete and create new keys.
```

> **Note:** If `aws iam create-access-key` also fails due to permissions, you can use the **AWS Console** instead:
> 1. Log in to AWS Console → Click your username (top right) → **Security credentials**
> 2. Scroll to **Access keys** → Click **Create access key**
> 3. Select **"Command Line Interface (CLI)"** → Check acknowledgment → **Create**
> 4. Download the `.csv` file or copy both keys immediately

#### 5.3 — Verify Your Account Has the Required Permissions

Your existing IAM user needs these permissions for the pipeline to work. Run this check:
```bash
# Test ECR access
aws ecr describe-repositories --region ap-south-1 2>&1 | head -5

# Test EC2 access
aws ec2 describe-instances --region ap-south-1 --max-items 1 2>&1 | head -5

# Test S3 access
aws s3 ls 2>&1 | head -5

# Test EKS access
aws eks list-clusters --region ap-south-1 2>&1 | head -5
```

If any of these return "AccessDenied," contact your AWS account administrator to attach the relevant policies to your user:
- `AmazonEKSClusterPolicy`
- `AmazonEC2ContainerRegistryFullAccess`
- `AmazonEC2FullAccess`
- `AmazonVPCFullAccess`
- `AmazonS3FullAccess`
- `AmazonEKSWorkerNodePolicy`

#### 5.4 — Alternative: Use EC2 Instance Role (Recommended for Production)

If your Jenkins runs on an EC2 instance, the most secure approach is to attach an **IAM Instance Profile** (role) to the EC2 instance. This eliminates the need for access keys entirely:

1. Go to **AWS Console → IAM → Roles → Create role**
2. Trusted entity: **AWS service** → Use case: **EC2**
3. Attach the required policies listed above
4. Role name: `jenkins-ec2-role`
5. Go to **EC2 → Instances** → Select Jenkins instance → **Actions → Security → Modify IAM role**
6. Select `jenkins-ec2-role` → **Update IAM role**

With an instance role, the Jenkins server automatically gets credentials — no access keys to manage or rotate.

---

### Step 6: Access Hosted Jenkins Instance

**What you're doing:** Using the pre-configured Jenkins instance hosted by HeroVired at `https://jenkinsacademics.herovired.com`. Since Jenkins is already set up and running, you only need to log in and configure your pipeline — no EC2 instance provisioning, SSH, or tool installation required.

#### 6.1 — Log in to Jenkins

1. Open your browser and go to: **https://jenkinsacademics.herovired.com**
2. Log in with your HeroVired credentials (provided by your instructor)
3. You should see the Jenkins Dashboard

> **Note:** Since this is a shared hosted Jenkins, Docker, kubectl, AWS CLI, Terraform, and Ansible are already pre-installed on the server. You do NOT need to install any tools manually.

#### 6.2 — Configure AWS Credentials on Hosted Jenkins

Since you cannot SSH into the hosted Jenkins server, AWS credentials must be configured via the Jenkins UI (covered in Step 7.3). The hosted Jenkins will use these credentials when running pipeline stages that interact with AWS.

---

### Step 7: Configure Jenkins via Web UI

**What you're doing:** Setting up Jenkins through its web interface — adding credentials and creating the pipeline job.

> **Jenkins URL:** https://jenkinsacademics.herovired.com

#### 7.1 — Log in to Jenkins

1. Open your browser and go to: **https://jenkinsacademics.herovired.com**
2. Log in with your HeroVired-provided credentials
3. You should see the Jenkins Dashboard

> **Note:** Since this is a pre-configured hosted instance, initial setup (plugins, admin user creation) has already been done by the administrator. Skip directly to adding credentials.

#### 7.2 — Verify Required Plugins Are Available

Since this is a hosted Jenkins instance, the required plugins should already be installed by the administrator. You can verify by going to **Dashboard → Manage Jenkins → Plugins → Installed plugins** and checking for:
   - **Docker Pipeline** — allows Jenkins to build Docker images
   - **Kubernetes CLI** — allows Jenkins to run kubectl commands
   - **AWS Credentials** — securely stores AWS keys
   - **Pipeline: AWS Steps** — AWS integration for pipelines
   - **AnsiColor** — shows colored output (makes Ansible logs readable)

> **Note:** If any plugin is missing and you don't have permission to install it, contact your HeroVired instructor/admin to request installation.

#### 7.3 — Add AWS Credentials to Jenkins

1. Go to **Dashboard → Manage Jenkins → Credentials**
2. Click on **(global)** under "Stores scoped to Jenkins"
3. Click **"Add Credentials"** (left sidebar)
4. Fill in:
   - **Kind:** AWS Credentials
   - **Scope:** Global
   - **ID:** `aws-credentials` (must match what's in the Jenkinsfile)
   - **Description:** `Jenkins CI/CD AWS Access`
   - **Access Key ID:** Paste the `AccessKeyId` from Step 5
   - **Secret Access Key:** Paste the `SecretAccessKey` from Step 5
5. Click **"Create"**

6. Add another credential — Click **"Add Credentials"** again:
   - **Kind:** Secret text
   - **Scope:** Global
   - **ID:** `aws-account-id`
   - **Description:** `AWS Account ID`
   - **Secret:** Your 12-digit AWS Account ID (e.g., `123456789012`)
7. Click **"Create"**

#### 7.4 — Set Global Environment Variables

1. Go to **Dashboard → Manage Jenkins → System** (scroll down)
2. Find the **"Global properties"** section
3. Check **"Environment variables"**
4. Click **"Add"** and enter:
   - Name: `AWS_ACCOUNT_ID`
   - Value: Your 12-digit AWS Account ID
5. Click **"Save"** at the bottom

#### 7.5 — Create the Pipeline Job

1. Go to **Dashboard → New Item** (left sidebar)
2. Enter name: `travelmemory-pipeline`
3. Select **"Pipeline"**
4. Click **"OK"**
5. On the configuration page:

   **General section:**
   - Check **"GitHub project"**
   - Project url: `https://github.com/manjulachukki/TravelMemoryMainGE/`

   **Build Triggers section:**
   - Check **"GitHub hook trigger for GITScm polling"**

   **Pipeline section:**
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/manjulachukki/TravelMemoryMainGE.git`
   - Credentials: (none needed for public repos)
   - Branch Specifier: `*/manjula-capstone-sp1`
   - Script Path: `Jenkinsfile`

6. Click **"Save"**

---

### Step 8: Replace AWS Account ID in Kubernetes Manifests

**What you're doing:** The K8s deployment files reference your ECR (Docker image registry) URL which contains your AWS Account ID. You need to replace the placeholder with your actual ID.

#### Option A — Using sed on Linux/Mac (or on the EC2 instance):
```bash
# Navigate to the project directory
cd TravelMemoryMainGE

# Replace in backend deployment
sed -i 's/<AWS_ACCOUNT_ID>/123456789012/g' k8s/backend-deployment.yml

# Replace in frontend deployment
sed -i 's/<AWS_ACCOUNT_ID>/123456789012/g' k8s/frontend-deployment.yml

# Verify the replacement worked
grep "image:" k8s/backend-deployment.yml
# Expected: image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/travelmemory-backend:latest

grep "image:" k8s/frontend-deployment.yml
# Expected: image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/travelmemory-frontend:latest
```

#### Option B — Manual editing (if you prefer):
1. Open `k8s/backend-deployment.yml` in any text editor
2. Find the line containing `<AWS_ACCOUNT_ID>`
3. Replace `<AWS_ACCOUNT_ID>` with your actual 12-digit Account ID
4. Repeat for `k8s/frontend-deployment.yml`
5. Save both files

#### Commit and Push the Changes:
```bash
git add k8s/backend-deployment.yml k8s/frontend-deployment.yml
git commit -m "Replace AWS Account ID placeholder with actual account ID"
git push origin manjula-capstone-sp1
```

---

### Step 9: Run Terraform to Provision AWS Infrastructure

**What you're doing:** Running Terraform to automatically create all the AWS resources defined in the `terraform/` folder — VPC, subnets, EKS cluster, ECR repositories, etc.

#### 9.1 — Navigate to Terraform Directory
```bash
cd TravelMemoryMainGE/terraform
```

#### 9.2 — Initialize Terraform
```bash
terraform init

# What this does:
# - Downloads the AWS provider plugin
# - Connects to the S3 bucket for remote state storage
# - Prepares the working directory

# Expected output (last line):
# Terraform has been successfully initialized!
```

> **Troubleshooting:** If you get "Error configuring S3 Backend" — check that the S3 bucket from Step 3 exists and your AWS credentials are configured.

#### 9.3 — Preview What Terraform Will Create
```bash
terraform plan

# What this does:
# - Shows you EXACTLY what resources will be created/modified/destroyed
# - Does NOT make any changes — it's a dry run

# You'll see a long list like:
# Plan: 25 to add, 0 to change, 0 to destroy.
#
# Review the output to understand what's being created.
# Look for any errors (red text).
```

#### 9.4 — Apply the Terraform Configuration (Create Resources)
```bash
terraform apply

# Terraform will show the plan again and ask for confirmation:
# Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Only 'yes' will be accepted to approve.
#
# Enter a value: yes

# ⏱️ This will take 10-15 minutes (EKS cluster creation is slow)
# You'll see resources being created one by one.
#
# Expected final output:
# Apply complete! Resources: 25 added, 0 changed, 0 destroyed.
#
# Outputs:
# cluster_endpoint = "https://XXXXX.gr7.ap-south-1.eks.amazonaws.com"
# cluster_name = "travelmemory-eks"
# ecr_backend_url = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/travelmemory-backend"
# ecr_frontend_url = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/travelmemory-frontend"
```

> **IMPORTANT:** Save the outputs! You'll need `cluster_name` and `cluster_endpoint` later.

> **Troubleshooting Common Errors:**
> - "UnauthorizedAccess" → Your AWS credentials don't have sufficient permissions
> - "ResourceLimitExceeded" → Your AWS account has hit a service limit; request increase in AWS Console
> - "VpcLimitExceeded" → Delete unused VPCs in your account (each account can have 5 per region)

#### 9.5 — Verify Resources Were Created
```bash
# Check EKS cluster exists
aws eks describe-cluster --name travelmemory-eks --region ap-south-1 --query "cluster.status"
# Expected: "ACTIVE"

# Check ECR repositories exist
aws ecr describe-repositories --region ap-south-1 --query "repositories[].repositoryName"
# Expected: ["travelmemory-backend", "travelmemory-frontend"]
```

---

### Step 10: Update Ansible Inventory with Actual IP

**What you're doing:** Telling Ansible which server to configure. Since Jenkins is hosted externally (`https://jenkinsacademics.herovired.com`), the Ansible inventory should point to any EC2 worker nodes you manage (e.g., EKS worker nodes or a bastion host). If Ansible configuration is handled entirely within the Jenkins pipeline (which runs on the hosted Jenkins server), you may skip this step.

#### 10.1 — Get the Target EC2 Instance Public IP
```bash
# List running EC2 instances in your account
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].[Tags[?Key=='Name'].Value|[0],PublicIpAddress]" \
  --output table \
  --region ap-south-1
```

#### 10.2 — Edit the Ansible Inventory File
Open `ansible/inventory/dev/hosts.ini` and replace the placeholder IP with the target instance:

```ini
[webservers]
<YOUR_EC2_IP> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/jenkins-key.pem

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

> **Note:** If the hosted Jenkins pipeline handles Ansible execution, you'll need to ensure the SSH private key is added as a Jenkins credential (Kind: SSH Username with private key) so the pipeline can reach your target servers.

---

### Step 11: Connect kubectl to EKS and Create Kubernetes Secret

**What you're doing:** Configuring your local `kubectl` to communicate with the EKS cluster, creating the namespace, and storing the MongoDB connection string securely in Kubernetes.

#### 11.1 — Configure kubectl to Connect to EKS
```bash
# This command updates your local kubeconfig file to add the EKS cluster
aws eks update-kubeconfig --name travelmemory-eks --region ap-south-1

# Expected output:
# Added new context arn:aws:eks:ap-south-1:123456789012:cluster/travelmemory-eks to /home/user/.kube/config

# Verify connection to the cluster
kubectl get nodes

# Expected output (may take a minute for nodes to be Ready):
# NAME                                       STATUS   ROLES    AGE   VERSION
# ip-10-0-3-xxx.ap-south-1.compute.internal  Ready    <none>   10m   v1.28.x
# ip-10-0-4-xxx.ap-south-1.compute.internal  Ready    <none>   10m   v1.28.x
```

> **Troubleshooting:** If `kubectl get nodes` shows no nodes or errors:
> - Wait 5 minutes — nodes take time to join
> - Run `aws eks describe-cluster --name travelmemory-eks --region ap-south-1 --query "cluster.status"` — must be "ACTIVE"
> - Ensure your IAM user has EKS permissions

#### 11.2 — Create the Application Namespace
```bash
kubectl apply -f k8s/namespace.yml

# Expected output:
# namespace/travelmemory created

# Verify
kubectl get namespaces | grep travelmemory
# Expected: travelmemory   Active   5s
```

#### 11.3 — Create the MongoDB Secret
```bash
# Replace the connection string below with YOUR MongoDB Atlas connection string from Step 4
kubectl create secret generic travelmemory-secrets \
  --from-literal=mongo-uri="mongodb+srv://travelmemory-app:YOUR_PASSWORD@cluster0.xxxxx.mongodb.net/travelmemory?retryWrites=true&w=majority" \
  -n travelmemory

# Expected output:
# secret/travelmemory-secrets created

# Verify the secret exists (the value is hidden)
kubectl get secrets -n travelmemory
# Expected:
# NAME                   TYPE     DATA   AGE
# travelmemory-secrets   Opaque   1      5s
```

> **Security Note:** Never commit secrets to Git. The secret is stored encrypted in the Kubernetes cluster. The `backend-deployment.yml` references this secret to inject the MongoDB URI as an environment variable into the backend containers.

#### 11.4 — Also Configure kubectl on the Jenkins Server
Since you're using the hosted Jenkins at `https://jenkinsacademics.herovired.com`, kubectl configuration is handled automatically via the AWS credentials you added in Jenkins (Step 7.3). The pipeline's `aws eks update-kubeconfig` command in the Jenkinsfile will configure kubectl at runtime using those credentials.

---

### Step 12: Configure GitHub Webhook (Auto-trigger Jenkins on Push)

**What you're doing:** Setting up GitHub to notify Jenkins every time code is pushed. This triggers the pipeline automatically — no manual intervention needed.

#### 12.1 — Ensure Jenkins is Accessible from GitHub
- The hosted Jenkins at `https://jenkinsacademics.herovired.com` is already publicly accessible via HTTPS — no security group changes needed.

#### 12.2 — Add Webhook in GitHub
1. Go to your GitHub repository: `https://github.com/manjulachukki/TravelMemoryMainGE`
2. Click **Settings** tab (you need admin access to the repo)
3. Click **Webhooks** in the left sidebar
4. Click **"Add webhook"**
5. Fill in:
   - **Payload URL:** `https://jenkinsacademics.herovired.com/github-webhook/`
     - ⚠️ Don't forget the trailing `/`
   - **Content type:** `application/json`
   - **Which events would you like to trigger this webhook?**
     - Select: **"Just the push event"**
   - **Active:** ✅ (checked)
6. Click **"Add webhook"**

#### 12.3 — Test the Webhook
1. After adding, GitHub will send a ping event
2. Click on the webhook you just created
3. Scroll down to **"Recent Deliveries"**
4. You should see a delivery with a green checkmark (✓ = 200 response)
5. If you see a red X, check:
   - Is the Payload URL correct (with trailing `/`)?
   - Is the Jenkins job configured with "GitHub hook trigger for GITScm polling"?

---

### Step 13: Run the Pipeline (First Execution)

**What you're doing:** Triggering the full CI/CD pipeline for the first time to build, deploy, and verify everything works end-to-end.

#### 13.1 — Trigger Manually (Recommended for First Run)
1. Go to Jenkins: **https://jenkinsacademics.herovired.com**
2. Click on `travelmemory-pipeline`
3. Click **"Build Now"** (left sidebar)
4. Click on the build number (e.g., `#1`) in the build history
5. Click **"Console Output"** to watch the pipeline in real-time

#### 13.2 — What Each Stage Does (Follow Along)
```
Stage 1 - Checkout:           Downloads code from GitHub (~10 seconds)
Stage 2 - Build Docker:       Builds both Docker images (~2-5 minutes)
Stage 3 - Terraform:          Provisions infrastructure (~1 minute if already exists)
Stage 4 - Ansible:            Configures servers (~2-3 minutes)
Stage 5 - Deploy to EKS:      Applies K8s manifests (~1-2 minutes)
Stage 6 - Deploy Monitoring:  Deploys Prometheus + Grafana (~1 minute)
Stage 7 - Verify:             Checks rollout status (~30 seconds)
```

#### 13.3 — Trigger via Push (Automatic)
After the first successful run, test automatic triggering:
```bash
# Make any small change (e.g., add a comment to README)
echo "" >> README.md
git add README.md
git commit -m "Test: trigger pipeline via webhook"
git push origin manjula-capstone-sp1

# Jenkins should auto-trigger within 10-30 seconds
# Check Jenkins dashboard — you should see a new build starting
```

---

### Step 14: Verify the Deployed Application

**What you're doing:** Confirming that the application is running correctly on EKS and accessible via the internet.

#### 14.1 — Check All Pods Are Running
```bash
kubectl get pods -n travelmemory

# Expected output (all pods should be "Running" with "1/1" READY):
# NAME                                READY   STATUS    RESTARTS   AGE
# backend-deployment-xxxxx-yyyyy      1/1     Running   0          5m
# backend-deployment-xxxxx-zzzzz      1/1     Running   0          5m
# frontend-deployment-xxxxx-yyyyy     1/1     Running   0          5m
# frontend-deployment-xxxxx-zzzzz     1/1     Running   0          5m
```

> **Troubleshooting Pod Issues:**
> ```bash
> # If STATUS is "CrashLoopBackOff" or "Error":
> kubectl logs <pod-name> -n travelmemory
> 
> # If STATUS is "ImagePullBackOff":
> # The ECR image doesn't exist or credentials are wrong
> kubectl describe pod <pod-name> -n travelmemory
> 
> # If STATUS is "Pending":
> # Not enough resources on nodes
> kubectl describe pod <pod-name> -n travelmemory | grep -A5 "Events"
> ```

#### 14.2 — Get the Application URL
```bash
kubectl get svc frontend-service -n travelmemory

# Expected output:
# NAME               TYPE           CLUSTER-IP    EXTERNAL-IP                                    PORT(S)        AGE
# frontend-service   LoadBalancer   10.100.x.x    xxxxx.ap-south-1.elb.amazonaws.com            80:30xxx/TCP   5m

# The EXTERNAL-IP is your application URL!
# ⚠️ It may show "<pending>" for 2-3 minutes while the Load Balancer is being created
```

#### 14.3 — Access the Application
1. Copy the `EXTERNAL-IP` value (the long `.elb.amazonaws.com` URL)
2. Open it in your browser: `http://xxxxx.ap-south-1.elb.amazonaws.com`
3. You should see the TravelMemory application homepage!

#### 14.4 — Access Monitoring
```bash
kubectl get svc -n monitoring

# Expected output:
# NAME                  TYPE           CLUSTER-IP    EXTERNAL-IP                              PORT(S)          AGE
# prometheus-service    LoadBalancer   10.100.x.x    yyyyy.ap-south-1.elb.amazonaws.com      9090:30xxx/TCP   5m
# grafana-service       LoadBalancer   10.100.x.x    zzzzz.ap-south-1.elb.amazonaws.com      3000:30xxx/TCP   5m
```

- **Prometheus:** Open `http://<prometheus-EXTERNAL-IP>:9090`
  - Try a query: Type `up` in the query box and click "Execute"
  - You should see a list of targets with value `1` (meaning they're up)

- **Grafana:** Open `http://<grafana-EXTERNAL-IP>:3000`
  - Login: Username = `admin`, Password = `admin123`
  - You'll be prompted to change the password (you can skip for dev)
  - Go to **Dashboards → Import** to add pre-built dashboards
  - Recommended dashboard IDs to import:
    - `3119` — Kubernetes cluster monitoring
    - `6417` — Kubernetes pods monitoring

---

### Step 15: Cleanup (When You're Done — SAVE MONEY!)

**What you're doing:** Destroying all AWS resources to avoid ongoing charges. EKS costs ~$0.10/hour even when idle.

> ⚠️ **Only do this when you're completely done with the project!**

```bash
# Step 1: Delete Kubernetes resources first
kubectl delete -f monitoring/ --ignore-not-found
kubectl delete -f k8s/ --ignore-not-found

# Step 2: Destroy Terraform infrastructure
cd terraform
terraform destroy

# Terraform will show what will be destroyed and ask for confirmation:
# Do you really want to destroy all resources?
# Enter: yes

# ⏱️ Takes 10-15 minutes (EKS deletion is slow)

# Step 3: (Optional) Delete the S3 state bucket
aws s3 rb s3://travelmemory-terraform-state --force

# Step 4: (Optional) Terminate the Jenkins EC2 instance
# Go to AWS Console → EC2 → select Jenkins-Server → Instance state → Terminate
```

---

## Troubleshooting Guide

### Common Issues and Solutions

| Problem | Possible Cause | Solution |
|---------|---------------|----------|
| `terraform init` fails with S3 error | Bucket doesn't exist or wrong region | Create bucket per Step 3, check region matches |
| Jenkins can't pull from GitHub | Repository is private | Add GitHub credentials in Jenkins or make repo public |
| Docker build fails | Dockerfile syntax error or missing files | Run `docker build .` locally to debug |
| `kubectl` times out | kubeconfig not set or cluster not ready | Run `aws eks update-kubeconfig` again |
| Pods in `ImagePullBackOff` | ECR image not pushed or wrong account ID | Check ECR repo has images: `aws ecr list-images` |
| Pods in `CrashLoopBackOff` | Application error (wrong env vars, DB connection) | Check logs: `kubectl logs <pod> -n travelmemory` |
| LoadBalancer stuck in `<pending>` | Security group or subnet tag issues | Check subnets have `kubernetes.io/role/elb` tag |
| Jenkins build hangs at Terraform | Previous state lock exists | Run `terraform force-unlock <LOCK_ID>` |
| MongoDB connection refused | Wrong connection string or IP not whitelisted | Verify Atlas Network Access allows `0.0.0.0/0` |
| Webhook not triggering Jenkins | Wrong URL, port not open, or trailing `/` missing | Test webhook delivery in GitHub settings |

### Useful Debugging Commands
```bash
# Check pod status and events
kubectl describe pod <pod-name> -n travelmemory

# View pod logs (stdout)
kubectl logs <pod-name> -n travelmemory

# View previous container's logs (if it crashed)
kubectl logs <pod-name> -n travelmemory --previous

# Get all resources in namespace
kubectl get all -n travelmemory

# Check HPA status
kubectl get hpa -n travelmemory

# Check node resources
kubectl top nodes

# Check pod resource usage
kubectl top pods -n travelmemory

# Force restart a deployment
kubectl rollout restart deployment/<deployment-name> -n travelmemory

# Check Terraform state
cd terraform && terraform state list

# Verify ECR images exist
aws ecr list-images --repository-name travelmemory-backend --region ap-south-1
aws ecr list-images --repository-name travelmemory-frontend --region ap-south-1
```

---

## Cost Optimization Notes (10% evaluation criteria)

| Decision | Reason | Estimated Savings |
|----------|--------|-------------------|
| `t3.medium` instances | Burstable, cost-effective for dev/staging | ~60% cheaper than `m5.large` |
| EKS node scaling (min 1, max 3) | Scales down during low traffic | ~50% savings during off-hours |
| HPA (2→5 pods) | Scales based on actual utilization, not peak capacity | Avoids over-provisioning |
| Single NAT Gateway (dev only) | Multi-AZ NAT costs 2x — only needed for production | Saves ~$32/month |
| ECR `scan_on_push` | Built-in vulnerability scanning; no need for 3rd-party tools | Saves tool licensing costs |
| Prometheus `emptyDir` | No EBS volumes for dev; metrics are temporary | Saves ~$10/month per volume |
| Use `t3.medium` for Jenkins | Burstable — only needs full CPU during builds | ~40% cheaper than `t3.large` |

### Monthly Cost Estimate (Dev Environment)
| Resource | Cost/Month (approximate) |
|----------|-------------------------|
| EKS Control Plane | $73 |
| 2x t3.medium nodes | $60 |
| NAT Gateway | $32 + data transfer |
| Application Load Balancer | $16 + data transfer |
| Jenkins EC2 (t3.medium) | $30 |
| **Total** | **~$211/month** |

> **Tip:** Stop the Jenkins EC2 instance when not in use. EKS control plane charges even when idle — destroy with `terraform destroy` when done learning.
