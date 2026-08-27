# Jenkins-SonarQube-Integration

# Jenkins & SonarQube Integration

End-to-end setup guide for integrating **Jenkins** (CI server) with **SonarQube** (code quality/static analysis) on AWS EC2, using a Maven project as the example (`studentapp`).

---

## Architecture

Two separate EC2 instances:

| Server    | Purpose                          | Instance Type   |
|-----------|-----------------------------------|------------------|
| Jenkins   | CI server, runs build & pipeline  | c7i-flex.large   |
| SonarQube | Code quality analysis (Docker)    | c7i-flex.large   |

Flow: Jenkins builds the project → runs Maven with the SonarQube scanner → results pushed to SonarQube → SonarQube notifies Jenkins via webhook (Quality Gate status).

---

## 1. Launch EC2 Instances

Launch **2 EC2 instances** with AMI type `c7i-flex.large`:
- `jenkins-server`
- `sonarqube-server`

Make sure security groups allow:
- Jenkins server: port `8080` (Jenkins UI), `22` (SSH)
- SonarQube server: port `9000` (SonarQube UI), `22` (SSH)

---

## 2. Setup Jenkins Server

Install Java 17 and Maven (required for building the project and running Jenkins):

```bash
apt update -y
apt install openjdk-17-jdk -y
apt install maven -y
```

> Also install Jenkins itself on this server if not already installed.

---

## 3. Setup SonarQube Server

Install Docker and run SonarQube as a container:

```bash
apt update -y
apt install docker.io -y

docker run -d --name sonarqube-custom -p 9000:9000 sonarqube:10.6-community
```

---

## 4. Configure SonarQube

### 4.1 Access SonarQube
Open in browser:
```
http://<sonarqube-server-ip>:9000
```

### 4.2 Login
- Username: `admin`
- Password: `admin`

You will be prompted to update the password on first login.

### 4.3 Create a Webhook
Go to **Administration → Configuration → Webhooks → Create**

| Field | Value |
|-------|-------|
| Name  | `sonar-webhook` |
| URL   | `http://<jenkins-server-ip>:8080/sonarqube-webhook/` |

### 4.4 Create a Local Project

Go to **Projects → Create Project → Local Project**

| Field       | Value        |
|-------------|--------------|
| Name        | `studentapp` |
| Project Key | `studentapp` |
| Branch      | `main`       |

Click **Next** → select **Use the global setting** → **Create project**.

### 4.5 Generate a Project Token

Go to the project → **Locally** → generate an analysis token.

> ⚠️ **Do not commit or share this token.** Store it only in Jenkins Credentials (see Step 5.2). Treat it as a secret, like a password.

### 4.6 Select Build Tool

Choose **Maven**, then copy the generated command (SonarQube will show you a template like below — replace placeholders with your own values):

```bash
mvn clean verify sonar:sonar \
  -Dsonar.projectKey=studentapp \
  -Dsonar.projectName='studentapp' \
  -Dsonar.host.url=http://<sonarqube-server-ip>:9000 \
  -Dsonar.token=<your-sonarqube-token>
```

---

## 5. Configure Jenkins

### 5.1 Install Required Plugins

Go to **Manage Jenkins → Plugins** and install:
- **SonarQube Scanner** plugin
- **Pipeline Stage View** plugin

### 5.2 Add SonarQube Token as a Jenkins Credential

Go to **Manage Jenkins → Credentials → (global) → Add Credentials**

| Field | Value |
|-------|-------|
| Kind  | Secret text |
| Secret| `<your-sonarqube-token>` |
| ID    | `sonar-token` |

### 5.3 Configure SonarQube Server in Jenkins

Go to **Manage Jenkins → System → SonarQube servers**

- Enable **Environment variables for SonarQube servers**
- Set:

| Field | Value |
|-------|-------|
| Name        | `Sonar-env` |
| Server URL  | `http://<sonarqube-server-ip>:9000` |
| Server authentication token | `sonar-token` (credential created above) |

Click **Save**, then restart Jenkins:

```
http://<jenkins-server-ip>:8080/restart
```

---

## 6. Create Jenkins Pipeline Job

1. Create a **New Item → Pipeline**.
2. Under **Pipeline script**, add a build + SonarQube analysis stage, e.g.:

```groovy
pipeline {
    agent any
    stages {
        stage('pull') {
            steps {
                git branch: 'main', url: 'https://github.com/Rohit-1920/EasyCRUD-Updated.git'
            }
        }
        stage('build') {
            steps {
                sh '''cd backend
                    mvn clean package -DskipTests'''
            }
        }
        stage('test') {
            steps {
                withSonarQubeEnv(credentialsId: 'sonar-token', installationName: 'Sonar-env'){
                    sh '''cd backend
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \\
                        -Dsonar.projectKey=studentapp \\
                        -Dsonar.projectName=studentapp'''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
                }
            }
        }
    }
}
```

3. Click **Save**.

### 6.1 Before Building
Ensure **Java 17** is selected/configured as the JDK on the Jenkins server (under **Manage Jenkins → Tools → JDK installations**).

### 6.2 Build the Project
Once configuration is confirmed, click **Build Now**. Check the **Pipeline Stage View** for stage-by-stage status, and verify the analysis results on the SonarQube dashboard.

---

## Security Notes

- **Never hardcode tokens, passwords, or IPs in version-controlled files.** Use Jenkins Credentials for secrets and environment variables / parameters for IPs and URLs.
- If a real SonarQube token was ever pasted into a script, chat, or repo, **revoke and regenerate it** from SonarQube under **My Account → Security → Tokens**.
- Restrict EC2 security groups to only the IPs/ports that need access (avoid `0.0.0.0/0` on ports 8080/9000 in production).
- Change the default SonarQube `admin` password immediately (already covered in Step 4.2).

---

## Summary

| Component | URL |
|-----------|-----|
| Jenkins   | `http://<jenkins-server-ip>:8080` |
| SonarQube | `http://<sonarqube-server-ip>:9000` |

With this setup, every Jenkins build triggers a Maven build, runs SonarQube static analysis, and blocks/passes the pipeline based on the SonarQube Quality Gate result.