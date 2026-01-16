# 🧟 Project Lazarus: Stranger Things App

A 3-tier web application (Frontend + Backend + Database) inspired by *Stranger Things*. This project demonstrates deploying a microservices architecture on AWS using both **Manual Docker Deployment** and an **Automated Jenkins CI/CD Pipeline**.

---

## 🏗️ Architecture



* **Frontend:** HTML/JS Web Interface (Nginx)
* **Backend:** Python Flask API
* **Database:** AWS RDS (MySQL)
* **Infrastructure:** AWS EC2 (Ubuntu 22.04)

---

## 🚀 Part 1: Manual Deployment Guide

Follow these steps to manually build and launch the application on AWS.

### 1. AWS Infrastructure Setup
* **Database (RDS):**
    * **Engine:** MySQL (Free Tier)
    * **DB Identifier:** `lazarus-db`
    * **Credentials:** `admin` / `strangerpassword` (Change strictly for production)
    * **Security Group:** Allow port `3306` from your App Server IP.
    * **Initial DB Name:** `stranger_db`
* **App Server (EC2):**
    * **OS:** Ubuntu 22.04 LTS (t2.micro)
    * **Security Group:** Allow ports `22` (SSH), `80` (HTTP), and `5000` (Custom TCP).

### 2. Server Preparation
SSH into your EC2 instance and install dependencies:
```bash
sudo apt update
sudo apt install docker.io mysql-client git -y

# Grant Docker permissions to user
sudo usermod -aG docker ubuntu
sudo chmod 666 /var/run/docker.sock
3. Database Configuration
Connect to your RDS instance from the App Server and initialize the schema:

Bash

mysql -h <YOUR_RDS_ENDPOINT> -u admin -p
# Password: strangerpassword
Run the following SQL commands:

SQL

USE stranger_db;

CREATE TABLE questions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    question VARCHAR(255)
);

CREATE TABLE answers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    question_id INT,
    answer VARCHAR(255)
);

EXIT;
4. Code Configuration
Clone the Repository:

Bash

git clone [https://github.com/yashas-nagaraj/stranger-things-app.git](https://github.com/yashas-nagaraj/stranger-things-app.git)
cd stranger-things-app
Update Frontend API: Edit frontend/index.html. Change the API constant to your server's public IP:

JavaScript

const API = "http://<APP_SERVER_PUBLIC_IP>:5000/api";
5. Docker Deployment
Run the following commands to launch the containers:

Bash

# 1. Create Network
docker network create lazarus-net

# 2. Build & Run Backend
cd backend
docker build -t lazarus-backend .

# Note: Replace <YOUR_RDS_ENDPOINT> with your actual AWS RDS Endpoint
docker run -d --name backend --net lazarus-net -p 5000:5000 \
  -e DB_HOST="<YOUR_RDS_ENDPOINT>" \
  -e DB_USER="admin" \
  -e DB_PASSWORD="strangerpassword" \
  -e DB_NAME="stranger_db" \
  lazarus-backend

# 3. Build & Run Frontend
cd ../frontend
docker build -t lazarus-frontend .

docker run -d --name frontend --net lazarus-net -p 80:80 lazarus-frontend
Verification: Open http://<APP_SERVER_PUBLIC_IP> in your browser.

♾️ Part 2: CI/CD Pipeline (Jenkins)
This section details how to automate deployments using Jenkins.

1. Jenkins Server Setup
Install Jenkins: Java 17, Jenkins repo, and Docker.

Permissions: Ensure Jenkins user can run Docker:

Bash

sudo usermod -aG docker jenkins
sudo chmod 666 /var/run/docker.sock
Plugins: Install "Docker Pipeline" and "SSH Agent" plugins.

2. SSH Handshake
Enable Jenkins to SSH into the App Server without a password.

On Jenkins: Generate keys (ssh-keygen) and copy the public key (cat ~/.ssh/id_rsa.pub).

On App Server: Paste the key into ~/.ssh/authorized_keys.

3. Jenkins Credentials
Configure the following in Manage Jenkins → Credentials:

dockerhub-login: Username/Password for Docker Hub.

ec2-ssh-key: SSH Username (ubuntu) and Private Key (id_rsa).

4. The Pipeline (Jenkinsfile)
Create a new Pipeline job and point it to your GitHub repository. Ensure your Jenkinsfile contains the following logic:

Groovy

pipeline {
    agent any
    environment {
        DOCKER_CREDS = credentials('dockerhub-login') 
        SSH_CREDS = credentials('ec2-ssh-key')
        APP_SERVER_IP = '<YOUR_APP_SERVER_IP>' // Update this with your EC2 IP
    }
    stages {
        stage('Build & Push') {
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-login') {
                        sh 'docker build --no-cache -t yashasnagaraj/stranger-backend:latest ./backend'
                        sh 'docker push yashasnagaraj/stranger-backend:latest'
                        sh 'docker build --no-cache -t yashasnagaraj/stranger-frontend:latest ./frontend'
                        sh 'docker push yashasnagaraj/stranger-frontend:latest'
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${APP_SERVER_IP} '
                            docker stop frontend backend || true
                            docker rm frontend backend || true
                            docker network rm lazarus-net || true
                            docker network create lazarus-net
                            
                            docker pull yashasnagaraj/stranger-backend:latest
                            docker pull yashasnagaraj/stranger-frontend:latest
                            
                            # Note: Ensure DB_HOST and DB_PASSWORD are correct
                            docker run -d --name backend --net lazarus-net -p 5000:5000 \
                                -e DB_HOST="<YOUR_RDS_ENDPOINT>" \
                                -e DB_USER="admin" \
                                -e DB_PASSWORD="strangerpassword" \
                                -e DB_NAME="stranger_db" \
                                yashasnagaraj/stranger-backend:latest
                                
                            docker run -d --name frontend --net lazarus-net -p 80:80 \
                                yashasnagaraj/stranger-frontend:latest
                        '
                    """
                }
            }
        }
    }
}
⚠️ Troubleshooting
Permission Denied (Docker): If Jenkins cannot connect to the Docker socket, run sudo chmod 666 /var/run/docker.sock and restart Jenkins.

Updates Not Showing: Ensure docker build uses --no-cache and docker pull is executed on the App Server before running containers.

Database Connection Failed: Double-check DB_HOST and DB_PASSWORD environment variables in the Docker run command.

SSH Errors: If the pipeline fails with NoSuchMethodError: sshagent, install the SSH Agent Plugin in Jenkins.
