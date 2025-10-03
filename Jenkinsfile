pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'mahmoud8824'
        IMAGE_NAME     = 'my-nginx'
        IMAGE_TAG      = 'v1'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/MahmoudEzzat8824/DEPI-task.git'
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build -t $DOCKERHUB_USER/$IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Login & Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker push $DOCKERHUB_USER/$IMAGE_NAME:$IMAGE_TAG'
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent (credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@<EC2_IP> "
                          docker login -u $DOCKERHUB_USER -p $DOCKER_PASS &&
                          docker pull $DOCKERHUB_USER/$IMAGE_NAME:$IMAGE_TAG &&
                          docker rm -f mynginx || true &&
                          docker run -d --name mynginx -p 80:80 $DOCKERHUB_USER/$IMAGE_NAME:$IMAGE_TAG
                        "
                    '''
                }
            }
        }
    }
}
