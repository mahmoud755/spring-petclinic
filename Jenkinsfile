pipeline {
  agent { label 'docker' }

  options {
    timestamps()
    skipDefaultCheckout(true)
    disableConcurrentBuilds()
  }

  environment {
    IMAGE_REPO  = "mahmoud377/spring-petclinic"
    SMOKE_NAME  = "petclinic-smoke"
    DOCKER_NET  = "jenkins"
    PROD_NAME   = "petclinic-prod"
    PROD_PORT   = "8085"
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
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
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
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
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

    stage('Deploy to Ubuntu (main only)') {
      when { branch 'main' }
      steps {
        sh '''
          set -euxo pipefail

          IMAGE="${IMAGE_REPO}:latest"

          docker pull "$IMAGE"

          docker rm -f ${PROD_NAME} || true

          docker run -d \
            --name ${PROD_NAME} \
            --restart unless-stopped \
            -p ${PROD_PORT}:8080 \
            "$IMAGE"

          for i in $(seq 1 30); do
            if curl -fsS "http://localhost:${PROD_PORT}/" >/dev/null; then
              echo "✅ Deploy OK on port ${PROD_PORT}"
              exit 0
            fi
            echo "waiting app... ($i)"
            sleep 2
          done

          echo "❌ Deploy failed (app not responding on ${PROD_PORT})"
          docker logs ${PROD_NAME} || true
          exit 1
        '''
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
