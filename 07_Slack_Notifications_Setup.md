
# Jenkins Slack Notifications - Detailed Documentation

This documentation explains **how to configure Slack notifications in Jenkins** so that teams are notified whenever a pipeline succeeds or fails. The setup integrates Slack with Jenkins using **webhooks, tokens, and plugins**.

---

## 📌 Why Slack Notifications?
- Provides **real-time alerts** about pipeline execution (success/failure).
- Displays **job details, environment, and quick access to logs**.
- More convenient compared to email notifications (Gmail/Outlook).
- Widely adopted in DevOps teams for collaboration.

---

## 🛠️ Prerequisites
1. A **Slack account** with workspace created.
2. An **AWS EC2 instance** (Ubuntu 24.04) for Jenkins installation.
3. **Ports & Security**: Jenkins requires port `8080` open in the security group.
4. **Java 21** installed (Jenkins prerequisite).

---

## 🚀 Step 1: Create Slack Workspace & App
1. Sign in to Slack → Create a **workspace** (e.g., `DevOpsShack`). → click url(https://slack.com) → click **Lunch workspace**
2. Inside the workspace, create an **App** (at https://api.slack.com/apps)→ Click **From scrach** → Name it (e.g., `Jenkins Integration`). → **Create App**
3. Click & Enable **Incoming Webhooks** (Left Menu) for the app.
   - Add a new webhook to your Slack channel (e.g., `#all-devops-shack`). → **Allow**
   - Copy the **Webhook URL** for later use.
4. Configure **OAuth Scopes/Permissions** (Left Menu):
Under **Scopes** → **Bot Token Scopes** → **Add an OAuth Scopes**
   - `chat:write`
   - `chat:write.public`
   - `channels:read`
   - `groups:read`
   - `users:read`
5. Reinstall the app so changes take effect.
6. Copy the **Bot User OAuth Token**.

---

## 🚀 Step 2: Set up Jenkins on AWS EC2
1. Launch **Ubuntu 24.04 EC2 Instance** (t2.medium, 25GB storage).
2. Install Java:
   ```bash
   sudo apt update
   sudo apt install openjdk-21-jdk -y
   ```
3. Install Jenkins (LTS version):
   ```bash
   curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee        /usr/share/keyrings/jenkins-keyring.asc > /dev/null
   echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]        https://pkg.jenkins.io/debian-stable binary/ | sudo tee        /etc/apt/sources.list.d/jenkins.list > /dev/null
   sudo apt-get update
   sudo apt-get install jenkins -y
   ```
4. Access Jenkins in the browser:  
   `http://<EC2-Public-IP>:8080`
5. Unlock Jenkins using the admin password from:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
6. Install recommended plugins.

---

## 🚀 Step 3: Configure Jenkins with Slack
1. Install **Slack Notification Plugin** in Jenkins.
2. Go to **Manage Jenkins → System → Scroll down to end till find **Slack** **.
3. Enter details:
   - **Workspace** → `<Your Workspace Name>`
   - **Credentials** → Add **Slack Bot User OAuth Token** as `Secret Text`. Click on **Add+**
   - **Default Channel** → `<Youe Channel Name ex: all-anshu-mbt>`
   - check **Custom slack app bot user** → **Apply**
4. Test the connection → should return **SUCCESS**.
<img width="1920" height="1080" alt="Screenshot 2025-09-01 at 5 30 11 PM" src="https://github.com/user-attachments/assets/499f2bc0-a705-429c-a276-43865247fe37" />
<img width="1918" height="990" alt="Screenshot 2025-09-01 at 5 31 35 PM" src="https://github.com/user-attachments/assets/d7b21223-ac5f-4023-8957-fc16801d072c" />

---

## 🚀 Step 4: Add Slack Webhook in Jenkins Credentials
1. Navigate to **Manage Jenkins → Credentials**.
2. Add new credential:
   - Kind → `Secret Text`
   - Secret → Slack **Webhook URL**
   - ID → `slack-webhook`
3. This will be used in the pipeline for posting messages.
<img width="1920" height="1080" alt="Screenshot 2025-09-01 at 5 39 08 PM" src="https://github.com/user-attachments/assets/a7505506-a53e-4854-a52d-64ca6ba18b6b" />

---

## 🚀 Step 5: Jenkins Pipeline with Slack Notifications
Here’s a sample **pipeline** with Slack notification integration:

```groovy
pipeline {
    agent any

    environment {
        PROJECT_NAME = '🧰 My-App'
        ENVIRONMENT  = '🚀 Production'
    }

    stages {
        stage('Compile') {
            steps {
                echo "🏗️ Compiling..."
            }
        }

        stage('Test') {
            steps {
                echo "🧪 Running tests..."
            }
        }

        stage('Build') {
            steps {
                echo "📦 Building..."
            }
        }

        stage('Security') {
            steps {
                echo "🔒 Running security checks..."
            }
        }

        stage('Deploy') {
            steps {
                echo "🚀 Deploying..."
            }
        }
    }

    post {
        success {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                script {
                    def message = """{
                        "text": "*✅ ${PROJECT_NAME} Build Successful!*",
                        "attachments": [
                            {
                                "color": "#36a64f",
                                "fields": [
                                    { "title": "Job", "value": "${env.JOB_NAME}", "short": true },
                                    { "title": "Build", "value": "#${env.BUILD_NUMBER}", "short": true },
                                    { "title": "Environment", "value": "${ENVIRONMENT}", "short": true }
                                ],
                                "footer": "Jenkins CI",
                                "footer_icon": "https://www.jenkins.io/images/logos/jenkins/jenkins.png",
                                "ts": ${System.currentTimeMillis() / 1000},
                                "actions": [
                                    {
                                        "type": "button",
                                        "text": "View Build",
                                        "url": "${env.BUILD_URL}",
                                        "style": "primary"
                                    }
                                ]
                            }
                        ]
                    }"""
                    sh """curl -X POST -H 'Content-type: application/json' --data '${message}' $SLACK_URL"""
                }
            }
        }

        failure {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                script {
                    def message = """{
                        "text": "<!here> *❌ ${PROJECT_NAME} Build Failed!*",
                        "attachments": [
                            {
                                "color": "#FF0000",
                                "fields": [
                                    { "title": "Job", "value": "${env.JOB_NAME}", "short": true },
                                    { "title": "Build", "value": "#${env.BUILD_NUMBER}", "short": true },
                                    { "title": "Environment", "value": "${ENVIRONMENT}", "short": true }
                                ],
                                "footer": "Jenkins CI",
                                "footer_icon": "https://www.jenkins.io/images/logos/jenkins/jenkins.png",
                                "ts": ${System.currentTimeMillis() / 1000},
                                "actions": [
                                    {
                                        "type": "button",
                                        "text": "View Build Logs",
                                        "url": "${env.BUILD_URL}",
                                        "style": "danger"
                                    }
                                ]
                            }
                        ]
                    }"""
                    sh """curl -X POST -H 'Content-type: application/json' --data '${message}' $SLACK_URL"""
                }
            }
        }

        always {
            echo "🎯 Post-build notification sent"
        }
    }
}
```
<img width="1920" height="1080" alt="Screenshot 2025-09-01 at 5 48 26 PM" src="https://github.com/user-attachments/assets/1cb606c1-72cc-4053-80f3-8b6646d70236" />
<img width="1920" height="1080" alt="Screenshot 2025-09-01 at 5 48 43 PM" src="https://github.com/user-attachments/assets/8e058df4-4f82-4c8c-a952-805c388d861e" />
<img width="1920" height="1080" alt="Screenshot 2025-09-01 at 5 48 54 PM" src="https://github.com/user-attachments/assets/50dbf45f-f8e4-45ee-a4d2-4853e8956ca1" />

---

## 📊 Example Slack Notifications
### ✅ Success Notification
- Green color with Job name, environment, and **View Build** button.

### ❌ Failure Notification
- Red color with Job name, environment, and quick access to logs.

---

## 🔒 Security Notes
- Always store **Slack tokens & webhooks** in Jenkins **credentials** (never hardcode).
- Use **scoped tokens** for limited permissions.
- Use separate channels for **production/staging notifications**.

---

## 🎯 Benefits of Slack Integration
- Real-time visibility of CI/CD status.
- Improved collaboration between DevOps teams.
- Faster failure response → reduced MTTR (Mean Time to Recovery).

---

## ✅ Conclusion
By following these steps, Jenkins can send detailed pipeline status notifications directly to Slack. This ensures better collaboration, faster issue detection, and improved CI/CD observability.

---
