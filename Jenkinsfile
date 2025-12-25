pipeline {
    agent any
    environment {
        DOCKER_CREDS = credentials('dockerhub-login') 
        SSH_CREDS = credentials('ec2-ssh-key')
        // UPDATE THIS: Put your LAZARUS APP SERVER Public IP here
        APP_SERVER_IP = '3.111.32.31' 
    }
    stages {
        stage('Build & Push') {
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-login') {
                        // We use the same image names as your manual test for consistency
                        sh 'docker build -t yashasnagaraj/stranger-backend:latest ./backend'
                        sh 'docker push yashasnagaraj/stranger-backend:latest'
                        sh 'docker build -t yashasnagaraj/stranger-frontend:latest ./frontend'
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
                            # 1. Clean up old containers (Stop & Remove)
                            docker stop frontend backend || true
                            docker rm frontend backend || true
                            
                            # 2. Clean up network (We use lazarus-net to match your manual setup)
                            docker network rm lazarus-net || true
                            docker network create lazarus-net
                            
                            # 3. Run Backend (With CORRECT Password)
                            docker run -d --name backend --net lazarus-net -p 5000:5000 \
                                -e DB_HOST="lazarus-db.chumewc4kop7.ap-south-1.rds.amazonaws.com" \
                                -e DB_USER="admin" \
                                -e DB_PASSWORD="strangerthings" \
                                -e DB_NAME="stranger_db" \
                                yashasnagaraj/stranger-backend:latest
                                
                            # 4. Run Frontend
                            docker run -d --name frontend --net lazarus-net -p 80:80 \
                                yashasnagaraj/stranger-frontend:latest
                        '
                    """
                }
            }
        }
    }
}
