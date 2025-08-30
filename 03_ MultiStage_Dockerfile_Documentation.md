# Day-4: DevSecOps Mega Project Series – MultiStage Dockerfile

## Overview  
It explains not only *what was done* but also *why* each step was important. The goal of this project is to **dockerize a full-stack Node.js + MySQL application**, implement **multi-stage Docker builds**, orchestrate with **Docker Compose**, and finally automate everything using a **Jenkins CI/CD pipeline** with **DevSecOps practices** integrated.

---

### 1. **Why Dockerization?**
- Running locally required Node.js, MySQL setup, and dependencies installed manually.
- Ports needed to be configured manually (`5000`, `3306`, `3000`).
- Sharing the application with others was difficult due to environment setup issues.
- Solution: **Dockerize the app** to package dependencies, environment setup, and runtime into containers.

---

### 2. **Docker Concepts Introduced**
- **Container**: Isolated environment where the app runs.
- **Docker Image**: Portable package containing code + dependencies + environment setup.
- **Docker Compose**: Used to manage multiple containers (frontend, backend, database) together.

---

### 3. **Multi-Stage Dockerfiles**
#### Why?
- Reduces image size by separating **build stage** and **production stage**.
- Keeps only necessary files (like `dist`/`build`) while discarding large unnecessary files (e.g., `node_modules`).

#### Example Process:
- **Stage 1 (Builder Stage):**
  - Use lightweight Node.js base image (`node:22-alpine`).
  - Set working directory (`/app`).
  - Copy `package.json` & `package-lock.json` → run `npm ci`.
  - Copy source code and run `npm run build` → generates static build (`dist`/`build`).
- **Stage 2 (Production Stage):**
  - Use **Nginx Alpine** image.
  - Copy only static build files from Stage 1.
  - Add Nginx config, expose port `80`.
  - Run Nginx in foreground (`daemon off`).

**Backend Dockerfile:**
- Single-stage Dockerfile was enough (copy code → install dependencies → run app).
- Multi-stage unnecessary since the entire directory was required.

---
---

## 🔑 Key Work Done

### 1. **Backend Dockerfile (`api/Dockerfile`)**
```dockerfile
FROM node:22-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install --only=production

COPY . .
EXPOSE 5000
CMD ["node", "app.js"]
```
#### Explanation:
- **Base Image**: `node:22-alpine` → lightweight Node.js runtime.
- **WORKDIR**: `/app` → sets the working directory for subsequent instructions.
- **Dependency Install**: Only production dependencies are installed (`npm install --only=production`) → reduces image size & attack surface.
- **Source Copy**: Copies all application files.
- **Expose Port 5000**: Backend API listens on port `5000`.
- **CMD**: Launches the backend via `node app.js`.

🔎 **Reasoning**: Backend needs all source code and dependencies → single-stage Dockerfile is enough (multi-stage unnecessary).

---

### 2. **Frontend Dockerfile (`client/Dockerfile`)**
```dockerfile
# Stage 1: Build React App
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx/default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
#### Explanation:
- **Multi-Stage Build**:
  - **Builder Stage**:  
    - Installs dependencies using `npm ci` (faster & reproducible).  
    - Builds React app → generates static files (`/build`).  
  - **Production Stage**:  
    - Uses `nginx:alpine` (tiny web server).  
    - Copies only the static build folder → keeps image small.  
    - Adds custom nginx config for SPA routing.  
    - Exposes port `80`.  
    - Runs nginx in foreground.

⚡ **Benefit**: Final image is minimal (~20–30MB) instead of hundreds of MB with dependencies.

---

### 3. **Docker Compose (`docker-compose.yaml`)**
```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8
    container_name: mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: Aditya
      MYSQL_DATABASE: crud_app
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql-init:/docker-entrypoint-initdb.d
    ports:
      - "3306:3306"

  backend:
    build: ./api
    container_name: backend-api
    environment:
      DB_HOST: mysql
      DB_USER: root
      DB_PASSWORD: Aditya
      DB_NAME: crud_app
      JWT_SECRET: devopsShackSuperSecretKey
      RESET_ADMIN_PASS: 'true'
    depends_on:
      - mysql
    ports:
      - "5000:5000"

  frontend:
    build: ./client
    container_name: frontend-react
    ports:
      - "3000:80"
    depends_on:
      - backend

volumes:
  mysql-data:
```
#### Explanation:
- **MySQL Service**:
  - Uses `mysql:8` image.
  - Sets root password + database name.
  - Mounts named volume (`mysql-data`) → ensures data persistence.
  - Mounts `./mysql-init` → runs schema setup (`init.sql`) automatically.
  - Exposes port `3306`.

- **Backend Service**:
  - Build context: `./api` (backend Dockerfile).
  - Depends on MySQL → ensures DB is up before backend starts.
  - Environment variables set DB connection + JWT secret.
  - Exposes port `5000`.

- **Frontend Service**:
  - Build context: `./client` (frontend Dockerfile).
  - Depends on backend.
  - Maps port `80` inside container to `3000` on host.

🔗 **Networking**: Docker Compose automatically creates a default bridge network → containers communicate using **service names** (`mysql`, `backend`, `frontend`).

---

### 4. **Jenkins Pipeline (`Jenkinsfile`)**
```groovy
pipeline {
    agent any
    
    tools{
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
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NodeJS-project-Dsonar.projectKey=NodeJS-PROJECT '''
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

### 4.1 **Jenkins Setup**
- Created **EC2 instances**: one for Jenkins, one for SonarQube.
- Installed Jenkins on Ubuntu:
  - Installed Java first (prerequisite).
  - Installed Jenkins LTS.
  - Enabled Jenkins service for auto-start on boot.
- Installed **plugins**:
  - Stage View.
  - SonarQube Scanner.
  - NodeJS.
  - Docker Pipeline.

---

### 4.2 **SonarQube Setup**
- Installed Docker & ran SonarQube container on port `9000`.
- Configured admin credentials & token.
- Integrated SonarQube with Jenkins:
  - Added SonarQube server in Jenkins config.
  - Added webhook for Quality Gate check.

---

### 4.3 **Jenkins CI/CD Pipeline**
Pipeline stages included:
1. **Checkout Code** – Clone repo from GitHub branch `docker-build-deploy`.
2. **NodeJS Compilation** – Install dependencies & build app.
3. **Code Scanning**
   - **GitLeaks** – Secret detection.
   - **SonarQube Analysis** – Code quality check.
   - **Trivy** – Security scan for Docker images.
4. **Docker Build & Push**
   - Built backend & frontend Docker images.
   - Tagged with `latest`.
   - Pushed to Docker Hub.
5. **Docker Image Scanning**
   - Used `trivy image` to scan vulnerabilities before pushing.
6. **Docker Compose Deployment**
   - Ran `docker-compose up -d` on Jenkins server.
   - Verified app was running successfully at `http://<jenkins-ip>:3000`.

---

### 4.4 **Pipeline Explanation**
1. **Git Checkout** – Pulls code from GitHub `dev` branch.  
2. **Compilation Check** – Runs Node syntax checks for frontend & backend JS files.  
3. **GitLeaks** – Scans repo for secrets (tokens, API keys).  
4. **SonarQube Analysis** – Runs code quality + bug/vulnerability analysis.  
5. **Quality Gate Check** – Pipeline waits for SonarQube result → can block if fails.  
6. **Trivy FS Scan** – Scans source code for vulnerabilities (SCA).  
7. **Docker Build & Push (Backend)** – Builds backend image → scans with Trivy → pushes to Docker Hub.  
8. **Docker Build & Push (Frontend)** – Same for frontend.  
9. **Deploy via Docker Compose** – Starts all services (`mysql`, `backend`, `frontend`) together.

🔒 **Security Layers in CI/CD**:
- **Secrets Scan**: GitLeaks.  
- **Code Quality & Vulnerabilities**: SonarQube.  
- **Image Vulnerability Scan**: Trivy.  

---

## ✅ Final Outcome
- Backend & Frontend dockerized with optimized **Dockerfiles**.  
- Services orchestrated using **docker-compose**.  
- Full **CI/CD pipeline** automated in Jenkins with integrated **DevSecOps practices**.  
- Security checks embedded at **every stage**.  
- Application successfully deployed and accessible at:  
  - `http://<jenkins-server-ip>:3000` → Frontend  
  - `http://<jenkins-server-ip>:5000` → Backend API  
  - `mysql://<jenkins-server-ip>:3306` → Database  

---

## 📌 Key Takeaways
- **Multi-stage Dockerfiles** save space and improve efficiency for frontend builds.  
- **Single-stage Dockerfiles** are fine when the entire codebase is needed (backend).  
- **Docker Compose** simplifies managing multi-container apps.  
- **CI/CD pipelines** should always integrate **security scanning** (SAST, SCA, secrets, container scans).  
- **DevSecOps** = Security at every stage of development lifecycle.  

