# 🚀 CI Pipeline Documentation

This guide provides a detailed step-by-step process to **build a complete CI pipeline from scratch** using Jenkins, SonarQube, GitLeaks, Trivy, and Docker.  
It includes infrastructure setup, tool installation, Jenkins configuration, pipeline stages, and real-world workflows.

---

## 1. Infrastructure Setup

To create a CI pipeline, we first prepare the required infrastructure.

### ✅ Requirements
- **GitHub Repository** – Store your application code (`api/` for backend, `client/` for frontend).
- **Jenkins Server** – Automates the pipeline stages.
- **SonarQube Server** – Performs static code analysis.
- **Docker** – Builds and runs application containers.
- **GitLeaks** – Detects secrets in source code.
- **Trivy** – Scans dependencies and container images for vulnerabilities.

### ⚙️ Jenkins Installation (Ubuntu 24.04)
```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee      /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]      https://pkg.jenkins.io/debian binary/ | sudo tee      /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
```

- Access Jenkins: `http://<public-ip>:8080`
- Unlock with password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### ⚙️ SonarQube Installation (Docker)
```bash
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker ubuntu
newgrp docker
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
Access SonarQube at: `http://<public-ip>:9000`  
Default login: `admin / admin`

### ⚙️ Security Tools Setup
```bash
# GitLeaks
curl -s https://raw.githubusercontent.com/gitleaks/gitleaks/master/install.sh | bash

# Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key |     gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo deb [signed-by=/usr/share/keyrings/trivy.gpg]     https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main |     sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
```

---

## 2. Jenkins Configuration

Inside Jenkins, we configure plugins, tools, and credentials.

### 🔌 Plugins to Install
- NodeJS
- SonarQube Scanner
- Docker Pipeline
- Pipeline Stage View

### ⚙️ Tool Configuration
- **NodeJS** → `Manage Jenkins > Global Tool Configuration > NodeJS` → Add installation `nodejs16`  
- **SonarQube Server** → `Manage Jenkins > Configure System > SonarQube servers`  
  - Name: `sonar`  
  - URL: `http://<sonarqube-ip>:9000`  
  - Token: Generate from SonarQube dashboard  
- **SonarQube Scanner** → `Manage Jenkins > Global Tool Configuration > SonarQube Scanner` → Name it `sonar-scanner`  
- **DockerHub Credentials** → `Manage Jenkins > Credentials` → Add with ID: `docker-cred`  

### 🔔 SonarQube Webhook
- In SonarQube → `Administration > Configuration > Webhooks`
- Add URL:
```
http://<jenkins-ip>:8080/sonarqube-webhook/
```

This allows Jenkins to receive quality gate results.

---

## 3. CI Pipeline Stages

The CI pipeline runs in multiple stages to ensure **code quality, security, and automation**.

1. **Git Checkout** – Clone source code from `dev` branch into Jenkins workspace.  
2. **Frontend Compilation** – Syntax validation for JavaScript in `/client`.  
3. **Backend Compilation** – Syntax validation for JavaScript in `/api`.  
4. **GitLeaks Scan** – Detects API keys, tokens, or secrets committed by mistake.  
5. **SonarQube Analysis** – Scans code for bugs, vulnerabilities, and code smells.  
6. **Quality Gate Check** – Ensures project meets predefined quality standards (e.g., ≥80% coverage, no critical bugs).  
7. **Trivy FS Scan** – Scans dependency files (e.g., `package.json`) for vulnerabilities.  
8. **Build Backend Docker Image** – Build, scan, and push backend image to DockerHub.  
9. **Build Frontend Docker Image** – Build, scan, and push frontend image to DockerHub.  
10. **Deploy with Docker Compose** – Run application containers.

---

## 4. Jenkinsfile (Pipeline Script)

```groovy
pipeline {
    agent any
    
    tools {
        nodejs 'nodejs16'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'dev', url: 'https://github.com/Anshuman-git-code/3-Tier-DevSecOps-Project.git'
            }
        }
        stage('Frontend compilation') {
            steps {
                dir('client') {
                    sh 'find . -name ".js" -exec node --check {} +'
                }
            }
        }
        stage('Backend compilation') {
            steps {
                dir('api') {
                    sh 'find . -name ".js" -exec node --check {} +'
                }
            }
        }
        stage('GitLeaks Scan') {
            steps {
                sh 'gitleaks detect --source ./client --exit-code 1'
                sh 'gitleaks detect --source ./api --exit-code 1'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NodeJS-project                             -Dsonar.projectKey=NodeJS-PROJECT '''
                }
            }
        }
        stage('Quality Gate Check') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }
        stage('Build-Tag & Push Backend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('api') {
                            sh 'docker build -t anshuman0506/backend:latest .'
                            sh 'trivy image --format table -o backend-image-report.html anshuman0506/backend:latest '
                            sh 'docker push anshuman0506/backend:latest'
                        }
                    }
                }
            }
        }
        stage('Build-Tag & Push Frontend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('client') {
                            sh 'docker build -t anshuman0506/frontend:latest .'
                            sh 'trivy image --format table -o frontend-image-report.html anshuman0506/frontend:latest '
                            sh 'docker push anshuman0506/frontend:latest'
                        }
                    }
                }
            }
        }
        stage('Docker Deploy via Compose') {
            steps {
                script {
                    sh 'docker-compose up -d'
                }
            }
        }
    }
}
```

---

## 5. Workflow

1. Developer pushes code → GitHub Webhook triggers Jenkins.  
2. Jenkins runs all pipeline stages automatically.  
3. Reports generated:  
   - GitLeaks secrets report  
   - SonarQube bug/vulnerability report  
   - Trivy FS dependency report  
   - Trivy Docker image report  
4. Docker images are pushed to DockerHub.  
5. Application is deployed with Docker Compose.

---

## 6. Example Situation (Real-World Flow)

**Situation:**  
When a client requests a simple change, for example, changing the background color, the workflow proceeds as follows:

1. The client raises a **Jira ticket**.  
2. A developer is assigned to the task.  
3. Branching strategy:  
   - `main` → stable production code  
   - `dev` → integration branch  
   - `feature/backchange` → new feature branch created from `dev`  
4. The developer clones the `feature/backchange` branch locally, makes the change, tests it, then commits & pushes the changes.  
5. Once changes are complete, the developer merges the feature branch back into `dev`.  
6. As soon as the merge happens, the **CI pipeline is automatically triggered**.  
7. The **DevOps engineer** is responsible for this automation by configuring a **webhook** in GitHub.  
   - Webhook ensures: any changes pushed to the `dev` branch trigger Jenkins pipeline execution.  

This ensures automation, consistency, and minimal manual intervention in the CI process.

---

✅ With this setup, you will have a **DevSecOps-ready CI pipeline** that covers:  
- **Code Quality** (SonarQube)  
- **Secret Detection** (GitLeaks)  
- **Dependency Scan** (Trivy FS)  
- **Image Scan** (Trivy Image)  
- **Automated Build & Deploy** (Docker & Compose)  
