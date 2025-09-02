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

# Initialize Terraform (before procced at variable.tf replace default = "DevOps-Shack" with default = "<Your Key-pair for this instance>")
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

## Phase 9: Production Kubernetes Manifests Development

### 9.1 3-Tier Application Architecture

**Manifest Directory Structure:**
```
3-Tier-DevSecOps-Project/k8s-prod/
├── backend.yaml      # API service deployment
├── ci.yaml           # Certificate issuer configuration
├── frontend.yaml     # Web application deployment
├── ingress.yaml      # External routing and SSL
├── mysql.yaml        # Database persistence layer
└── sc.yaml          # Storage class configuration
```

### 9.2 Database Layer (mysql.yaml)

**MySQL StatefulSet Configuration:**
- **Image:** mysql:8
- **Replicas:** 1 (single instance for consistency)
- **Storage:** 5Gi EBS volume with ebs-sc StorageClass
- **Persistence:** StatefulSet with volumeClaimTemplates

**Security Configuration:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: prod
stringData:
  MYSQL_ROOT_PASSWORD: Aditya
```

**Database Configuration:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: prod
data:
  MYSQL_DATABASE: crud_app
```

**Database Schema Initialization:**
```sql
CREATE TABLE IF NOT EXISTS users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  role ENUM('admin', 'viewer') NOT NULL DEFAULT 'viewer',
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Service Configuration:**
- **Type:** Headless service (ClusterIP: None)
- **Port:** 3306 for MySQL communication
- **Service Discovery:** Internal DNS resolution

### 9.3 Application Layer (backend.yaml)

**Backend API Configuration:**
- **Image:** anshuman0506/backend:latest
- **Replicas:** 3 (high availability)
- **Port:** 5000 (REST API endpoint)

**Database Integration:**
```yaml
env:
  - name: DB_HOST
    value: mysql
  - name: DB_USER
    value: root
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: MYSQL_ROOT_PASSWORD
  - name: DB_NAME
    valueFrom:
      configMapKeyRef:
        name: mysql-config
        key: MYSQL_DATABASE
```

**Service Configuration:**
- **Type:** ClusterIP (internal communication)
- **Port:** 5000 → 5000 mapping

### 9.4 Presentation Layer (frontend.yaml)

**Frontend Web Application:**
- **Image:** anshuman0506/frontend:latest
- **Replicas:** 3 (load distribution)
- **Port:** 80 (HTTP web server)

**Service Configuration:**
- **Type:** ClusterIP (internal communication)
- **Port:** 80 → 80 mapping

### 9.5 Ingress and SSL Configuration

**Domain Configuration (ingress.yaml):**
- **Primary Domain:** 05anshuman.com
- **Secondary Domain:** www.05anshuman.com

**Routing Rules:**
```yaml
paths:
  - path: /api
    pathType: Prefix
    backend:
      service:
        name: backend-svc
        port:
          number: 5000
  - path: /
    pathType: Prefix
    backend:
      service:
        name: frontend-svc
        port:
          number: 80
```

**SSL/TLS Configuration:**
```yaml
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

**Certificate Issuer (ci.yaml):**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: anshuman.mohapatra04@gmail.com
```

### 9.6 Storage Configuration (sc.yaml)

**EBS Storage Class:**
```yaml
apiVersion: storage.k8s.io/v1 
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

---

## Phase 10: Production Deployment Automation

### 10.1 Enhanced Pipeline Stages

**Deployment Stage Configuration:**
```groovy
stage('Deployment To Prod') {
    steps {
        script {
            withKubeConfig(
                caCertificate: '', 
                clusterName: 'devopsshack-cluster', 
                contextName: '', 
                credentialsId: 'k8-prod-token', 
                namespace: 'prod', 
                restrictKubeConfigAccess: false, 
                serverUrl: 'https://3AC42B30506471B63B6AEBF1CD95ADF2.gr7.ap-south-1.eks.amazonaws.com'
            ) {
                sh 'kubectl apply -f k8s-prod/sc.yaml'
                sleep 20
                sh 'kubectl apply -f k8s-prod/mysql.yaml -n prod'
                sh 'kubectl apply -f k8s-prod/backend.yaml -n prod'
                sh 'kubectl apply -f k8s-prod/frontend.yaml -n prod'
                sh 'kubectl apply -f k8s-prod/ci.yaml'
                sh 'kubectl apply -f k8s-prod/ingress.yaml -n prod'
                sleep 30
            }
        }
    }
}
```

**Verification Stage Configuration:**
```groovy
stage('Verify Deployment To Prod') {
    steps {
        script {
            withKubeConfig(
                caCertificate: '', 
                clusterName: 'devopsshack-cluster', 
                contextName: '', 
                credentialsId: 'k8-prod-token', 
                namespace: 'prod', 
                restrictKubeConfigAccess: false, 
                serverUrl: 'https://3AC42B30506471B63B6AEBF1CD95ADF2.gr7.ap-south-1.eks.amazonaws.com'
            ) {
                sh 'kubectl get pods -n prod'
                sleep 20
                sh 'kubectl get ingress -n prod'
            }
        }
    }
}
```

### 10.2 Deployment Issues and Resolutions

#### Issue 1: EBS Storage Provisioning
**Problem:** `failed to provision volume with StorageClass "ebs-sc": no EC2 IMDS role found`
**Root Cause:** EBS CSI driver lacking proper IAM permissions
**Impact:** MySQL StatefulSet couldn't create persistent storage

**Diagnostic Commands:**
```bash
kubectl get pvc -n prod
kubectl get storageclass
kubectl get events -n prod --sort-by='.lastTimestamp'
```

#### Issue 2: StatefulSet Immutable Fields
**Problem:** `The StatefulSet "mysql" is invalid: spec: Forbidden: updates to statefulset spec`
**Root Cause:** StatefulSet `volumeClaimTemplates` cannot be modified after creation
**Resolution:** Implemented StatefulSet and PVC deletion before recreation

**Temporary Solution Applied:**
```groovy
// Delete existing StatefulSet and PVC to allow recreation
sh 'kubectl delete statefulset mysql -n prod --ignore-not-found=true'
sh 'kubectl delete pvc mysql-persistent-storage-mysql-0 -n prod --ignore-not-found=true'
sleep 10
```

#### Issue 3: Incorrect EKS Cluster Endpoint
**Problem:** Verification stage authentication failure
**Resolution Commands:**
```bash
kubectl cluster-info
aws eks describe-cluster --name devopsshack-cluster --region ap-south-1 --query 'cluster.endpoint' --output text
```

### 10.3 Successful Deployment Results

**Production Pods Status:**
```
NAME                        READY   STATUS    RESTARTS   AGE
backend-556bb9c668-28t8p    1/1     Running   0          46m
backend-556bb9c668-mxcph    1/1     Running   0          46m
backend-556bb9c668-z5qh8    1/1     Running   0          46m
frontend-6c8dbd5864-bnswh   1/1     Running   0          46m
frontend-6c8dbd5864-gmkrh   1/1     Running   0          46m
frontend-6c8dbd5864-rg6zk   1/1     Running   0          46m
mysql-0                     1/1     Running   0          7m22s
```

**Ingress Configuration:**
```
NAME                        HOSTS                               ADDRESS
user-management-ingress     05anshuman.com,www.05anshuman.com   aaa8b0a7955434d64a979c7d425fa1c6-361776107.ap-south-1.elb.amazonaws.com
```
<img width="1920" height="1080" alt="Screenshot 2025-09-01 at 1 43 07 PM" src="https://github.com/user-attachments/assets/02d3d107-3558-4059-b15f-bdf665bcd92c" />
<img width="814" height="435" alt="Screenshot 2025-09-01 at 1 15 45 PM" src="https://github.com/user-attachments/assets/35616953-45f7-4b37-b53e-2338fc4bd1f2" />
<img width="960" height="540" alt="Screenshot 2025-09-01 at 1 22 25 PM" src="https://github.com/user-attachments/assets/3cac372e-f952-4424-9650-83edf5175f1d" />

### 10.4 DNS Configuration and Domain Setup

**Load Balancer Resolution:**
```bash
nslookup aaa8b0a7955434d64a979c7d425fa1c6-361776107.ap-south-1.elb.amazonaws.com
# Result: 13.234.94.234
```

**GoDaddy DNS Configuration:**
- **A Record:** 13.234.94.234 (Load Balancer IP)
- **CNAME Record:** aaa8b0a7955434d64a979c7d425fa1c6-361776107.ap-south-1.elb.amazonaws.com

**DNS Verification:**
- **Tool:** whatsmydns.net
- **Verification:** A record and CNAME record propagation

### 10.5 Final Pipeline Reversion

**Current Deployment Stage (Reverted):**
- Removed StatefulSet/PVC deletion commands
- Simplified deployment approach
- Relies on clean initial deployment or manual cleanup

---

## Updated Complete Infrastructure Architecture

### End-to-End Production Workflow
1. **Code Push** → GitHub repository (main branch)
2. **Webhook Trigger** → Jenkins pipeline activation
3. **Automated CI** → Security scanning (GitLeaks, Trivy) + Quality analysis (SonarQube)
4. **Quality Gates** → SonarQube webhook validation
5. **Manual Approval** → Human production deployment gate
6. **Kubernetes Deployment** → 3-tier application to EKS prod namespace
7. **Verification** → Pod and ingress status validation
8. **Live Application** → https://05anshuman.com with SSL certificates

### Production Environment Components
- **Frontend:** 3-replica React/web application
- **Backend:** 3-replica REST API service
- **Database:** MySQL 8 StatefulSet with persistent EBS storage
- **Ingress:** NGINX with SSL termination and domain routing
- **Security:** Let's Encrypt certificates, Kubernetes secrets
- **Storage:** EBS CSI driver with 5Gi persistent volumes
- **Monitoring:** Automated deployment verification

### Infrastructure Security Features
- **Secrets Management:** Kubernetes native secrets for database passwords
- **HTTPS Enforcement:** SSL redirect and force SSL redirect
- **Certificate Automation:** Let's Encrypt with cert-manager
- **Access Control:** RBAC with dedicated service account tokens
- **Network Security:** ClusterIP services for internal communication
- **Persistent Storage:** Data retention across pod restarts

### Deployment Characteristics
- **High Availability:** Multiple replicas for critical components
- **Zero-Downtime:** Rolling deployments for application updates
- **SSL/TLS:** Automatic certificate provisioning and renewal
- **Domain Integration:** Custom domain with proper DNS configuration
- **Health Checks:** Automated verification of deployment success
- **Rollback Capability:** StatefulSet and deployment versioning

---

## Next Steps

1. **Monitoring Implementation:** Set up Prometheus and Grafana for application monitoring
2. **Logging Solution:** Implement centralized logging with ELK stack
3. **Backup Strategy:** Configure automated database backups
4. **Disaster Recovery:** Document and test recovery procedures
5. **Performance Testing:** Load testing and optimization
6. **Security Hardening:** Additional security measures and vulnerability assessments

---

## Troubleshooting Notes

- **Jenkins Restart:** Required after Docker installation to apply group permissions
- **Service Account Token:** Must be extracted from Kubernetes secret and configured in Jenkins
- **Webhook URL:** Must include trailing slash for proper SonarQube integration
- **Namespace Consistency:** Ensure all RBAC configurations use 'prod' namespace
- **Branch Configuration:** Pipeline must reference 'main' branch, not 'master'

---

*This documentation represents the complete setup of a production-ready DevOps CI/CD pipeline with integrated security scanning and quality gates, deployed on AWS infrastructure with Kubernetes orchestration.*
