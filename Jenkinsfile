pipeline {
  agent { label 'ec2-docker' }

  environment {
    IMAGE_NAME = 'redis'
    TAG = 'custom'
    CONTAINER_NAME = 'redis-container'
    INSPECT_FILE = 'dockerinspect.txt'
    SCAN_FILE = 'twistcli-scan-report.txt'
    TWISTCLI_URL = 'https://us-east1.cloud.twistlock.com/us-2-158290582/api/v1/util/twistcli'
    PRISMA_CONSOLE = 'https://us-east1.cloud.twistlock.com/us-2-158290582'
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
                ${IMAGE_NAME}:${TAG} > ${SCAN_FILE} || true
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
      node ('ec2-docker') {
        script {
          echo 'Cleaning up Docker container and image...'
          sh "docker rm -f ${env.CONTAINER_NAME} || true"
          sh "docker rmi ${env.IMAGE_NAME}:${env.TAG} || true"
          archiveArtifacts artifacts: "${env.SCAN_FILE}, ${env.INSPECT_FILE}", fingerprint: true
        }
      }
    }
  }
}
