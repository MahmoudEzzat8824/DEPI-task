pipeline {
  agent any

  environment {
    IMAGE_NAME = "mahmoud8824/my-nginx"   
    IMAGE_TAG  = "v1"
    IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"


    DOCKERHUB_CREDS = 'dockerhub-creds'   
    EC2_SSH_CRED    = 'ec2-ssh-key'       
    EC2_USER        = 'ubuntu'          
    EC2_IP          = '13.49.75.178'  
    CONTAINER_NAME  = 'my-nginx-app'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build image') {
      steps {
        sh """
           docker --version || true
           echo "Building ${IMAGE}"
           docker build -t ${IMAGE} .
        """
      }
    }

    stage('Login & Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDS}", usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
          sh '''
            echo "Logging in to Docker Hub as $DOCKERHUB_USER"
            echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin
            docker push ${IMAGE}
            docker logout
          '''
        }
      }
    }

    stage('Deploy to EC2') {
      steps {
        sshagent (credentials: ["${EC2_SSH_CRED}"]) {
          withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDS}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
            sh """
              set -e
              echo "Deploying to ${EC2_USER}@${EC2_IP}"
              ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} /bin/bash -s <<'REMOTE_EOF'
                #!/bin/bash
                set -e

                echo "Logging in to Docker Hub on remote host"
                echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin

                echo "Pulling image: ${IMAGE}"
                docker pull ${IMAGE}

                # Stop & remove old container if exists
                if docker ps -a --format '{{.Names}}' | grep -Eq "^${CONTAINER_NAME}\$"; then
                  echo "Stopping existing container ${CONTAINER_NAME}"
                  docker rm -f ${CONTAINER_NAME} || true
                fi

                # Run new container (detached, restart policy)
                echo "Running new container ${CONTAINER_NAME}"
                docker run -d --name ${CONTAINER_NAME} -p 80:80 --restart unless-stopped ${IMAGE}

                echo "Deployment finished on remote host."
                docker logout
              REMOTE_EOF
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline finished successfully. Image: ${IMAGE}"
    }
    failure {
      echo "Pipeline failed. Check logs."
    }
  }
}
