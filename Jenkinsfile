pipeline {
    agent any
    
    environment {
        // Docker Hub
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-cred')
        DOCKERHUB_USERNAME = 'mahmoud8824'
        IMAGE_NAME = 'my-nginx'
        IMAGE_TAG = "${BUILD_NUMBER}"
        
        // EC2
        EC2_HOST = '13.49.75.178'
        EC2_USER = 'ubuntu'
        CONTAINER_NAME = 'nginx-app'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Transfer Files to EC2') {
            steps {
                echo "Transferring files to EC2..."
                sshagent (credentials: ['ec2-ssh-key']) {
                    sh """
                        scp -o StrictHostKeyChecking=no Dockerfile index.html ${EC2_USER}@${EC2_HOST}:/home/${EC2_USER}/
                    """
                }
            }
        }
        
        stage('Build Docker Image on EC2') {
            steps {
                echo "Building Docker image on EC2..."
                sshagent (credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            cd /home/${EC2_USER} &&
                            docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} . &&
                            docker tag ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest
                        '
                    """
                }
            }
        }
        
        stage('Login to Docker Hub & Push') {
            steps {
                echo 'Logging into Docker Hub and pushing image...'
                withCredentials([usernamePassword(credentialsId: 'docker-hub-cred', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                    sshagent (credentials: ['ec2-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                                echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin &&
                                docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} &&
                                docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest
                            '
                        """
                    }
                }
            }
        }
        
        stage('Deploy Container') {
            steps {
                echo "Deploying container on EC2..."
                sshagent (credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            docker stop ${CONTAINER_NAME} || true &&
                            docker rm ${CONTAINER_NAME} || true &&
                            docker run -d --name ${CONTAINER_NAME} -p 80:80 ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker ps | grep ${CONTAINER_NAME}
                        '
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up old Docker images on EC2...'
            sshagent (credentials: ['ec2-ssh-key']) {
                sh """
                    ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                        docker logout || true &&
                        docker image prune -f || true
                    '
                """
            }
        }
        success {
            echo '✅ Pipeline completed successfully!'
            echo "🌍 Application is accessible at: http://${EC2_HOST}"
        }
        failure {
            echo '❌ Pipeline failed. Please check the logs.'
        }
    }
}
