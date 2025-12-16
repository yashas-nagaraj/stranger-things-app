pipeline {
    agent any
    environment {
        DOCKER_CREDS = credentials('dockerhub-login') 
        SSH_CREDS = credentials('ec2-ssh-key')
        // We will update this IP automatically or manually later
        APP_SERVER_IP = '3.110.209.179' 
    }
    stages {
        stage('Build & Push') {
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-login') {
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
                            # Clean up
                            docker stop frontend backend || true
                            docker rm frontend backend || true
                            docker network rm stranger-net || true
                            
                            # Create Network
                            docker network create stranger-net
                            
                            # Run Backend
                            docker run -d --name backend --net stranger-net -p 5000:5000 \
                                -e DB_HOST="YOUR_RDS_ENDPOINT" \
                                -e DB_USER="admin" \
                                -e DB_PASSWORD="strangerpassword" \
                                -e DB_NAME="stranger_db" \
                                YOUR_DOCKERHUB_USER/stranger-backend:latest
                                
                            # Run Frontend
                            docker run -d --name frontend --net stranger-net -p 80:80 \
                                YOUR_DOCKERHUB_USER/stranger-frontend:latest
                        '
                    """
                }
            }
        }
    }
}
