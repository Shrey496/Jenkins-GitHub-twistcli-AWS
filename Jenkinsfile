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

    stage('Run Container') {
      steps {
        sh "docker run -d -p 8080:80 --name ${CONTAINER_NAME} ${IMAGE_NAME}:${TAG}"
      }
    }

    stage('Inspect Image and Container') {
      steps {
        script {
          def imageInspectRaw = sh(script: "docker inspect ${IMAGE_NAME}:${TAG} | jq '.[0] | {Id, Created, Config: {ExposedPorts}}'", returnStdout: true).trim()
          echo imageInspectRaw
          def containerInspectRaw = sh(script: "docker inspect ${CONTAINER_NAME} | jq '.[0] | {Id, Created, NetworkSettings: {Ports}}'", returnStdout: true).trim()
          echo containerInspectRaw
          writeFile file: INSPECT_FILE, text: """\
========== Image Inspect ==========
${imageInspect}

========== Container Inspect ==========
${containerInspect}
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

