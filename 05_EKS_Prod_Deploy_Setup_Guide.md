# DevOps CI/CD Pipeline with EKS Deployment - Complete Setup Documentation

## Project Overview
This documentation covers the complete setup of a DevOps CI/CD pipeline infrastructure using AWS services, Jenkins, SonarQube, and Kubernetes (EKS). The setup includes automated security scanning, quality gates, and deployment to a production Kubernetes cluster.

## Architecture Components
- **AWS EKS Cluster** - Managed Kubernetes cluster for application deployment
- **Jenkins** - CI/CD automation server with security integrations
- **SonarQube** - Code quality analysis and security scanning
- **Security Tools** - GitLeaks for secrets scanning, Trivy for vulnerability assessment
- **Infrastructure as Code** - Terraform for AWS resource provisioning

---

## Phase 1: AWS Infrastructure Setup

### 1.1 Installer VM Instance
**Purpose:** Primary instance for EKS cluster management and Terraform operations

**Instance Configuration:**
- **Name:** Installer-vm
- **OS:** Ubuntu
- **Instance Type:** t2.medium
- **Key Pair:** k8s-deploy
- **Security Ports:** 3000-9000 (opened)
- **Storage:** 25GB EBS volume

### 1.2 Jenkins Instance
**Purpose:** CI/CD automation server with integrated security tools

**Instance Configuration:**
- **Name:** Jenkins Instance
- **OS:** Ubuntu
- **Instance Type:** t2.medium
- **Key Pair:** k8s-deploy (same as installer-vm)
- **Security Ports:** 8080, 9000
- **Storage:** 30GB EBS volume

### 1.3 SonarQube Instance
**Purpose:** Code quality analysis and security vulnerability scanning

**Instance Configuration:**
- **Name:** SonarQube Instance
- **OS:** Ubuntu
- **Instance Type:** t2.medium
- **Key Pair:** k8s-deploy (same as installer-vm)
- **Security Ports:** 8080, 9000
- **Storage:** 30GB EBS volume

---

## Phase 2: EKS Cluster Setup (Installer-VM)

### 2.1 System Prerequisites Installation

**System Update:**
```bash
sudo apt update
```

**AWS CLI v2 Installation:**
```bash
# Download AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Extract and install
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
```

**kubectl Installation:**
```bash
# Download latest kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make executable and move to PATH
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

**eksctl Installation:**
```bash
# Download eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# Move to PATH
sudo mv /tmp/eksctl /usr/local/bin

# Verify installation
eksctl version
```

### 2.2 AWS Configuration

**Configure AWS Credentials:**
```bash
aws configure
```
**Required Input:**
- AWS Access Key ID
- AWS Secret Access Key
- Default region: ap-south-1
- Default output format: json

### 2.3 Terraform Infrastructure Deployment

**Clone Project Repository:**
```bash
git clone https://github.com/jaiswaladi246/Mega-Project-Terraform.git
```

**Terraform Installation:**
```bash
# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update and install Terraform
sudo apt update && sudo apt install terraform

# Verify installation
terraform version
```

**Deploy Infrastructure:**
```bash
# Navigate to project directory
cd Mega-Project-Terraform

# Initialize Terraform (before procced replace default = "DevOps-Shack" with default = "<Your Key-pair for this instance>")
terraform init

# Plan deployment
terraform plan

# Apply configuration
terraform apply -auto-approve
```

### 2.4 EKS Cluster Configuration

**Update kubeconfig:**
```bash
aws eks --region ap-south-1 update-kubeconfig --name devopsshack-cluster
```

**Verify Cluster Connection:**
```bash
kubectl cluster-info
kubectl get nodes
```

### 2.5 EKS Add-ons Configuration

**OIDC Provider Setup:**
```bash
eksctl utils associate-iam-oidc-provider --region ap-south-1 --cluster devopsshack-cluster --approve
```

**EBS CSI Driver Service Account:**
```bash
eksctl create iamserviceaccount \
--region ap-south-1 \
--name ebs-csi-controller-sa \
--namespace kube-system \
--cluster devopsshack-cluster \
--attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--approve \
--override-existing-serviceaccounts
```

**Install Kubernetes Components:**

**EBS CSI Driver:**
```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/ecr/?ref=release-1.11"
```

**NGINX Ingress Controller:**
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

**Cert-Manager:**
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

**Verify Installation:**
```bash
kubectl get pods -n ingress-nginx
kubectl get pods -n cert-manager
kubectl get pods -n kube-system
```

<img width="1931" height="1000" alt="everything up and running at installer-vm" src="https://github.com/user-attachments/assets/56e9c9d3-22c8-4dda-97b6-38f3e0b36b9a" />

---

## Phase 3: Jenkins Server Setup

### 3.1 Jenkins Installation

**System Update:**
```bash
sudo apt-get update
```

**Java Installation (Jenkins Prerequisite):**
```bash
sudo apt install openjdk-11-jdk -y
java -version
```

**Jenkins LTS Installation:**
```bash
# Add Jenkins repository key
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -

# Add Jenkins repository
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

# Update package list and install Jenkins
sudo apt-get update
sudo apt-get install jenkins -y

# Start and enable Jenkins service
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

**kubectl Installation:**
```bash
# Download latest kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make executable and move to PATH
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

**Retrieve Initial Admin Password:**
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
**Access:** Jenkins is accessible at `http://<jenkins-instance-ip>:8080`

### 3.2 Jenkins Plugin Installation

**Required Plugins:**
- Stage View
- SonarQube Scanner
- NodeJS
- Docker Pipeline
- Kubernetes
- Kubernetes CLI
- Kubernetes Credentials

**Installation Path:** Manage Jenkins → Plugins → Available Plugins

### 3.3 Jenkins Tools Configuration

**Configure Tools (Manage Jenkins → Tools):**

**SonarQube Scanner:**
- Name: sonar-scanner
- Version: Latest (Install automatically)

**NodeJS:**
- Name: nodejs16
- Install automatically: ✓
- Install from nodejs.org
- Version: NodeJS 16.1.0

### 3.4 Docker Installation on Jenkins

```bash
# Add Docker's official GPG key
sudo apt-get update
sudo apt-get install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update and install Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Add jenkins user to docker group
sudo usermod -aG docker jenkins
```

**Restart Jenkins:** Visit `http://<jenkins-ip>:8080/restart` to apply Docker permissions

### 3.5 Security Tools Installation

**GitLeaks (Secret Scanning):**
```bash
sudo apt install gitleaks
```

**Trivy (Vulnerability Assessment):**
```bash
# Install prerequisites
sudo apt-get install wget apt-transport-https gnupg lsb-release -y

# Add Trivy repository key
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -

# Add Trivy repository
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list

# Update and install Trivy
sudo apt-get update
sudo apt-get install trivy
```

---

## Phase 4: SonarQube Server Setup

### 4.1 SonarQube Installation

**System Update:**
```bash
sudo apt-get update
```

**Docker Installation:**
```bash
# Install Docker prerequisites
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update and install Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

**Configure Docker Permissions:**
```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

**Deploy SonarQube Container:**
```bash
docker run -d -p 9000:9000 sonarqube:lts-community
docker ps
```
**Access:** SonarQube is accessible at `http://<sonarqube-instance-ip>:9000`

---

## Phase 5: Integration and Configuration

### 5.1 SonarQube-Jenkins Integration

**Step 1: Generate SonarQube Authentication Token**
- Access SonarQube: Administration → Security → Users → Tokens
- Generate new token and copy it

**Step 2: Configure SonarQube Token in Jenkins**
- Path: Manage Jenkins → Credentials → Global → Add Credentials
- Configuration:
  - Kind: Secret text
  - Secret: [SonarQube token]
  - ID: sonar-token
  - Description: sonar-token

**Step 3: Configure SonarQube Server in Jenkins**
- Path: Manage Jenkins → System → SonarQube servers
- Configuration:
  - Name: sonar
  - Server URL: `http://<sonarqube-ip>:9000`
  - Server authentication token: sonar-token

### 5.2 Docker Hub Integration

**Configure Docker Hub Credentials:**
- Path: Manage Jenkins → Credentials → Global → Add Credentials
- Configuration:
  - Kind: Username with password
  - Scope: Global
  - Username: ansuuman0506
  - Password: [Docker Hub token]
  - ID: docker-cred
  - Description: docker-cred
<img width="960" height="540" alt="Credentials" src="https://github.com/user-attachments/assets/5bc02253-a005-4ba5-a1c8-361a39a5859c" />

### 5.3 SonarQube Webhook Configuration

**Configure Quality Gate Webhook:**
- Path: SonarQube → Administration → Configuration → Webhooks
- Create Webhook:
  - Name: SonarQube-Webhook
  - URL: `http://13.201.88.16:8080/sonarqube-webhook/`

<img width="960" height="540" alt="Webhook-Configuration" src="https://github.com/user-attachments/assets/9d4044a1-1fb4-49d5-9964-78608735c300" />

---

## Phase 6: Kubernetes RBAC Configuration

### 6.1 RBAC Setup (Production Namespace)

**Create RBAC Directory:**
```bash
# On Installer-VM
mkdir rbac
cd rbac
```

**RBAC Configuration Reference:**
- Follow: `3-Tier-DevSecOps-Project/04_EKS_Dev_Deploy_Setup_Guide.md`
- **Important:** Replace namespace `'dev'` with `'prod'` in all configurations

**Apply RBAC Configurations:**
```bash
kubectl apply -f rbac/sa.yaml -n prod
kubectl apply -f rbac/role.yaml -n prod
kubectl apply -f rbac/rb.yaml -n prod
kubectl apply -f rbac/cr.yaml -n prod
kubectl apply -f rbac/crb.yaml -n prod
kubectl apply -f rbac/secret.yaml -n prod
```

<img width="1920" height="1080" alt="RBAC-CREATED" src="https://github.com/user-attachments/assets/8d9e3e99-46d2-42db-b8c5-42aa994e0ca3" />

### 6.2 Service Account Token Configuration

**Extract Service Account Token:**
```bash
kubectl describe secret <secret-name-from-secret.yaml> -n prod
```

**Configure Kubernetes Token in Jenkins:**
- Path: Manage Jenkins → Credentials → Global → Add Credentials
- Configuration:
  - Kind: Secret text
  - Scope: Global
  - Secret: [Kubernetes service account token]
  - ID: k8-prod-token
  - Description: k8-prod-token

---

## Phase 7: CI/CD Pipeline Configuration

### 7.1 Pipeline Setup

**Create Jenkins Pipeline:**
- Path: Jenkins → New Item → Pipeline
- Configuration:
  - Pipeline Script: [Import from repository]
  - **Critical:** Change branch reference from `master` to `main`

### 7.2 Pipeline Components

**Security Scanning Integration:**
- **GitLeaks:** Secret detection in source code
- **Trivy:** Container vulnerability scanning
- **SonarQube:** Code quality and security analysis

**Quality Gates:**
- SonarQube quality gate integration with webhook
- Automated build failure on quality gate failure

**Deployment Target:**
- EKS cluster in `prod` namespace
- RBAC-controlled access with service account authentication
<img width="960" height="540" alt="Pipeline-build-Done" src="https://github.com/user-attachments/assets/8f7ad2fc-c334-4b00-9f41-6a2866b21f19" />

---

## Final Infrastructure Summary

### Deployed Components

1. **Installer-VM (t2.medium, 25GB)**
   - EKS cluster management tools (kubectl, eksctl, terraform)
   - AWS CLI configuration
   - RBAC configurations for production namespace

2. **Jenkins Server (t2.medium, 30GB)**
   - Jenkins LTS with essential plugins
   - Docker integration for containerized builds
   - Security tools: GitLeaks, Trivy
   - Integration with SonarQube and Kubernetes
   - Credentials management for all integrations

3. **SonarQube Server (t2.medium, 30GB)**
   - SonarQube LTS Community Edition
   - Docker-based deployment
   - Webhook integration with Jenkins
   - Quality gate enforcement

4. **AWS EKS Cluster**
   - Managed Kubernetes cluster: devopsshack-cluster
   - Production namespace with RBAC controls
   - EBS CSI Driver for persistent storage
   - NGINX Ingress Controller
   - Cert-Manager for TLS certificate management

### Security Features

- **Secrets Management:** GitLeaks scanning for leaked credentials
- **Vulnerability Assessment:** Trivy scanning for container vulnerabilities
- **Code Quality:** SonarQube static analysis and security rules
- **Access Control:** Kubernetes RBAC with least-privilege service accounts
- **Secure Communication:** TLS certificate management with cert-manager

### Automation Capabilities

- **Continuous Integration:** Automated builds triggered by code changes
- **Quality Gates:** Automated quality checks with build failure on violations
- **Security Scanning:** Integrated security scanning in pipeline
- **Container Registry:** Integration with Docker Hub for image management
- **Deployment Automation:** Automated deployment to production Kubernetes namespace

---

## Phase 8: Continuous Delivery and Deployment Configuration

### 8.1 Continuous Delivery Setup (Manual Approval Process)

#### Plugin Installation
**Required Plugin:** Generic Webhook Trigger 2.3.1
- Path: Manage Jenkins → Plugins → Available Plugins
- Install: Generic Webhook Trigger plugin

#### Pipeline Webhook Configuration

**Step 1: Enable Generic Webhook Trigger**
- Navigate to: Pipeline → Configure
- Scroll to Generic Webhook Trigger section
- Check the checkbox for Generic Webhook Trigger

**Step 2: Configure Post Parameters**
- In Post Parameters section, add:
  - **Variable:** `ref`
  - **Expression (JSON):** `$.ref`
- Purpose: Extract branch reference from GitHub webhook payload

**Step 3: Token Configuration**
- Create a custom keyword string as token
- **Payload URL Format:** `http://JENKINS_URL/generic-webhook-trigger/invoke?token=TOKEN_HERE`
- Replace `TOKEN_HERE` with your custom keyword string

**Step 4: Configure Optional Filter**
- **Expression:** `refs/heads/main`
- **Text:** `$ref`
- Purpose: Filter to trigger only on main branch pushes

#### GitHub Webhook Integration

**Configure Repository Webhook:**
1. Go to GitHub repository settings
2. Navigate to Webhooks section
3. Click "Add webhook"

**Webhook Configuration:**
- **Payload URL:** `http://JENKINS_URL/generic-webhook-trigger/invoke?token=TOKEN_HERE`
- **Content type:** application/json
- **SSL verification:** Disable (not recommended for production)
- **Trigger events:** Just the push event
- Click "Add webhook"

**Testing:** Modify repository content to verify automatic pipeline triggering(before test remove these stage('K8s-Deploy') and stage('K8s-Verify') from pipeline)
<img width="960" height="540" alt="Automatic-Triggers" src="https://github.com/user-attachments/assets/b910cf4c-1c1b-49e7-a9a0-5c6b5ef7500f" />


### 8.2 Manual Approval Gate Implementation

#### Pipeline Stage Addition
**Stage Location:** After `'Build-Tag & Push Frontend Docker Image'` stage

**Manual Approval Stage Configuration:**
```groovy
stage('Manual Approval for Production') {
    steps {
        timeout(time: 1, unit: 'HOURS') {
            input message: 'Approve deployment to PRODUCTION?', ok: 'Deploy'
        }
    }
}
```

**Stage Features:**
- **Timeout:** 1 hour maximum wait time
- **User Interaction:** Manual approval required with "Deploy" button
- **Pipeline Behavior:** Pipeline pauses until approval or timeout
- **Fail-Safe:** Pipeline fails if no approval within timeout period
<img width="960" height="540" alt="Screenshot 2025-09-01 at 11 53 07 AM" src="https://github.com/user-attachments/assets/fcec74a9-1b9d-4981-a605-fc34498b6013" />

### 8.3 Continuous Delivery Workflow

#### Complete Pipeline Flow
```
Code Push (GitHub) → Webhook Trigger → Automated CI Stages → Manual Approval Gate → Production Deployment
```

**Automated Stages (Before Manual Approval):**
1. Source code checkout
2. Security scanning (GitLeaks, Trivy)
3. Code quality analysis (SonarQube)
4. Quality gate validation
5. Docker image build and push
6. **Manual Approval Gate** ← Human intervention required

**Post-Approval Stages:**
1. Kubernetes deployment to prod namespace
2. Service exposure and configuration
3. Deployment verification

#### Deployment Types Implemented

**Type 1: Continuous Delivery (Manual)**
- **Trigger:** Automatic (GitHub webhook)
- **CI Stages:** Automatic execution
- **Deployment:** Manual approval required
- **Control:** Human gate before production
- **Use Case:** Production deployments requiring human oversight

**Type 2: Continuous Deployment (Automatic)** - *Prepared for future implementation*
- **Trigger:** Automatic (GitHub webhook)
- **CI Stages:** Automatic execution
- **Deployment:** Fully automatic
- **Control:** Quality gates only
- **Use Case:** Development/staging environments

---

## Updated Pipeline Architecture

### Security and Quality Integration
- **Secret Scanning:** GitLeaks integration in pipeline
- **Vulnerability Assessment:** Trivy container scanning
- **Code Quality:** SonarQube static analysis with quality gates
- **Manual Control:** Production deployment approval process

### Infrastructure Integration
- **Source Control:** GitHub with webhook integration
- **CI/CD Server:** Jenkins with automated triggering
- **Quality Analysis:** SonarQube with webhook feedback
- **Container Registry:** Docker Hub integration
- **Deployment Target:** AWS EKS prod namespace
- **Access Control:** Kubernetes RBAC with service accounts

### Monitoring and Control Points
1. **GitHub Push Event** - Automatic trigger point
2. **Security Scan Results** - Automated failure points
3. **Quality Gate Status** - SonarQube integration checkpoint
4. **Manual Approval** - Human control point
5. **Deployment Status** - Kubernetes deployment verification

---

## Next Steps

1. **Pipeline Testing:** Execute initial pipeline build to validate webhook integration
2. **Manual Approval Testing:** Test the approval process and timeout behavior
3. **Continuous Deployment Setup:** Implement automatic deployment for non-production environments
4. **Monitoring Setup:** Implement monitoring and logging solutions
5. **Backup Configuration:** Set up automated backup strategies
6. **Documentation:** Create operational runbooks and troubleshooting guides

---

## Troubleshooting Notes

- **Jenkins Restart:** Required after Docker installation to apply group permissions
- **Service Account Token:** Must be extracted from Kubernetes secret and configured in Jenkins
- **Webhook URL:** Must include trailing slash for proper SonarQube integration
- **Namespace Consistency:** Ensure all RBAC configurations use 'prod' namespace
- **Branch Configuration:** Pipeline must reference 'main' branch, not 'master'

---

*This documentation represents the complete setup of a production-ready DevOps CI/CD pipeline with integrated security scanning and quality gates, deployed on AWS infrastructure with Kubernetes orchestration.*
