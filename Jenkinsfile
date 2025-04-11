pipeline {
  agent { label 'ec2-docker' }

  environment {
    IMAGE_NAME = 'nginx'
    TAG = 'custom'
    CONTAINER_NAME = 'nginx-webserver'
    INSPECT_FILE = 'dockerinspect.txt'
  }

  stages {
    stage('Clone Repository') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build --no-cache -t ${IMAGE_NAME}:${TAG} ."
      }
    }
    stage('Download twistcli') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: 'Prisma-access-secret', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            sh """
              curl -u "$USERNAME:$PASSWORD" -o twistcli --silent --insecure ${TWISTCLI_URL}
              chmod +x twistcli
            """
          }
        }
      }
    }

    stage('Scan Docker Image using twistcli') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: 'Prisma-access-secret', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            sh """
              ./twistcli --debug images scan --address https://${PRISMA_CONSOLE} \
                --user $USERNAME --password $PASSWORD --details \
                ${IMAGE_NAME}:${TAG}
            """
          }
        }
      }
    }

    stage('Run Container') {
      steps {
        script{
          sh "docker run -d --name ${CONTAINER_NAME} ${IMAGE_NAME}:${TAG}"
        }
      }
    }

    stage('Inspect Image and Container') {
      steps {
        script {
          sh """
            echo "Image Inspect:" > ${INSPECT_FILE}
            docker inspect ${IMAGE_NAME}:${TAG} | jq '.[0] | {Id, Created}' >> ${INSPECT_FILE}
            
            echo "\\nContainer Info:" >> ${INSPECT_FILE}
            docker inspect ${CONTAINER_NAME} | jq '.[0] | {Id, Created}' >> ${INSPECT_FILE}
          """
        }
      }
    }
  }

  post {
    always {
      echo 'Cleaning up...'
      sh """
        docker rm -f ${CONTAINER_NAME} || true
        docker rmi ${IMAGE_NAME}:${TAG} || true
      """
      archiveArtifacts artifacts: INSPECT_FILE, onlyIfSuccessful: false
      echo 'Done.'
    }
  }
}

