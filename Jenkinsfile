pipeline {
  agent { label 'docker' }

  options {
    timestamps()
    skipDefaultCheckout(true)
    disableConcurrentBuilds()
  }

  environment {
    IMAGE_REPO = "mahmoud377/spring-petclinic"
    SMOKE_NAME = "petclinic-smoke"
    DOCKER_NET = "jenkins"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Prepare Tags') {
      steps {
        script {
          env.COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.IMAGE_TAG    = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
        }
      }
    }

    stage('Build & Test (Skip PostgresIntegrationTests)') {
      steps {
        sh '''
          set -euxo pipefail
          chmod +x mvnw
          ./mvnw -B clean package -Dtest=!PostgresIntegrationTests
        '''
      }
      post {
        always {
          junit '**/target/surefire-reports/*.xml'
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -euxo pipefail
          docker build -t ${IMAGE_REPO}:${IMAGE_TAG} .
          docker tag ${IMAGE_REPO}:${IMAGE_TAG} ${IMAGE_REPO}:${COMMIT_SHORT}
        '''
      }
    }

    stage('Smoke Test Container') {
      steps {
        sh '''
          set -euxo pipefail

          docker rm -f ${SMOKE_NAME} || true

          docker run -d \
            --name ${SMOKE_NAME} \
            --network ${DOCKER_NET} \
            ${IMAGE_REPO}:${IMAGE_TAG}

          # wait a bit for app startup (later we can replace with healthcheck loop)
          sleep 15

          curl -f http://${SMOKE_NAME}:8080/ || (docker logs ${SMOKE_NAME} && exit 1)

          docker rm -f ${SMOKE_NAME}
        '''
      }
    }

    stage('Push Docker Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'mahmoud377',
          passwordVariable: 'dckr_pat_XIJS-s_jAVPP25l9Iq1eFBBAAUM'
        )]) {
          sh '''
            set -euxo pipefail
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            docker push ${IMAGE_REPO}:${IMAGE_TAG}
            docker push ${IMAGE_REPO}:${COMMIT_SHORT}

            docker logout
          '''
        }
      }
    }

    stage('Tag & Push latest (main only)') {
      when { branch 'main' }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'mahmoud377',
          passwordVariable: 'dckr_pat_XIJS-s_jAVPP25l9Iq1eFBBAAUM'
        )]) {
          sh '''
            set -euxo pipefail
            docker tag ${IMAGE_REPO}:${IMAGE_TAG} ${IMAGE_REPO}:latest

            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${IMAGE_REPO}:latest
            docker logout
          '''
        }
      }
    }

    stage('Archive Artifact') {
      steps {
        archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
      }
    }
  }

  post {
    always {
      sh '''
        docker rm -f ${SMOKE_NAME} || true
      '''
    }
  }
}
